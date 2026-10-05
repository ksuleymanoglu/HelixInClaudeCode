# Helix Workflow — Setup Guide

Helix is a plan-then-build-under-review workflow for coding agents. You describe a
feature; a planner breaks it into ordered, independently reviewable **checkpoints**;
then, for each checkpoint, a **test-author** generates tests from its acceptance criteria,
the agent implements it, and **two independent, context-isolated reviewers** must _both_
approve before the next checkpoint begins. Two enforcement hooks make the process binding
rather than advisory:

- a **guard** that stops the main agent from marking its own work as approved or rewriting
  a reviewer's findings — and confines each subagent to its own role, and
- a **stop-gate** that refuses to let the agent end its turn while any checkpoint is
  still unapproved.

Both hooks also enforce a **budget** so a run cannot loop forever: a limit on review rounds
per checkpoint and per run, a limit on active time per checkpoint and per run, and a
**diff alarm** when the code changed grows past a set size. When any part of the budget is
spent, the run **halts** and goes back to you; the agent cannot raise its own limits.

A checkpoint's `passed` flag is never written by a tool call at all: the stop-gate
**derives** it from the two reviewers' verdicts and flips it to `true` only when _both_
have approved **and those approvals still describe the code**: each approval records the
files it covers and a hash of them, and the stop-gate recomputes that hash before it
counts the approval. This mirrors Shopify's design, where "two independent, context-isolated
reviewer agents check the new code" and the loop repeats "until both reviewers approve,"
and where "a subagent reads the reference code and generates test cases for each
checkpoint" that define what "proven" means.

This guide sets Helix up in Claude Code (the reference implementation).
Claude Code has the two capabilities Helix depends on: a pre-tool hook that can
block a call, and a stop hook that can refuse to let the agent finish.

> Credit: **Helix** is the name of an agent-driven migration system published by Shopify
> Engineering, used to migrate their mobile app from React Native to native Swift and
> Kotlin. Its published design — breaking work into checkpoints of increasing complexity,
> using independent context-isolated reviewer agents, and enforcing gates the agent
> "can't override" — is the pattern this guide implements for general coding tasks in Claude Code.
> This guide is an independent implementation, not affiliated with or
> endorsed by Shopify. See <https://shopify.engineering/helix>.

---

## The four primitives Helix needs

| Primitive                | Role in Helix                                                                                            | Why it must exist                                                                                                                                 |
| ------------------------ | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| A reusable command       | The `orchestrator` that drives plan → approve → test → implement → review loop                           | Entry point the user invokes                                                                                                                      |
| Subagents                | The `planner`, the `test-author`, and two reviewers `reviewer-a` / `reviewer-b`, each in its own context | Separation of duties: the builder cannot be the approver, and two independent reviewers must agree                                                |
| A blocking pre-tool hook | The `guard`                                                                                              | Confines each role: the builder can't approve or edit findings; each reviewer writes only its own verdict/file, and only for the code on disk; the test-author writes only tests |
| A blocking stop hook     | The `stop-gate`                                                                                          | Forces the loop to continue until every checkpoint is approved by both reviewers and every approval still matches the code, and halts it when the budget is spent |

The state of a run lives, per project, in the state file plus one findings file per reviewer:

```
.claude/helix/state.json     # the plan: checkpoints, acceptance criteria, per-reviewer verdicts with their files and hash, passed flags, status, recheck mode, limits
.claude/helix/critique-a.md  # reviewer-a's latest findings or approval
.claude/helix/critique-b.md  # reviewer-b's latest findings or approval
.claude/helix/ledger.jsonl   # usage against the budget; written only by the hooks
```

No agent writes a checkpoint's `passed` flag. Each reviewer records only its own verdict
in its own slot (`reviews.reviewer_a` / `reviews.reviewer_b`); the stop-gate **derives**
`passed = (reviewer-a approved AND reviewer-b approved AND both approvals match the files
on disk)` and persists it. The main agent may only change the top-level `status` (to
`in_progress` or `paused`). The guard hook enforces exactly that separation of roles.

### Approvals name the code they approved

A verdict on its own says nothing about _which_ code was approved. Without more, two
things would be left to the prompt rather than the tooling: after a rejection, a fix
re-sent to only the rejecting reviewer would be paired with the other reviewer's approval
of the older code; and a later checkpoint could rewrite a module an earlier checkpoint
passed on. So every approval carries two extra fields in the reviewer's slot:

```json
"reviewer_a": {
  "verdict": "approved",
  "tests_verified": true,
  "notes": "<one line>",
  "files": ["app.py", "tests/test_app.py"],
  "hash": "<hash of those files>"
}
```

- `files` is the list the reviewer was given: every file the checkpoint changed, plus its
  `test_target`. Both reviewers must record the same list.
- `hash` is computed by `helix-hash.js` over the paths and contents of those files (a
  deleted file counts as a change). The reviewer takes it _before_ reading any code, so
  an edit during the review is caught too:

  ```
  node ~/.claude/hooks/helix-hash.js app.py tests/test_app.py
  ```

The guard refuses to record an approval whose hash does not match the files on disk. The
stop-gate recomputes the hash whenever it derives `passed`; an approval that no longer
matches is sent back to `pending` with a note starting `RECHECK:`, and the checkpoint
needs a re-review (not a rebuild).

To see at any time which approvals in the current project are stale, run this from the
project root:

```
node ~/.claude/hooks/helix-hash.js --check
```

### Recheck mode: strict or deferred

A checkpoint that has not passed yet can never pass on a stale approval. For checkpoints
that _have_ passed, the top-level `recheck` field in `state.json` chooses when their
approvals are checked again:

| `recheck`              | When a passed checkpoint is rechecked                 | Trade-off                                                                                                              |
| ---------------------- | ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `"deferred"` (default) | Once, when every checkpoint has passed (a final gate) | Fewest reviews. A regression in an earlier checkpoint surfaces at the end.                                              |
| `"strict"`             | Every time the stop-gate runs; the orchestrator also checks after each approval | Catches a regression right after the checkpoint that caused it. Review count grows quadratically when files are shared. |

In both modes the run cannot complete while any approval is stale. Choose the mode as the
first word when you start a run (`/orchestrator strict <feature request>`), or change it
when you approve the plan (`/orchestrator approve deferred`); without either, it is
`"deferred"`. The guard locks it against the agent once the loop has started, but you can
still edit the field by hand. With `"strict"` on a plan where one file runs through every
checkpoint, consider raising `max_iterations`.

### The budget: pass limits, time limits, and the diff alarm

Two independent reviewers who must both approve, approvals that go stale when code changes,
and rechecks of earlier checkpoints can between them keep a run going for a long time: a
fix for one reviewer draws a new finding from the other, or a fix in one checkpoint makes
another stale, and so on. Without a budget, the only thing stopping such a run is you.

Every plan therefore carries a `limits` object, which the planner writes and you can edit
before approving:

```json
"max_iterations": 25,
"limits": {
  "max_review_rounds": 4,
  "max_total_review_rounds": null,
  "max_active_minutes": 240,
  "max_checkpoint_minutes": 60,
  "max_diff_lines": null,
  "max_checkpoint_diff_lines": 800
}
```

| Limit                       | What it caps                                                                                              | Default                          |
| --------------------------- | --------------------------------------------------------------------------------------------------------- | -------------------------------- |
| `max_review_rounds`         | Review rounds on one checkpoint, re-reviews (`RECHECK:`) included. A round is one verdict from each reviewer. | 4                              |
| `max_total_review_rounds`   | Review rounds in the whole run                                                                             | 3 per checkpoint (at least 6)    |
| `max_active_minutes`        | Active time for the whole run                                                                              | 240                              |
| `max_checkpoint_minutes`    | Active time on the checkpoint currently being built or re-reviewed                                         | 60                               |
| `max_diff_lines`            | **Diff alarm:** lines changed since the run started                                                        | 500 per checkpoint (at least 1000) |
| `max_checkpoint_diff_lines` | **Diff alarm:** how much the diff grew while the current checkpoint was being worked on                    | 800                              |
| `max_iterations` (top level) | Blocked stops in the whole run                                                                            | 25                               |

A `null` limit uses the default sized from the number of checkpoints.

How usage is measured:

- **Review rounds** are counted by the guard each time a reviewer records a verdict.
- **Active time** is measured from hook activity: the hooks record a tick at most once a
  minute while the run is `in_progress` or `paused`. A gap of more than 10 minutes with no
  tool activity counts as idle (a run waiting on you overnight) and is not counted.
- **Diff size** is lines added plus lines deleted against the commit the run started on
  (`git diff --numstat`), plus the lines of new untracked files, excluding `.claude/`. The
  checkpoint figure is the growth since that checkpoint became the current one. Outside a
  git repository the diff alarm is off.
- **Blocked stops** are counted for the whole run. Pausing does not reset the count.

Usage lives in `.claude/helix/ledger.jsonl`, an append-only log that only the hooks write.
A new plan from the planner starts a new ledger (the previous one is kept as
`ledger.prev.jsonl`). To see usage against the limits at any time, run from the project
root:

```
node ~/.claude/hooks/helix-budget.js
```

When any limit is spent:

1. The guard refuses to delegate to the test-author or either reviewer, and refuses to set
   the run back to `in_progress`. A reviewer already working may still record its verdict.
2. The stop-gate sets `status` to `"halted"`, records why in `halt_reason`, and lets the
   agent stop, with a message that says which limit was spent.
3. The orchestrator reports to you: the limit, the usage, which checkpoints passed, and the
   open findings on the one that did not.

Only you can resume a halted run. The guard denies any change to `limits` from the agent,
and it denies moving a halted run out of `halted` while the budget is still spent. To
continue, raise the spent limit (or `max_iterations`) in `state.json` by hand and run
`/orchestrator`. Or re-plan with narrower checkpoints, or stop and keep what passed.

### Single-concern modules keep reviews cheap

Because an approval goes stale when _any_ of its files changes, the cost of rechecking
depends on how much the checkpoints share files. The planner therefore gives each
checkpoint its own module, builds any wiring between modules early so that it discovers
new modules by itself (later checkpoints add files instead of editing a shared one), and
lists in each checkpoint's `touches` the source files it expects to change. When the
orchestrator presents the plan, it shows which files more than one checkpoint touches, so
you can see what `"strict"` would cost before you approve. This is guidance to the
planner, not a rule the hooks enforce: a checkpoint that edits another checkpoint's file
is still caught by the hash, it just costs an extra review.

---

## Claude Code (reference implementation)

### File layout

```
.claude/
  skills/orchestrator/SKILL.md   # the /orchestrator command (drives the whole workflow)
  agents/planner.md              # subagent: turns a request into checkpoints
  agents/test-author.md          # subagent: generates a checkpoint's tests from its criteria
  agents/reviewer-a.md           # subagent: independent reviewer A (own verdict + critique-a.md)
  agents/reviewer-b.md           # subagent: independent reviewer B (own verdict + critique-b.md)
  hooks/helix-guard.js           # PreToolUse hook (the guard)
  hooks/helix-stop.js            # Stop hook (the stop-gate)
  hooks/helix-hash.js            # shared approval hash: used by both hooks and run by the reviewers
  hooks/helix-budget.js          # shared budget: limits, usage ledger, diff alarm (used by both hooks)
  settings.json                  # registers the two hooks
  helix/state.json               # per-run plan + status (created when a run starts)
  helix/critique-a.md            # reviewer-a's review notes
  helix/critique-b.md            # reviewer-b's review notes
  helix/ledger.jsonl             # usage against the budget (written only by the hooks)
```

Put the skill, agents, hooks, and `settings.json` in your user directory
(`~/.claude/`) to make Helix available in **every** project, or in a project's
`.claude/` to scope it to one repo. The `helix/` state directory stays **per project**
either way — each project runs its own workflow.

`helix-hash.js` and `helix-budget.js` are not registered as hooks, but they must sit in the
same `hooks/` folder as the other two: both hooks load them from there, and the reviewers
and the orchestrator run them by that path.

### Register the hooks

Claude Code reads hook registrations from `settings.json`. The two Helix hooks attach to
the `PreToolUse` and `Stop` events:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit|Bash|PowerShell|Agent|Task",
        "hooks": [
          {
            "type": "command",
            "command": "node \"~/.claude/hooks/helix-guard.js\"",
            "timeout": 15
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node \"~/.claude/hooks/helix-stop.js\"",
            "timeout": 15
          }
        ]
      }
    ]
  }
}
```

Line by line:

- The `"PreToolUse"` array holds hooks that run **before** a tool call.
- The `"matcher"` limits the guard to the tools that could touch state (the file editors and the shells) and to delegation to subagents (`Agent`, called `Task` in older versions of Claude Code), so it can refuse new review work on a spent budget.
- The guard `"command"` runs the guard script through Node.
- The `"Stop"` array holds hooks that run **when the agent tries to end its turn**.
- The stop `"command"` runs the stop-gate script.

On Windows, use an absolute path such as
`node "C:\\Users\\you\\.claude\\hooks\\helix-guard.js"` if the tilde is not expanded in
your environment.

### What each hook must do (the contract)

The **stop-gate** (`Stop` event):

- If the run's `status` is not `in_progress`, allow the stop (planning, paused, and complete states hand control back to the user).
- **Derive** each checkpoint's `passed` from its two verdicts and persist it. A checkpoint newly passes only when both reviewers approved, both verified the tests, both recorded the same `files` (including the `test_target`), and the recorded `hash` still matches those files on disk. This — not any tool call — is what flips `passed`. (The hook writes state via `fs`, which does not pass through the guard.)
- Send an approval that no longer matches the code back to `pending`, with a note starting `RECHECK:`. For a checkpoint that has already passed, do this every time the hook runs when `recheck` is `"strict"`, and only at the final gate (once every checkpoint has passed) when it is `"deferred"`.
- If every checkpoint is `passed` and every approval is fresh, set `status` to `complete` and allow the stop.
- Otherwise **block** the stop (exit code 2) and print, to standard error, which checkpoint is next, which reviewers are still pending/rejected, which checkpoints need only a re-review, and to read both critique files. The checkpoint still being built comes before any re-review. Claude Code feeds that text back to the agent as the reason it cannot stop.
- Enforce the budget (see [The budget](#the-budget-pass-limits-time-limits-and-the-diff-alarm)). Record usage in the ledger, including each blocked stop; never reset the count on `paused`. If the run is `in_progress` or `paused` and any limit is spent, set `status` to `"halted"` with a `halt_reason` and allow the stop. Include the usage against the limits in every block message.

The **guard** (`PreToolUse` event) — enforced per role (the subagent's name; "main" is the builder):

- **planner**: may write the whole plan to `state.json` (only while `status` is `awaiting_approval`, with every checkpoint unapproved and both verdicts `pending`) and create the empty `critique-a.md` / `critique-b.md`. No source files.
- **reviewer-a / reviewer-b**: each may change _only_ its own verdict slot (`reviews.reviewer_a` or `reviews.reviewer_b`) in `state.json`, and write _only_ its own findings file (`critique-a.md` or `critique-b.md`). Any attempt to touch `passed`, `status`, the other reviewer's slot, another checkpoint, or a source file is denied. A new approval is also denied unless its `files` and `hash` match the files on disk at that moment. This keeps the two reviews **independent**, un-forgeable, and tied to the code they describe.
- **test-author**: may write _only_ test files (a file in a `tests/` directory, or `test_*.py` / `*_test.py` / `*.spec.*` / `conftest.py`). Every write under `.claude/helix/` is denied, so it can never approve anything or touch state.
- **main (the builder)**: during the planning phase (`status` is `awaiting_approval`, or before a state file exists) it may edit the plan freely _as long as_ nothing starts out approved (all `passed` false, `review_notes` empty, both verdicts `pending`), and this is when it may set `recheck`. Once `status` is `in_progress` the plan, including `recheck`, is frozen: the only change allowed to `state.json` is the top-level `status` (to `in_progress` or `paused`). Any Write/Edit to a `critique-*.md`, or any other write under `.claude/helix/`, is denied — this is the rule that stops the builder from approving its own work or rewriting a review.
- **Budget:** record a usage tick (at most once a minute) on any tool call, and a review round each time a reviewer records a verdict. While any limit is spent, deny delegation (`Agent`/`Task`) to the test-author and the reviewers, and deny the builder setting `status` to `in_progress` or moving it out of `halted`. Deny any change to `limits` from the builder; only the user changes them, by hand. When the planner writes a new plan, start a new ledger.
- Deny any shell command from **any** role that touches the `.claude/helix/` directory (this also stops one reviewer from reading the other's findings via the shell).
- The guard **fails closed**: if it hits an unexpected error it denies rather than allowing, so a bug can never silently wave a write through.
- Allow everything else.

Claude Code supports both of these because, in its hooks reference, a `PreToolUse` hook
that returns exit code 2 (or a JSON `permissionDecision` of `deny`) **"Blocks the tool
call"**, and a `Stop` hook that returns exit code 2 **"Prevents Claude from stopping,
continues the conversation."** 

### The subagents

A subagent is a Markdown file with YAML frontmatter followed by a system prompt, and
each subagent runs in its own context window. Helix uses four:

- **`planner`** — turns the request into checkpoints (each with a `test_target`, the
  source files it `touches`, and two `pending` reviewer slots), keeping each checkpoint in
  its own module where it can, and reports which files are shared.
- **`test-author`** — for a checkpoint, generates tests from its acceptance criteria into
  `test_target`, _before_ implementation. Restricted so it can write only test files.
- **`reviewer-a`** and **`reviewer-b`** — two independent reviewers with the same restricted
  toolset. Each runs in its own fresh context, judges the code against the acceptance
  criteria, runs the checkpoint's tests, and records **only its own** verdict and findings
  file, together with the files it reviewed and their hash. Neither is ever shown the
  other's notes — that independence is the whole point of running two.

`reviewer-b` is identical to `reviewer-a` except that it writes `reviews.reviewer_b` and
`critique-b.md`. A checkpoint passes only when both have approved the same files and those
files are unchanged since; the stop-gate derives that. The full system prompt of every subagent is in [Complete files](#complete-files).

### The orchestrator skill

A skill is a `SKILL.md` file that becomes a slash command; a skill at
`.claude/skills/orchestrator/SKILL.md` creates `/orchestrator`. Its body is
the workflow: delegate to the planner, present the plan for approval, then loop over
checkpoints — for the first unapproved one, delegate to the **test-author** to generate
its tests, implement it, run the tests yourself, then delegate to **reviewer-a and
reviewer-b independently and in parallel**, giving both the same file list. The checkpoint
advances only when the tests pass _and_ both reviewers approve; on any rejection the
builder fixes the findings and re-reviews with both. Repeat until every checkpoint is
approved by both reviewers. A checkpoint the stop-gate reopened with a `RECHECK:` note is
re-reviewed, not rebuilt; in `"strict"` mode the skill also runs
`node ~/.claude/hooks/helix-hash.js --check` after each approval to find stale earlier
checkpoints straight away. The full skill is in [Complete files](#complete-files).

### Run it

```
/orchestrator build an interactive tutor that teaches AI agent evaluations
```

To choose the recheck mode, put it first:

```
/orchestrator strict build an interactive tutor that teaches AI agent evaluations
```

Review the plan, including the files shared between checkpoints, then approve it. You can
change the mode at this point too:

```
/orchestrator approve deferred
```

The stop-gate keeps the agent working until both reviewers have approved every
checkpoint and every approval still matches the code, or until the budget is spent. Then
the run halts and the orchestrator tells you which limit was reached and what is still
open.

If a run pauses (the agent needs a decision from you), resume it with no arguments. If it
halted, first raise the spent limit in `.claude/helix/state.json` by hand, then resume:

```
/orchestrator
```

The mode word is read only as the first word of a request. A feature request that itself
starts with "strict" or "deferred" would be read as the mode, so start it with another
word.

## Complete files

These are the exact files of the working implementation. Save each one at the path shown,
under `~/.claude/` (every project) or a project's `.claude/` (one project), then register
the two hooks as shown in [Register the hooks](#register-the-hooks). `helix-hash.js` and
`helix-budget.js` need no registration; keep them beside the hooks.

### The `/orchestrator` skill

`.claude/skills/orchestrator/SKILL.md`

````markdown
---
name: orchestrator
description: Run the Helix plan/test/implement/review loop for a feature. Plans checkpoints, waits for user approval, then for each checkpoint generates tests, implements, and gets it independently approved by two reviewers before moving on.
argument-hint: [strict|deferred] <feature request> | approve [strict|deferred] | (empty to resume)
disable-model-invocation: true
---

# Helix Orchestrator

You are the master coordinator of the Helix workflow. State lives in `.claude/helix/state.json`; the two reviewers' findings live in `.claude/helix/critique-a.md` and `.claude/helix/critique-b.md`. A Stop hook blocks you from ending your turn while `status` is `"in_progress"` and any checkpoint is unpassed. A checkpoint's `passed` is derived by that hook and becomes true only when **both** independent reviewers approve it — you cannot set it yourself. Each approval records the files it covers and a hash of them; if any of those files changes afterwards, the hook sends that approval back to `pending` with a note starting `RECHECK:`.

Every run also has a **budget** in the state file's `limits` (review rounds per checkpoint and per run, active minutes per checkpoint and per run, and a diff alarm on how many lines the run and the current checkpoint have changed), plus `max_iterations` for blocked stops. When any of it is spent, the hooks refuse new test-author and reviewer work and the Stop hook sets `status` to `"halted"`. Only the user can raise a limit, by hand; you cannot change `limits` or move a halted run back to work. Check usage at any time with `node ~/.claude/hooks/helix-budget.js`.

Arguments: $ARGUMENTS

## Decide what to do

First, the recheck mode (see "Recheck mode" below): if the first word of the arguments, or the word right after `approve`, is `strict` or `deferred` (any case), that word is the mode the user chose. Remove it, then decide with what remains:

- **A feature request** → Phase 1, passing the mode to the planner.
- **`approve`**, or the user approves the plan in conversation → Phase 2.
- **Nothing** → read the state file and resume: `awaiting_approval` → show the plan and ask for approval; `paused` or `in_progress` → Phase 2; `halted` → see "When the budget is spent"; `complete` or missing → ask the user for a feature request.

A mode given while the state is `awaiting_approval` is applied in Phase 2 step 1. A mode given while the loop is `in_progress` or `paused` cannot be applied: the plan is locked. Tell the user in one line which mode the run is using and that they can change `recheck` in the state file by hand, then resume.

## Phase 1: Plan

1. Delegate to the `planner` subagent with the full feature request and, if the user gave them, the recheck mode and any limits.
2. Read `.claude/helix/state.json` and present to the user: the checkpoints as a numbered list with their acceptance criteria and `touches`, every file that more than one checkpoint touches, the recheck mode, and the budget (`limits` and `max_iterations`; a `null` limit is sized from the number of checkpoints, as `node ~/.claude/hooks/helix-budget.js` shows once the run starts). If several checkpoints share files, point out that `strict` will re-review those checkpoints repeatedly and `deferred` costs less.
3. Ask the user to review, edit the file if they wish (including the limits), and reply `approve` (optionally `approve strict` or `approve deferred` to change the mode). Then end your turn. (The hook allows this because status is `awaiting_approval`.)

## Phase 2: Implement loop

1. If the state is `awaiting_approval` and the user chose a recheck mode that differs from the plan's, set the top-level `"recheck"` to it first; it is locked once the loop starts. Then set `"status": "in_progress"` in the state file. From now on you cannot stop until every checkpoint passes.
2. Take the first checkpoint with `"passed": false` (if a reviewer's notes for it start with `RECHECK:`, follow "Recheck mode" instead of steps 2-5). Read `.claude/helix/critique-a.md` **and** `.claude/helix/critique-b.md`; if either has findings for this checkpoint, address every one of them.
3. **Generate tests first.** Set `"status": "paused"`, delegate to the `test-author` subagent with the checkpoint id, and set `"status": "in_progress"` again when it returns. It writes tests derived from the acceptance criteria to the checkpoint's `test_target`. (On a re-attempt where tests already exist and still fit, you may skip this, but never delete or weaken those tests yourself.)
4. **Implement** the checkpoint so those tests — and every acceptance criterion — are satisfied.
5. **Run the tests yourself.** Run `test_command` and any commands named in the acceptance criteria. Do not ask for review until they pass; the reviewers will reject a red or missing test.
6. **Review independently, in parallel.** Set `"status": "paused"` first (see "Waiting on subagents" below). Then, in a single message, delegate to **both** `reviewer-a` and `reviewer-b`, giving each only the checkpoint id and the **same** list of files: every file you created or changed for this checkpoint plus its `test_target`. The hook rejects approvals of differing file lists. Never pass one reviewer the other's notes, verdict, or critique file — their independence is the point. When both have returned, set `"status": "in_progress"` again before doing anything else.
7. Read both verdicts from `state.json` (`reviews.reviewer_a` and `reviews.reviewer_b`) and both critique files. The checkpoint advances **only when the tests pass AND both reviewers have `"verdict": "approved"`.** If either reviewer rejected, go back to step 2 for the **same** checkpoint: fix the findings, re-run tests, and re-delegate to both reviewers. One approval is never enough, and an approval given before your fix no longer counts: its hash is stale.
8. If `recheck` is `"strict"`, do the strict recheck described under "Recheck mode" now. Then move to the next unpassed checkpoint and repeat. When every checkpoint is passed (both reviewers approved each), the Stop hook marks the run complete; give the user a short summary of what was built, the reviewers' notes, and the budget used (`node ~/.claude/hooks/helix-budget.js`).

## Rules

- Never edit a `passed` flag, a `review_notes` field, or any `reviews.*` verdict yourself. Only the reviewers record verdicts, and only the Stop hook derives `passed` (both-approved). A PreToolUse hook enforces this: from the main session you may only change the top-level `status` (to `in_progress` or `paused`) with the Edit tool, you may not write any `critique-*.md`, and shell commands may not touch `.claude/helix/`. Read state with the Read tool.
- Never weaken acceptance criteria, delete/skip generated tests, or edit a reviewer's findings to get an approval.
- If you are truly blocked (missing credentials, an ambiguous requirement, a decision only the user can make), set `"status": "paused"`, explain exactly what you need, and end your turn. Do not use this to escape hard work.

## When the budget is spent

A hook denial that says the budget is spent, or a `"halted"` status, ends the work. Do not retry, work around it, or keep editing code to get under a limit.

1. If the run is `in_progress`, set `"status": "paused"` (the Stop hook then halts it).
2. Report to the user: which limit was spent (`halt_reason`, or the denial's text), the output of `node ~/.claude/hooks/helix-budget.js`, which checkpoints passed, and for the checkpoint that did not, the open findings from both critique files in a few lines each. If the reviewers keep rejecting for the same reason, or the diff alarm went off, say what you think is causing it (an acceptance criterion that cannot be met, reviewers who disagree, a fix that keeps growing).
3. Offer the user the choices: raise the spent limit by hand in `state.json` and run `/orchestrator` to resume; re-plan with narrower checkpoints; or stop and keep what passed.
4. End your turn.

When the user runs `/orchestrator` on a `halted` run, set `"status": "in_progress"` and continue Phase 2. If the guard denies it, the budget is still spent: tell the user which limit and end your turn.

## Recheck mode

Later checkpoints often change files an earlier checkpoint was approved on. The top-level `recheck` field decides when those earlier approvals are checked again:

- `"deferred"` (default): the Stop hook rechecks every checkpoint once all of them have passed. Stale ones come back as unpassed with `RECHECK:` notes, and the run cannot complete until they are re-approved.
- `"strict"`: recheck after every approval. After both reviewers approve a checkpoint, run `node ~/.claude/hooks/helix-hash.js --check`. For every earlier checkpoint it reports as `STALE`, re-review it before starting the next checkpoint. The Stop hook enforces the same rule whenever it runs.

A recheck is a re-review, not a rebuild. Always finish the checkpoint you are building before rechecking earlier ones. Then, for each stale checkpoint in id order: do not regenerate its tests or re-implement it; run the tests, then delegate to **both** reviewers as in step 6, naming the checkpoint id and the file list its earlier approval recorded (`reviews.*.files`). If a reviewer rejects, fix the findings and re-delegate to both; that fix may make other checkpoints stale in turn.

## Waiting on subagents

The Stop hook counts every blocked stop toward `max_iterations`, for the whole run (pausing does not reset the count). If the run is `in_progress` while a subagent works and you end your turn to wait for it, each wait burns an iteration, and the cap can run out before the work is done. So:

- Set `"status": "paused"` immediately before delegating to the `test-author` or the reviewers.
- Set `"status": "in_progress"` immediately after they return, before reading results or making any other change.
- Pausing is only for waiting on subagents or on the user. Never leave the run paused to skip work; while you are working on a checkpoint yourself, the status must be `in_progress`.
- If a session ends while paused, running `/orchestrator` with no arguments resumes the loop (Phase 2 sets `in_progress`).
````

### The `planner` subagent

`.claude/agents/planner.md`

````markdown
---
name: planner
description: Helix checkpoint planner. Breaks a feature request into ordered, independently reviewable checkpoints and writes them to .claude/helix/state.json. Use when the Helix orchestrator starts a new feature.
tools: Read, Grep, Glob, Bash, Write
---

# Checkpoint Planner

You are a task-planning sub-agent. You do not write feature code.

## Process

1. Explore the codebase (Read, Grep, Glob, read-only Bash) to understand the stack, conventions, test setup, and where the feature will live.
2. Break the requested feature into sequential checkpoints, ordered from the simplest foundation to the most complex. Aim for 3-8 checkpoints. Each one must be small enough to implement and review in one pass, and leave the project in a working state.
3. **Give each checkpoint its own module.** Each review is bound to a hash of the files the checkpoint changed, so when a later checkpoint edits an earlier checkpoint's file, that earlier checkpoint has to be reviewed again. Plan the code so each checkpoint's concern lives in files no other checkpoint edits:
   - If the modules must be connected (routes, plugins, templates, configuration), have the first checkpoint that needs it build wiring that discovers modules by itself (for example, registering every module found in a package or folder). Later checkpoints then add files instead of editing a shared one.
   - Where a later checkpoint must still edit an earlier checkpoint's file, keep that edit small and say so in its description.
   - One concern per module, not one function per file: do not split a single concern across files just to avoid sharing.
   - Fit the project's existing structure. Do not plan restructuring of existing code the request does not ask for.
4. Give every checkpoint concrete, verifiable acceptance criteria (a command that passes, a behavior that can be observed, a file that exists). Avoid vague criteria like "code is clean". These criteria are what the test-author turns into tests and what both reviewers check against, so make them testable. Keep at least one checkpoint whose criteria test the modules working together.
5. Choose a `test_target` for each checkpoint: the path where the test-author will write that checkpoint's tests, following the project's convention (for a Python/pytest project, e.g. `tests/test_<slug>.py`). List in `touches` the source files the checkpoint is expected to create or change (not its `test_target`).
6. Write the plan to `.claude/helix/state.json` (create the directory if needed), overwriting any previous plan, using exactly this shape:

```json
{
  "feature": "<the original request>",
  "status": "awaiting_approval",
  "max_iterations": 25,
  "recheck": "deferred",
  "limits": {
    "max_review_rounds": 4,
    "max_total_review_rounds": null,
    "max_active_minutes": 240,
    "max_checkpoint_minutes": 60,
    "max_diff_lines": null,
    "max_checkpoint_diff_lines": 800
  },
  "test_command": "<command that runs the project's tests, or null>",
  "checkpoints": [
    {
      "id": 1,
      "title": "<short name>",
      "description": "<what to build and where>",
      "acceptance_criteria": ["<verifiable criterion>", "..."],
      "test_target": "tests/test_<slug>.py",
      "touches": ["<source file this checkpoint creates or changes>", "..."],
      "reviews": {
        "reviewer_a": { "verdict": "pending", "tests_verified": false, "notes": "", "files": [], "hash": "" },
        "reviewer_b": { "verdict": "pending", "tests_verified": false, "notes": "", "files": [], "hash": "" }
      },
      "passed": false,
      "review_notes": ""
    }
  ]
}
```

7. Also create an empty `.claude/helix/critique-a.md` and `.claude/helix/critique-b.md` (one per reviewer).

## Rules

- Set `recheck` to the mode the orchestrator gives you (`"strict"` or `"deferred"`); if it gives none, use `"deferred"`. It controls when earlier checkpoints are re-reviewed after their files change; the user may switch it before approving.
- Write `limits` as shown, replacing any value the orchestrator gives you. Each is a positive integer or `null` (`null` sizes it from the number of checkpoints). Size the checkpoints to fit: one checkpoint should not need more than about `max_checkpoint_diff_lines` changed lines or `max_checkpoint_minutes` of work; split it if it would.
- `status` must be `"awaiting_approval"`. Never set it to anything else; the orchestrator does that after the user approves.
- Every checkpoint starts with `"passed": false`, empty `review_notes`, and both reviewer verdicts `"pending"`. The Helix guard will reject a plan where anything starts out approved.
- Do not modify any file outside `.claude/helix/`.
- Finish by returning a short numbered summary of the checkpoints, then list every file that appears in more than one checkpoint's `touches` and which checkpoints share it (or say there are no shared files).
````

### The `test-author` subagent

`.claude/agents/test-author.md`

````markdown
---
name: test-author
description: Helix test-case author. For a given checkpoint, generates or augments tests from its acceptance criteria (and any reference code) and writes them into the project's test suite at the checkpoint's test_target. Runs BEFORE implementation and review. Writes ONLY test files — it can never approve a checkpoint or edit Helix state.
tools: Read, Grep, Glob, Bash, Write, Edit
---

# Test-Case Author

You turn a checkpoint's acceptance criteria into real, runnable tests. You do **not**
implement the feature, and you do **not** judge or approve anything. Your tests are the
bar the implementation must clear and the reviewers will run.

## Process

1. Read `.claude/helix/state.json`. The target checkpoint is the id the caller names, or
   else the first checkpoint with `"passed": false`. Read its `description`,
   `acceptance_criteria`, and `test_target`, plus the top-level `test_command`.
2. Learn the project's test conventions: read `test_command`, the existing files under the
   test directory, and any reference code the checkpoint description points to. Match the
   framework, imports, fixtures, and style already in use (for a Python/pytest project,
   write `pytest` tests; for others, follow what is there).
3. Write tests to the checkpoint's `test_target` that encode **each** acceptance criterion
   as one or more assertions. Favor behavioral / integration-style tests that describe the
   feature from the user's perspective, and deliberately include edge cases beyond the
   happy path (empty input, invalid input, boundaries, error paths) — not just the case
   that obviously works.
4. Be **additive and non-destructive**: create or extend `test_target`; never delete,
   weaken, skip, or loosen existing passing tests to make room. If tests for this checkpoint
   already exist, augment coverage rather than duplicating.
5. Make the tests syntactically valid and importable so they can run, even though they are
   expected to **fail until the feature is implemented** (that is normal — you write the
   tests first). You may run `test_command` to confirm the file is collected/parses; do not
   try to make it green by touching source.

## Rules

- You may write **only test files** — the checkpoint's `test_target` and other files in the
  project's test directory (e.g. `test_*.py`, `conftest.py`). The Helix guard will DENY any
  attempt to write source files, `state.json`, or any `critique-*.md`. You cannot approve a
  checkpoint or set any flag; that is by design.
- Never edit anything under `.claude/helix/`.
- Finish by returning a short summary: which file(s) you wrote, and how each acceptance
  criterion maps to a test.
````

### The `reviewer-a` subagent

`.claude/agents/reviewer-a.md`

````markdown
---
name: reviewer-a
description: Helix independent reviewer A. One of two context-isolated adversarial reviewers. Verifies the current checkpoint against its acceptance criteria and the generated tests, then records ONLY its own verdict in .claude/helix/state.json (reviews.reviewer_a) and writes findings to .claude/helix/critique-a.md. A checkpoint passes only when BOTH reviewers approve.
tools: Read, Grep, Glob, Bash, Write, Edit
---

# Independent Reviewer A

You are **reviewer-a**, one of two independent reviewers in the Helix workflow. You are a
merciless code reviewer. You do not write or fix feature code. Your job is to find reasons
the current checkpoint is NOT done. You start with no knowledge of how the code was
written, so judge only what is actually on disk.

You have a twin, **reviewer-b**, reviewing the same checkpoint in a separate context. You
must reach your verdict **independently**:

- Do NOT read `.claude/helix/critique-b.md` or reviewer-b's `reviews.reviewer_b` slot.
- Do NOT try to infer or wait for reviewer-b's verdict. Judge the code yourself.

## Process

1. Read `.claude/helix/state.json`. The checkpoint under review is the first one with
   `"passed": false`, unless the caller named a specific id. Note its `test_target` and the
   top-level `test_command`.
2. Identify the changes for that checkpoint (use `git diff` / `git status` if this is a git
   repo; otherwise read the files the caller lists and the files the checkpoint description
   points to).
3. **Tests gate (do this before anything else):** confirm the checkpoint's tests exist at
   `test_target` and actually exercise the acceptance criteria — they must not be missing,
   empty, trivially always-passing, or weakened/commented-out. Then run `test_command`
   yourself. **If the tests are missing/inadequate, or if any test fails, you must REJECT.**
   Tests passing is a precondition for approval; never approve around a red or absent test.
4. Verify every acceptance criterion yourself. Run any other command a criterion names. Do
   not trust claims in comments or in the caller's message.
5. Review for correctness bugs, unhandled edge cases, security issues, missing error
   handling at boundaries, broken existing behavior, and code that ignores the project's
   conventions.

## Verdict

Record your verdict by editing **only** your own slot in `.claude/helix/state.json`. For the
checkpoint under review, set:

```json
"reviews": {
  "reviewer_a": { "verdict": "approved", "tests_verified": true, "notes": "<one line>", "files": ["<path>", "..."], "hash": "<hash>" }
}
```

- `files` and `hash` name the code your verdict describes. **Before you read any code**, run
  `node ~/.claude/hooks/helix-hash.js <file> [<file> ...]` from the project root with exactly
  the files the caller listed plus the checkpoint's `test_target`, and copy the `files` and
  `hash` it prints. Do not add or drop files: both reviewers must record the same list. The
  guard denies an approval whose hash does not match the files on disk; if that happens a
  file changed during your review, so review again before approving.

- `verdict` is `"approved"` or `"rejected"`.
- `tests_verified` is `true` only if you ran the tests and they passed.
- `notes` is a one-line summary of what you verified or the key reason for rejection.

Do the edit with a single Edit that targets your `reviewer_a` object. **Never** touch
`passed`, `review_notes`, `status`, any other checkpoint, or the `reviewer_b` slot — the
Helix guard will deny it and your review will fail.

**If you reject**, also overwrite `.claude/helix/critique-a.md` with:

```markdown
# Critique (reviewer-a): Checkpoint <id> - <title>
Verdict: REJECTED

## Findings
1. `path/to/file.ext:<line>` - <what is wrong and why it matters>
...

## Unmet acceptance criteria
- <criterion> - <evidence>
```

**If you approve**, overwrite `.claude/helix/critique-a.md` with `Verdict: APPROVED` and a
one-line note of what you verified (including that the tests passed).

You do **not** set `passed`. The Stop hook derives `passed` from both reviewers' verdicts;
a checkpoint passes only when reviewer-a AND reviewer-b have both approved.

## Rules

- Only edit `.claude/helix/state.json` (your slot only) and `.claude/helix/critique-a.md`.
  Never touch source files, `critique-b.md`, or the other reviewer's slot.
- Never approve if a criterion is unverified or a test is failing/absent. Unverified means
  rejected.
- Return your verdict and the findings list to the caller.
````

### The `reviewer-b` subagent

`.claude/agents/reviewer-b.md`

````markdown
---
name: reviewer-b
description: Helix independent reviewer B. One of two context-isolated adversarial reviewers. Verifies the current checkpoint against its acceptance criteria and the generated tests, then records ONLY its own verdict in .claude/helix/state.json (reviews.reviewer_b) and writes findings to .claude/helix/critique-b.md. A checkpoint passes only when BOTH reviewers approve.
tools: Read, Grep, Glob, Bash, Write, Edit
---

# Independent Reviewer B

You are **reviewer-b**, one of two independent reviewers in the Helix workflow. You are a
merciless code reviewer. You do not write or fix feature code. Your job is to find reasons
the current checkpoint is NOT done. You start with no knowledge of how the code was
written, so judge only what is actually on disk.

You have a twin, **reviewer-a**, reviewing the same checkpoint in a separate context. You
must reach your verdict **independently**:

- Do NOT read `.claude/helix/critique-a.md` or reviewer-a's `reviews.reviewer_a` slot.
- Do NOT try to infer or wait for reviewer-a's verdict. Judge the code yourself.

## Process

1. Read `.claude/helix/state.json`. The checkpoint under review is the first one with
   `"passed": false`, unless the caller named a specific id. Note its `test_target` and the
   top-level `test_command`.
2. Identify the changes for that checkpoint (use `git diff` / `git status` if this is a git
   repo; otherwise read the files the caller lists and the files the checkpoint description
   points to).
3. **Tests gate (do this before anything else):** confirm the checkpoint's tests exist at
   `test_target` and actually exercise the acceptance criteria — they must not be missing,
   empty, trivially always-passing, or weakened/commented-out. Then run `test_command`
   yourself. **If the tests are missing/inadequate, or if any test fails, you must REJECT.**
   Tests passing is a precondition for approval; never approve around a red or absent test.
4. Verify every acceptance criterion yourself. Run any other command a criterion names. Do
   not trust claims in comments or in the caller's message.
5. Review for correctness bugs, unhandled edge cases, security issues, missing error
   handling at boundaries, broken existing behavior, and code that ignores the project's
   conventions.

## Verdict

Record your verdict by editing **only** your own slot in `.claude/helix/state.json`. For the
checkpoint under review, set:

```json
"reviews": {
  "reviewer_b": { "verdict": "approved", "tests_verified": true, "notes": "<one line>", "files": ["<path>", "..."], "hash": "<hash>" }
}
```

- `files` and `hash` name the code your verdict describes. **Before you read any code**, run
  `node ~/.claude/hooks/helix-hash.js <file> [<file> ...]` from the project root with exactly
  the files the caller listed plus the checkpoint's `test_target`, and copy the `files` and
  `hash` it prints. Do not add or drop files: both reviewers must record the same list. The
  guard denies an approval whose hash does not match the files on disk; if that happens a
  file changed during your review, so review again before approving.

- `verdict` is `"approved"` or `"rejected"`.
- `tests_verified` is `true` only if you ran the tests and they passed.
- `notes` is a one-line summary of what you verified or the key reason for rejection.

Do the edit with a single Edit that targets your `reviewer_b` object. **Never** touch
`passed`, `review_notes`, `status`, any other checkpoint, or the `reviewer_a` slot — the
Helix guard will deny it and your review will fail.

**If you reject**, also overwrite `.claude/helix/critique-b.md` with:

```markdown
# Critique (reviewer-b): Checkpoint <id> - <title>
Verdict: REJECTED

## Findings
1. `path/to/file.ext:<line>` - <what is wrong and why it matters>
...

## Unmet acceptance criteria
- <criterion> - <evidence>
```

**If you approve**, overwrite `.claude/helix/critique-b.md` with `Verdict: APPROVED` and a
one-line note of what you verified (including that the tests passed).

You do **not** set `passed`. The Stop hook derives `passed` from both reviewers' verdicts;
a checkpoint passes only when reviewer-a AND reviewer-b have both approved.

## Rules

- Only edit `.claude/helix/state.json` (your slot only) and `.claude/helix/critique-b.md`.
  Never touch source files, `critique-a.md`, or the other reviewer's slot.
- Never approve if a criterion is unverified or a test is failing/absent. Unverified means
  rejected.
- Return your verdict and the findings list to the caller.
````

### The guard (`PreToolUse` hook)

`.claude/hooks/helix-guard.js`

```js
#!/usr/bin/env node
// Helix Guard: PreToolUse hook that enforces who may write what in a Helix run.
//
// Roles (by subagent name; "main" = the builder / orchestrator session):
//   planner      - writes the whole plan to state.json while status is
//                  "awaiting_approval" (all checkpoints unapproved), and creates the
//                  empty critique-a.md / critique-b.md. No source files.
//   reviewer-a   - may change ONLY reviews.reviewer_a in state.json, and write ONLY
//                  critique-a.md. Never passed/status/other slots/source. An approval
//                  must carry the reviewed "files" and a "hash" that matches them on disk.
//   reviewer-b   - may change ONLY reviews.reviewer_b in state.json, and write ONLY
//                  critique-b.md. Never passed/status/other slots/source.
//   test-author  - may write ONLY test files. Denied every .claude/helix/ write, so it
//                  can never approve anything.
//   main         - may change ONLY the top-level "status" (in_progress/paused) in
//                  state.json. Never passed, reviews, review_notes, or any findings file.
//
// "passed" is never set through a tool call. The Stop hook derives it from the two
// reviewers' verdicts (a checkpoint passes only when BOTH approve) and writes it via fs,
// which does not pass through this hook.
//
// No shell command from any role may touch .claude/helix/.
//
// Budget (helix-budget.js): this hook also records usage in the hooks' ledger (a tick on
// tool activity, each reviewer verdict as a review round), and refuses to go past a spent
// budget: no delegation to the test-author or a reviewer, and no move back to
// "in_progress", while any limit is spent. Only the user may change "limits" (by hand).

const fs = require("fs");
const path = require("path");

const REVIEWERS = {
  "reviewer-a": { slot: "reviewer_a", file: "critique-a.md" },
  "reviewer-b": { slot: "reviewer_b", file: "critique-b.md" },
};
const MAIN_STATUSES = new Set(["awaiting_approval", "in_progress", "paused"]);
const VERDICTS = new Set(["pending", "approved", "rejected"]);
const SLOT_KEYS = new Set(["verdict", "tests_verified", "notes", "files", "hash"]);
const RECHECK_MODES = new Set(["deferred", "strict"]);
const { slotProblem } = require("./helix-hash.js");
const budget = require("./helix-budget.js");
const BUDGETED_AGENTS = new Set(["test-author", "reviewer-a", "reviewer-b"]);

function allow() {
  process.exit(0);
}

function deny(reason) {
  process.stdout.write(
    JSON.stringify({
      hookSpecificOutput: {
        hookEventName: "PreToolUse",
        permissionDecision: "deny",
        permissionDecisionReason: `Helix guard: ${reason}`,
      },
    })
  );
  process.exit(0);
}

// Fail CLOSED: any unexpected error becomes a deny, never a silent allow. (A hook that
// merely crashes with a non-zero exit is treated as a non-blocking error and would let
// the tool call through — exactly what this guard must prevent.)
process.on("uncaughtException", (e) => {
  deny(`internal error, denying to stay safe: ${(e && e.message) || e}`);
});

let input;
try {
  input = JSON.parse(fs.readFileSync(0, "utf8"));
} catch {
  allow();
}

// "main" is the builder/orchestrator session. agent_type is only set inside a subagent.
const role = input.agent_id ? input.agent_type || "" : "main";

const projectDir = process.env.CLAUDE_PROJECT_DIR || input.cwd || process.cwd();
const helixDir = path.resolve(projectDir, ".claude", "helix");
const statePath = path.join(helixDir, "state.json");
const norm = (p) => path.resolve(input.cwd || projectDir, p).toLowerCase();
const helixFile = (name) => path.join(helixDir, name).toLowerCase();

const tool = input.tool_name;
const ti = input.tool_input || {};

function readState() {
  try {
    return JSON.parse(fs.readFileSync(statePath, "utf8"));
  } catch {
    return undefined;
  }
}

// Usage bookkeeping never blocks a tool call (tick() never throws).
if (budget.tickDue(projectDir)) budget.tick(projectDir, readState());

// --- Delegation: no new test or review work on a spent budget. --------------------------
if (tool === "Agent" || tool === "Task") {
  if (!BUDGETED_AGENTS.has(ti.subagent_type)) allow();
  const state = readState();
  if (!state) allow();
  if (state.status === "halted") {
    deny(`the run is halted (${state.halt_reason || "budget spent"}). Do not delegate more work. Report the halt to the user and end your turn. ${budget.HALT_HELP}`);
  }
  if (budget.ACTIVE.has(state.status)) {
    const { problems } = budget.evaluate(projectDir, state);
    if (problems.length) {
      deny(`the run's budget is spent: ${problems.join("; ")}. Do not delegate more work. Set "status" to "paused", report this to the user, and end your turn; the stop-gate will halt the run. ${budget.HALT_HELP}`);
    }
  }
  allow();
}

// --- Shell: nobody may touch .claude/helix/ from a shell. ------------------------------
if (tool === "Bash" || tool === "PowerShell") {
  if (/\.claude[\\/]+helix/i.test(ti.command || "")) {
    deny(
      'shell commands may not touch .claude/helix/. Use the Read tool to inspect state, and the Edit tool to change it where your role allows.'
    );
  }
  allow();
}

if (!ti.file_path) allow();
const target = norm(ti.file_path);
const isHelix =
  target === statePath.toLowerCase() ||
  target.startsWith(helixDir.toLowerCase() + path.sep);

// --- test-author: test files only; never anything under .claude/helix/. ----------------
if (role === "test-author") {
  if (isHelix) {
    deny(
      "the test-author may write only test files, never state.json or any findings file. It cannot approve a checkpoint."
    );
  }
  if (!isTestFile(target)) {
    deny(
      "the test-author may write only test files (a file in a tests/ directory, or test_*.py / *_test.py / *.spec.* / conftest.py). It may not modify source files."
    );
  }
  allow();
}

// --- Non-helix files. -------------------------------------------------------------------
if (!isHelix) {
  // Only the builder writes project/source files. Reviewers and the planner do not.
  if (role === "reviewer-a" || role === "reviewer-b") {
    deny("reviewers do not modify project files; write only your own critique file and verdict.");
  }
  if (role === "planner") {
    deny("the planner may write only inside .claude/helix/ (the plan and empty critique files).");
  }
  allow(); // main builder, or any other subagent, may edit source.
}

// --- From here: a write to some file under .claude/helix/. ------------------------------

// Reviewers: their own critique file, or their own slot in state.json. Nothing else.
if (REVIEWERS[role]) {
  const { slot, file } = REVIEWERS[role];
  if (target === helixFile(file)) allow();
  if (target !== statePath.toLowerCase()) {
    deny(`reviewer ${role} may write only ${file} and its own verdict slot in state.json.`);
  }
  const { before, after } = computeStateChange();
  if (after === undefined) allow(); // the tool call would fail on its own; let it.
  const problem = onlyReviewerSlotChanged(before, after, slot);
  if (problem) deny(problem);
  recordVerdicts(before, after, slot);
  allow();
}

// Planner: creates the plan and the empty critique files.
if (role === "planner") {
  if (target === helixFile("critique-a.md") || target === helixFile("critique-b.md")) allow();
  if (target !== statePath.toLowerCase()) {
    deny("the planner may write only state.json, critique-a.md, and critique-b.md.");
  }
  const { after } = computeStateChange();
  if (after === undefined) allow();
  if (after.status !== "awaiting_approval") {
    deny('the planner must leave status as "awaiting_approval"; the orchestrator starts the loop after you approve.');
  }
  const bad = notAllUnapproved(after) || badRecheck(after) || budget.badLimits(after);
  if (bad) deny(bad);
  budget.resetLedger(projectDir); // a new plan is a new run with a fresh budget
  allow();
}

// Main builder (and any untrusted subagent, e.g. a deprecated critic): only status may
// change in state.json; no findings file may be written.
if (target !== statePath.toLowerCase()) {
  deny(
    "only reviewers write findings under .claude/helix/. From the main session you may change only the top-level \"status\" in state.json (with the Edit tool)."
  );
}

const { before, after } = computeStateChange();
if (after === undefined) allow();

if (before && before.status === "halted" && after.status !== "halted") {
  const { problems } = budget.evaluate(projectDir, after);
  if (problems.length) {
    deny(`the run is halted and its budget is still spent: ${problems.join("; ")}. Only the user can resume it. ${budget.HALT_HELP}`);
  }
}

if (before && !deepEqual(before.limits, after.limits)) {
  deny('only the user may change "limits", by editing state.json by hand.');
}

if (!MAIN_STATUSES.has(after.status)) {
  deny(
    `status may only be set to ${[...MAIN_STATUSES].join(", ")}. The Stop hook marks the workflow complete.`
  );
}

if (after.status === "in_progress" && (!before || before.status !== "in_progress")) {
  const { problems } = budget.evaluate(projectDir, after);
  if (problems.length) {
    deny(`the run's budget is spent: ${problems.join("; ")}. Leave it paused, report this to the user, and end your turn. ${budget.HALT_HELP}`);
  }
}

if (!before || before.status === "awaiting_approval") {
  // Planning phase: the builder may edit the plan and flip status to in_progress, as long
  // as nothing starts out approved. This is also where "recheck" may be chosen.
  const bad = notAllUnapproved(after) || badRecheck(after) || budget.badLimits(after);
  if (bad) deny(bad);
  allow();
}

// Loop running or paused: the plan is frozen; only "status" may change.
const stripStatus = (s) => JSON.stringify({ ...s, status: undefined });
if (stripStatus(before) !== stripStatus(after)) {
  deny(
    'once the loop has started, only "status" (in_progress/paused) may change from the main session. Checkpoints, criteria, reviews, passed flags and review_notes are locked. Reviewers record verdicts; the Stop hook derives "passed".'
  );
}
allow();

// ======================================================================================
// helpers
// ======================================================================================

// Each verdict a reviewer records (approved or rejected) is one review by that reviewer.
function recordVerdicts(before, after, slot) {
  const ba = cps(before);
  cps(after).forEach((c, i) => {
    const prev = ((ba[i] || {}).reviews || {})[slot] || {};
    const s = (c.reviews || {})[slot] || {};
    if (s.verdict !== "pending" && !deepEqual(prev, s)) {
      budget.record(projectDir, { ev: "verdict", cp: c.id, slot, verdict: s.verdict });
    }
  });
}

function isTestFile(p) {
  const b = path.basename(p);
  const parts = p.split(/[\\/]+/);
  const inTestsDir = parts.includes("tests") || parts.includes("test");
  const looksTest =
    /^test_.*\.py$/.test(b) ||
    /_test\.py$/.test(b) ||
    /\.(test|spec)\.[jt]sx?$/.test(b) ||
    b === "conftest.py";
  return inTestsDir || looksTest;
}

function applyEdit(text, { old_string, new_string, replace_all }) {
  if (!old_string || !text.includes(old_string)) return null; // the tool itself will fail
  if (replace_all) return text.split(old_string).join(new_string);
  const i = text.indexOf(old_string);
  return text.slice(0, i) + new_string + text.slice(i + old_string.length);
}

// Compute the parsed before/after state for a write to state.json.
// Returns { before, after }. `after` is undefined if we cannot compute a proposed value
// (let the tool fail on its own) and the caller should allow.
function computeStateChange() {
  const current = fs.existsSync(statePath) ? fs.readFileSync(statePath, "utf8") : "";
  let nextText;
  if (tool === "Write") {
    nextText = ti.content;
  } else if (tool === "Edit") {
    nextText = applyEdit(current, ti);
  } else if (tool === "MultiEdit") {
    nextText = current;
    for (const e of ti.edits || []) nextText = nextText === null ? null : applyEdit(nextText, e);
  } else {
    deny(`${tool} may not modify state.json.`);
  }
  if (nextText === null || nextText === undefined) return { before: undefined, after: undefined };

  let before;
  try {
    if (current) before = JSON.parse(current);
  } catch {
    before = undefined;
  }
  let after;
  try {
    after = JSON.parse(nextText);
  } catch (err) {
    deny(`the edit would leave state.json as invalid JSON (${err.message}).`);
  }
  return { before, after };
}

// Declared as a hoisted function (not a const arrow) so the validation helpers above can
// call it during top-level execution without a temporal-dead-zone crash.
function cps(s) {
  return s && Array.isArray(s.checkpoints) ? s.checkpoints : [];
}

// Returns a deny-reason string if any checkpoint is approved / non-pending; else "".
function notAllUnapproved(after) {
  for (const c of cps(after)) {
    const r = c.reviews || {};
    const a = r.reviewer_a || {};
    const b = r.reviewer_b || {};
    if (
      c.passed !== false ||
      (c.review_notes || "") !== "" ||
      a.verdict !== "pending" ||
      b.verdict !== "pending"
    ) {
      return `checkpoint ${c.id} must start unapproved: "passed": false, empty review_notes, and both reviewer verdicts "pending". Only the reviewers review, and only the Stop hook flips "passed".`;
    }
  }
  return "";
}

// Returns a deny-reason string if "recheck" is set to something the Stop hook would
// silently treat as deferred; else "".
function badRecheck(after) {
  if (after.recheck === undefined || RECHECK_MODES.has(after.recheck)) return "";
  return `"recheck" must be one of: ${[...RECHECK_MODES].join(", ")}.`;
}

function deepEqual(a, b) {
  if (a === b) return true;
  if (typeof a !== typeof b) return false;
  if (a === null || b === null) return a === b;
  if (Array.isArray(a) !== Array.isArray(b)) return false;
  if (Array.isArray(a)) return a.length === b.length && a.every((x, i) => deepEqual(x, b[i]));
  if (typeof a === "object") {
    const ka = Object.keys(a);
    const kb = Object.keys(b);
    if (ka.length !== kb.length) return false;
    return ka.every((k) => Object.prototype.hasOwnProperty.call(b, k) && deepEqual(a[k], b[k]));
  }
  return false;
}

// True if checkpoints a and b are identical except (possibly) reviews[slot].
function cpEqualExceptSlot(a, b, slot) {
  const ca = JSON.parse(JSON.stringify(a));
  const cb = JSON.parse(JSON.stringify(b));
  if (ca.reviews) delete ca.reviews[slot];
  if (cb.reviews) delete cb.reviews[slot];
  return deepEqual(ca, cb);
}

// Returns a deny-reason if the change is anything other than this reviewer's own slot.
function onlyReviewerSlotChanged(before, after, slot) {
  if (!before) return "state.json does not exist yet; a reviewer cannot create it.";
  if (after.status !== before.status) return "a reviewer may not change status.";
  const ba = cps(before);
  const aa = cps(after);
  if (ba.length !== aa.length) return "a reviewer may not add or remove checkpoints.";
  for (let i = 0; i < ba.length; i++) {
    if (ba[i].id !== aa[i].id) return "a reviewer may not reorder or re-id checkpoints.";
    if (!cpEqualExceptSlot(ba[i], aa[i], slot)) {
      return `you may change only your own reviews.${slot} entry (verdict/tests_verified/notes) for checkpoint ${aa[i].id} — not "passed", not review_notes, not the other reviewer's slot, not any other field.`;
    }
    const s = (aa[i].reviews || {})[slot] || {};
    if (Object.keys(s).some((k) => !SLOT_KEYS.has(k))) {
      return `reviews.${slot} may contain only verdict, tests_verified, notes, files, and hash.`;
    }
    if (!VERDICTS.has(s.verdict)) {
      return `reviews.${slot}.verdict must be one of: ${[...VERDICTS].join(", ")}.`;
    }
    // A new approval must name the code it approves, as that code is on disk right now.
    const prev = (ba[i].reviews || {})[slot] || {};
    if (s.verdict === "approved" && !deepEqual(prev, s)) {
      const stale = slotProblem(projectDir, after, aa[i], s);
      if (stale) {
        return `cannot record an approval for checkpoint ${aa[i].id}: ${stale}. Run \`node ~/.claude/hooks/helix-hash.js <files>\` and record its "files" and "hash" exactly; if a file changed during your review, review it again first.`;
      }
    }
  }
  return null;
}
```

### The stop-gate (`Stop` hook)

`.claude/hooks/helix-stop.js`

```js
#!/usr/bin/env node
// Helix Enforcer: Stop hook that blocks Claude from ending its turn while any checkpoint
// in .claude/helix/state.json is still unpassed.
//
// A checkpoint's "passed" is DERIVED here, never set by a tool call: it becomes true only
// when BOTH reviewer-a and reviewer-b have approved it AND those approvals still describe
// the code (same file list, hash unchanged; see helix-hash.js). This hook writes state.json
// via fs (it does not go through the PreToolUse guard), so it is the single authority that
// flips "passed", sends a stale approval back to "pending", and marks the run complete.
//
// state.recheck selects when an already-passed checkpoint is rechecked for staleness:
//   "strict"   - on every derive, so a later checkpoint that rewrites its files reopens it.
//   "deferred" - (default) only once every checkpoint has passed, as a final gate.
// In both modes a checkpoint cannot newly pass on a stale approval, and the run cannot
// complete while any approval is stale.
//
// The run also has a budget (helix-budget.js): review rounds, active time, a diff alarm,
// and a cap on blocked stops. When any of it is spent, this hook sets status to "halted"
// and lets Claude stop, so the run goes back to the user instead of looping. The count of
// blocked stops lives in the hooks' ledger, so pausing to wait for subagents does not
// reset it.
//
// Exit 0 -> Claude may stop.
// Exit 2 -> Claude is blocked; stderr is fed back to it as the reason.

const fs = require("fs");
const path = require("path");
const { checkCheckpoint } = require("./helix-hash.js");
const { RECHECK, isRecheck, tick, evaluate, summary, record, HALT_HELP } = require("./helix-budget.js");

const projectDir = process.env.CLAUDE_PROJECT_DIR || process.cwd();
const helixDir = path.join(projectDir, ".claude", "helix");
const statePath = path.join(helixDir, "state.json");
const counterPath = path.join(helixDir, ".iterations");
const DEFAULT_MAX_ITERATIONS = 25;

function allow(message) {
  if (message) process.stdout.write(JSON.stringify({ systemMessage: message }));
  process.exit(0);
}

function block(reason) {
  process.stderr.write(reason);
  process.exit(2);
}

function readCounter() {
  try {
    return parseInt(fs.readFileSync(counterPath, "utf8"), 10) || 0;
  } catch {
    return 0;
  }
}

function resetCounter() {
  try {
    fs.unlinkSync(counterPath);
  } catch {}
}

function reviewerState(cp) {
  const r = cp.reviews || {};
  const v = (s) => (r[s] && r[s].verdict) || "pending";
  return `reviewer-a: ${v("reviewer_a")}, reviewer-b: ${v("reviewer_b")}`;
}

if (!fs.existsSync(statePath)) allow();

let state;
try {
  state = JSON.parse(fs.readFileSync(statePath, "utf8"));
} catch (err) {
  // A loop in progress with a corrupt state file must be repaired, not abandoned.
  const n = readCounter() + 1;
  if (n > DEFAULT_MAX_ITERATIONS) {
    resetCounter();
    allow(`Helix: state.json is unreadable (${err.message}); iteration cap reached, letting Claude stop.`);
  }
  fs.writeFileSync(counterPath, String(n));
  block(`Helix gate: .claude/helix/state.json is not valid JSON (${err.message}). Repair it without changing any verdicts or "passed" values, then continue.`);
}

function writeState() {
  fs.writeFileSync(statePath, JSON.stringify(state, null, 2) + "\n");
}

// Spent budget: halt the run and hand it to the user. Only the user can resume it, by
// raising the spent limit by hand; the guard keeps the agent from doing either.
function halt(result) {
  state.status = "halted";
  state.halt_reason = result.problems.join("; ");
  writeState();
  record(projectDir, { ev: "halt", reason: state.halt_reason });
  allow(
    `Helix HALTED: ${state.halt_reason}. ${summary(result, state)} ${HALT_HELP}`
  );
}

resetCounter(); // the state file parsed; the corrupt-state counter starts over
tick(projectDir, state);

// Only enforce while the loop is actively running. Planning/approval, halted and complete
// states must let Claude hand control back to the user. A paused run may stop, but if its
// budget is spent it is halted first, so it cannot be resumed past its limits.
if (state.status === "paused") {
  const result = evaluate(projectDir, state);
  if (result.problems.length) halt(result);
  allow();
}
if (state.status !== "in_progress") allow();

const checkpoints = Array.isArray(state.checkpoints) ? state.checkpoints : [];

// Derive "passed" for every checkpoint from the two reviewers' verdicts, and persist it.
// Where staleness is enforced, an approval that no longer describes the code goes back to
// "pending". Returns the checkpoints still not passed.
let dirty = false;
function derive(enforceAll) {
  for (const cp of checkpoints) {
    const { approved, problems, fresh } = checkCheckpoint(projectDir, state, cp);
    // Deferred mode leaves an already-passed checkpoint alone until the final gate.
    const enforce = enforceAll || cp.passed !== true;
    if (enforce) {
      for (const name of Object.keys(problems)) {
        Object.assign(cp.reviews[name], {
          verdict: "pending",
          tests_verified: false,
          notes: `${RECHECK} ${problems[name]}`,
        });
        dirty = true;
      }
    }
    const should = enforce ? fresh : approved;
    if (cp.passed !== should) {
      cp.passed = should;
      dirty = true;
      if (should) {
        const r = cp.reviews || {};
        const an = (r.reviewer_a && r.reviewer_a.notes) || "approved";
        const bn = (r.reviewer_b && r.reviewer_b.notes) || "approved";
        cp.review_notes = `Passed: reviewer-a and reviewer-b both approved. a: ${an} | b: ${bn}`;
      } else if (Object.keys(problems).length) {
        cp.review_notes = `Recheck needed: ${Object.values(problems).join("; ")}`;
      }
    }
  }
  return checkpoints.filter((c) => c.passed !== true);
}

const strict = state.recheck === "strict";
let pending = derive(strict);
// Final gate: nothing completes on an approval of code that has since changed.
if (pending.length === 0 && !strict) pending = derive(true);

if (checkpoints.length > 0 && pending.length === 0) {
  state.status = "complete";
  delete state.halt_reason;
  writeState();
  record(projectDir, { ev: "complete" });
  allow("Helix: every checkpoint approved by BOTH reviewers. Workflow complete.");
}

const result = evaluate(projectDir, state);
if (result.problems.length) halt(result);

if (state.halt_reason !== undefined) {
  delete state.halt_reason; // resumed after the user raised the limit
  dirty = true;
}
if (dirty) writeState();

record(projectDir, { ev: "block" });
const n = result.usage.blocks + 1;
const max = result.limits.max_iterations;

// Finish the checkpoint being built before re-reviewing the ones this hook reopened.
const next = pending.find((c) => !isRecheck(c)) || pending[0] || {};
const rechecks = pending.filter(isRecheck).map((c) => c.id);
const lines = [
  `Helix gate failed (iteration ${n}/${max}): ${pending.length} of ${checkpoints.length} checkpoint(s) not passed.`,
  `Next: checkpoint ${next.id} - ${next.title || "(untitled)"} [${reviewerState(next)}].`,
  "A checkpoint passes only when BOTH reviewer-a and reviewer-b approve the same file list and those files are unchanged since.",
  rechecks.length
    ? `Approvals no longer match the code for checkpoint(s) ${rechecks.join(", ")} (see the RECHECK notes in state.json). These are already built: do not regenerate tests or re-implement them. Run the tests, then delegate each to reviewer-a AND reviewer-b for a re-review with the same file list.`
    : "",
  isRecheck(next)
    ? ""
    : "Read .claude/helix/critique-a.md and .claude/helix/critique-b.md for the latest findings, fix every one, re-run the tests, then delegate to reviewer-a AND reviewer-b again.",
  'Do not edit "passed" flags or reviewer verdicts yourself; only the reviewers record verdicts and only this gate derives "passed".',
  'If you are genuinely blocked and need the user, set "status" to "paused" in .claude/helix/state.json and explain why.',
  summary(result, state) + " When any limit is spent the run halts and goes to the user.",
];
block(lines.filter(Boolean).join("\n"));
```

### The approval hash (shared by both hooks and the reviewers)

`.claude/hooks/helix-hash.js`

```js
#!/usr/bin/env node
// Helix Hash: ties a reviewer's approval to the code it approved.
//
// A reviewer records the files it reviewed and a hash of their contents in its slot. The
// guard (on write) and the Stop hook (on every derive) recompute that hash with the same
// function, so an approval stops counting as soon as any of its files changes.
//
// CLI (project root = CLAUDE_PROJECT_DIR or the current directory):
//   node helix-hash.js <file> [<file> ...]   print the "files" and "hash" to record
//   node helix-hash.js --check               report which checkpoints' approvals are stale

const crypto = require("crypto");
const fs = require("fs");
const path = require("path");

const SLOTS = ["reviewer_a", "reviewer_b"];

// Project-relative, forward-slashed (lower-cased on Windows) so both reviewers and both
// hooks agree on what a path is called.
function canon(projectDir, file) {
  const rel = path.relative(projectDir, path.resolve(projectDir, file));
  if (!rel || rel.startsWith("..") || path.isAbsolute(rel)) {
    throw new Error(`${file} is outside the project`);
  }
  const p = rel.split(path.sep).join("/");
  return process.platform === "win32" ? p.toLowerCase() : p;
}

function canonList(projectDir, files) {
  return [...new Set(files.map((f) => canon(projectDir, f)))].sort();
}

// A deleted file hashes as "missing", so removing a reviewed file also changes the hash.
function hashFiles(projectDir, files) {
  const h = crypto.createHash("sha256");
  for (const f of canonList(projectDir, files)) {
    let body;
    try {
      body = crypto.createHash("sha256").update(fs.readFileSync(path.join(projectDir, f))).digest("hex");
    } catch {
      body = "missing";
    }
    h.update(`${f}\0${body}\n`);
  }
  return h.digest("hex").slice(0, 16);
}

// Returns "" if an approved slot still describes the code, else the reason it does not.
function slotProblem(projectDir, state, cp, s) {
  if (state.test_command && s.tests_verified !== true) return "approved without tests_verified";
  if (!Array.isArray(s.files) || s.files.length === 0 || s.files.some((f) => typeof f !== "string")) {
    return "no reviewed files recorded";
  }
  try {
    const files = canonList(projectDir, s.files);
    if (cp.test_target && !files.includes(canon(projectDir, cp.test_target))) {
      return `the reviewed files do not include the test_target (${cp.test_target})`;
    }
    if (s.hash !== hashFiles(projectDir, files)) {
      return `the hash does not match the current contents of: ${files.join(", ")}`;
    }
  } catch (err) {
    return err.message;
  }
  return "";
}

// approved: both verdicts are "approved". fresh: they also still describe the same code.
// problems: per-slot reason an approval no longer counts.
function checkCheckpoint(projectDir, state, cp) {
  const reviews = cp.reviews || {};
  const problems = {};
  let approved = true;
  for (const name of SLOTS) {
    const s = reviews[name] || {};
    if (s.verdict !== "approved") {
      approved = false;
      continue;
    }
    const p = slotProblem(projectDir, state, cp, s);
    if (p) problems[name] = p;
  }
  if (approved && Object.keys(problems).length === 0) {
    const [a, b] = SLOTS.map((n) => canonList(projectDir, reviews[n].files).join("\n"));
    if (a !== b) {
      for (const name of SLOTS) problems[name] = "the two reviewers approved different file lists";
    }
  }
  return { approved, problems, fresh: approved && Object.keys(problems).length === 0 };
}

module.exports = { SLOTS, canonList, hashFiles, slotProblem, checkCheckpoint };

if (require.main === module) {
  const projectDir = process.env.CLAUDE_PROJECT_DIR || process.cwd();
  const args = process.argv.slice(2);

  if (args[0] === "--check") {
    const state = JSON.parse(fs.readFileSync(path.join(projectDir, ".claude", "helix", "state.json"), "utf8"));
    console.log(`recheck mode: ${state.recheck === "strict" ? "strict" : "deferred"}`);
    for (const cp of state.checkpoints || []) {
      const { approved, problems, fresh } = checkCheckpoint(projectDir, state, cp);
      const detail = Object.entries(problems).map(([k, v]) => `${k}: ${v}`).join("; ");
      const label = fresh ? "fresh" : Object.keys(problems).length ? `STALE (${detail})` : approved ? "fresh" : "not approved";
      console.log(`checkpoint ${cp.id}: ${label}`);
    }
  } else if (args.length === 0) {
    console.error("usage: node helix-hash.js <file> [<file> ...] | --check");
    process.exit(1);
  } else {
    const files = canonList(projectDir, args);
    const missing = files.filter((f) => {
      try {
        return !fs.statSync(path.join(projectDir, f)).isFile();
      } catch {
        return true;
      }
    });
    if (missing.length) {
      console.error(`not a file: ${missing.join(", ")}`);
      process.exit(1);
    }
    console.log(JSON.stringify({ files, hash: hashFiles(projectDir, files) }));
  }
}
```

### The budget (shared by both hooks)

`.claude/hooks/helix-budget.js`

```js
#!/usr/bin/env node
// Helix Budget: the limits that stop a run from looping forever.
//
// Every run has a budget in state.json "limits" (defaults below). When any part of it is
// spent, the hooks halt the run (status "halted") and hand it to the user:
//   max_review_rounds          review rounds per checkpoint, re-reviews (RECHECK) included
//   max_total_review_rounds    review rounds for the whole run (null: 3 per checkpoint, min 6)
//   max_active_minutes         active time for the whole run
//   max_checkpoint_minutes     active time on the checkpoint currently being worked on
//   max_diff_lines             diff alarm: lines changed since the run started
//                              (null: 500 per checkpoint, min 1000)
//   max_checkpoint_diff_lines  diff alarm: growth of the diff while one checkpoint is current
// plus the top-level "max_iterations": blocked stops for the whole run.
//
// Usage is read from an append-only ledger, .claude/helix/ledger.jsonl, that only the hooks
// write (the guard denies every tool write under .claude/helix/ that is not a role's own
// file, and every shell command that touches it). It never resets on "paused", so pausing
// to wait for subagents cannot reset any count.
//
// Active time: the hooks record a tick at most once a minute while a run is in_progress or
// paused. A gap of more than 10 minutes between two events is idle time (for example, a
// run waiting on you overnight) and is not counted.
//
// Diff size: lines added + deleted versus the commit the run started on (`git diff
// --numstat`), plus the lines of new untracked files, excluding .claude/. Outside a git
// repository the diff alarm is off.
//
// CLI (project root = CLAUDE_PROJECT_DIR or the current directory):
//   node helix-budget.js        print the run's usage against its limits

const fs = require("fs");
const path = require("path");
const { execFileSync } = require("child_process");

const DEFAULTS = {
  max_review_rounds: 4,
  max_total_review_rounds: null,
  max_active_minutes: 240,
  max_checkpoint_minutes: 60,
  max_diff_lines: null,
  max_checkpoint_diff_lines: 800,
};
const DEFAULT_MAX_ITERATIONS = 25;
const ACTIVE = new Set(["in_progress", "paused"]);
const TICK_MS = 60 * 1000;
const IDLE_GAP_MS = 10 * 60 * 1000;
const EMPTY_TREE = "4b825dc642cb6eb9a060e54bf8d69288fbee4904";
// Prefix the Stop hook puts in a reviewer's notes when it sends an approval back to pending.
const RECHECK = "RECHECK:";

const helixDir = (projectDir) => path.join(projectDir, ".claude", "helix");
const ledgerPath = (projectDir) => path.join(helixDir(projectDir), "ledger.jsonl");

function cps(state) {
  return state && Array.isArray(state.checkpoints) ? state.checkpoints : [];
}

// Returns a problem string if state.limits is malformed, else "".
function badLimits(state) {
  const l = state.limits;
  if (l === undefined) return "";
  if (!l || typeof l !== "object" || Array.isArray(l)) return '"limits" must be an object.';
  for (const [k, v] of Object.entries(l)) {
    if (!Object.prototype.hasOwnProperty.call(DEFAULTS, k)) {
      return `unknown limit "${k}"; the limits are: ${Object.keys(DEFAULTS).join(", ")}.`;
    }
    if (v !== null && !(Number.isInteger(v) && v > 0)) return `limits.${k} must be a positive integer or null.`;
  }
  return "";
}

function limitsFor(state) {
  const n = cps(state).length || 1;
  const l = { ...DEFAULTS };
  for (const [k, v] of Object.entries((state && state.limits) || {})) {
    if (Object.prototype.hasOwnProperty.call(DEFAULTS, k) && v !== undefined) l[k] = v;
  }
  if (l.max_total_review_rounds == null) l.max_total_review_rounds = Math.max(6, 3 * n);
  if (l.max_diff_lines == null) l.max_diff_lines = Math.max(1000, 500 * n);
  if (l.max_review_rounds == null) l.max_review_rounds = DEFAULTS.max_review_rounds;
  if (l.max_active_minutes == null) l.max_active_minutes = DEFAULTS.max_active_minutes;
  if (l.max_checkpoint_minutes == null) l.max_checkpoint_minutes = DEFAULTS.max_checkpoint_minutes;
  if (l.max_checkpoint_diff_lines == null) l.max_checkpoint_diff_lines = DEFAULTS.max_checkpoint_diff_lines;
  l.max_iterations = Number.isInteger(state && state.max_iterations) ? state.max_iterations : DEFAULT_MAX_ITERATIONS;
  return l;
}

function isRecheck(cp) {
  const r = cp.reviews || {};
  return ["reviewer_a", "reviewer_b"].some((s) => String((r[s] && r[s].notes) || "").startsWith(RECHECK));
}

// The checkpoint being worked on: the first unpassed one that is not only waiting for a
// re-review (the orchestrator finishes that one first), else the first unpassed one.
function currentCheckpoint(state) {
  const pending = cps(state).filter((c) => c.passed !== true);
  return pending.find((c) => !isRecheck(c)) || pending[0] || null;
}

// ---- ledger ---------------------------------------------------------------------------

// The events of the current run: everything from the last "start" on.
function readEvents(projectDir) {
  let text;
  try {
    text = fs.readFileSync(ledgerPath(projectDir), "utf8");
  } catch {
    return [];
  }
  const events = [];
  for (const line of text.split("\n")) {
    if (!line.trim()) continue;
    try {
      const e = JSON.parse(line);
      if (e && typeof e.t === "number" && typeof e.ev === "string") events.push(e);
    } catch {}
  }
  let start = -1;
  for (let i = 0; i < events.length; i++) if (events[i].ev === "start") start = i;
  return start < 0 ? [] : events.slice(start);
}

function record(projectDir, ev) {
  fs.mkdirSync(helixDir(projectDir), { recursive: true });
  fs.appendFileSync(ledgerPath(projectDir), JSON.stringify({ t: Date.now(), ...ev }) + "\n");
}

// A new plan starts a new run: keep the previous ledger as ledger.prev.jsonl.
function resetLedger(projectDir) {
  const p = ledgerPath(projectDir);
  if (!fs.existsSync(p)) return;
  fs.renameSync(p, path.join(helixDir(projectDir), "ledger.prev.jsonl"));
}

// ---- diff -----------------------------------------------------------------------------

function git(projectDir, args) {
  return execFileSync("git", args, {
    cwd: projectDir,
    encoding: "utf8",
    stdio: ["ignore", "pipe", "ignore"],
    timeout: 5000,
    maxBuffer: 64 * 1024 * 1024,
  });
}

function headCommit(projectDir) {
  try {
    git(projectDir, ["rev-parse", "--is-inside-work-tree"]);
  } catch {
    return null; // not a git repository: the diff alarm is off
  }
  try {
    return git(projectDir, ["rev-parse", "HEAD"]).trim();
  } catch {
    return EMPTY_TREE; // a repository with no commits yet
  }
}

// Lines changed since `base`, or null if it cannot be measured.
function measureDiff(projectDir, base) {
  if (!base) return null;
  try {
    let lines = 0;
    const spec = ["--", ".", ":(exclude).claude"];
    for (const row of git(projectDir, ["diff", "--numstat", base, ...spec]).split("\n")) {
      const [add, del] = row.split("\t");
      if (/^\d+$/.test(add)) lines += Number(add);
      if (/^\d+$/.test(del)) lines += Number(del);
    }
    const untracked = git(projectDir, ["ls-files", "--others", "--exclude-standard", "-z", ...spec]);
    for (const f of untracked.split("\0").filter(Boolean)) {
      try {
        const buf = fs.readFileSync(path.join(projectDir, f));
        if (buf.length > 2 * 1024 * 1024 || buf.subarray(0, 8000).includes(0)) continue; // binary or huge
        const text = buf.toString("utf8");
        lines += text.split("\n").length - (text.endsWith("\n") ? 1 : 0);
      } catch {}
    }
    return lines;
  } catch {
    return null;
  }
}

// ---- recording usage ------------------------------------------------------------------

// Called by both hooks on every invocation. Starts the run's ledger, notes when the current
// checkpoint changes (with the diff size at that moment, the base for the checkpoint's diff
// alarm), and records a tick at most once a minute. Never throws.
function tick(projectDir, state) {
  try {
    if (!state || !ACTIVE.has(state.status)) return;
    let events = readEvents(projectDir);
    if (events.length === 0) {
      const base = headCommit(projectDir);
      record(projectDir, { ev: "start", base, diff: measureDiff(projectDir, base) });
      events = readEvents(projectDir);
    }
    const cur = currentCheckpoint(state);
    const id = cur ? cur.id : null;
    const lastCp = [...events].reverse().find((e) => e.ev === "cp");
    if (!lastCp || lastCp.cp !== id) {
      record(projectDir, { ev: "cp", cp: id, diff: measureDiff(projectDir, events[0].base) });
      return;
    }
    if (Date.now() - events[events.length - 1].t >= TICK_MS) record(projectDir, { ev: "tick", cp: id });
  } catch {}
}

// Cheap pre-check for the guard, which runs on every tool call: skip the tick if the ledger
// was written less than a minute ago.
function tickDue(projectDir) {
  try {
    return Date.now() - fs.statSync(ledgerPath(projectDir)).mtimeMs >= TICK_MS;
  } catch {
    return true;
  }
}

// ---- evaluating the budget ------------------------------------------------------------

function usage(projectDir, state, { diff = true } = {}) {
  const events = readEvents(projectDir);
  const now = Date.now();
  const u = {
    started: events.length > 0,
    activeMs: 0,
    cpActiveMs: {},
    rounds: {},
    totalRounds: 0,
    blocks: 0,
    diff: null,
    cpDiff: null,
  };
  if (!u.started) return u;

  // Active time: sum the gaps between events (and up to now), skipping idle gaps. Each gap
  // is charged to the checkpoint that was current when it began.
  let cpNow = null;
  const counts = {};
  for (let i = 0; i <= events.length; i++) {
    const t = i < events.length ? events[i].t : now;
    if (i > 0) {
      const gap = t - events[i - 1].t;
      if (gap > 0 && gap <= IDLE_GAP_MS) {
        u.activeMs += gap;
        if (cpNow !== null) u.cpActiveMs[cpNow] = (u.cpActiveMs[cpNow] || 0) + gap;
      }
    }
    if (i === events.length) break;
    const e = events[i];
    if (e.ev === "cp") cpNow = e.cp;
    if (e.ev === "block") u.blocks++;
    if (e.ev === "verdict") {
      counts[e.cp] = counts[e.cp] || {};
      counts[e.cp][e.slot] = (counts[e.cp][e.slot] || 0) + 1;
    }
  }
  // A round is one review by each reviewer; count the busier reviewer.
  for (const [cp, c] of Object.entries(counts)) {
    u.rounds[cp] = Math.max(...Object.values(c));
    u.totalRounds += u.rounds[cp];
  }

  if (diff) {
    const start = events[0];
    const lines = measureDiff(projectDir, start.base);
    if (lines !== null) {
      u.diff = Math.max(0, lines - (start.diff || 0));
      const lastCp = [...events].reverse().find((e) => e.ev === "cp");
      if (lastCp && typeof lastCp.diff === "number") u.cpDiff = Math.max(0, lines - lastCp.diff);
    }
  }
  return u;
}

const minutes = (ms) => Math.floor(ms / 60000);

// Returns { limits, usage, problems }. problems lists every limit that is spent; while the
// run is not complete, any problem halts it.
function evaluate(projectDir, state, opts) {
  const l = limitsFor(state);
  const u = usage(projectDir, state, opts);
  const problems = [];
  if (!u.started) return { limits: l, usage: u, problems };

  for (const cp of cps(state)) {
    const r = u.rounds[cp.id] || 0;
    if (cp.passed !== true && r >= l.max_review_rounds) {
      problems.push(
        `checkpoint ${cp.id} has used ${r} of ${l.max_review_rounds} review rounds (limits.max_review_rounds) without passing`
      );
    }
  }
  if (u.totalRounds >= l.max_total_review_rounds) {
    problems.push(
      `the run has used ${u.totalRounds} of ${l.max_total_review_rounds} review rounds (limits.max_total_review_rounds)`
    );
  }
  if (u.activeMs >= l.max_active_minutes * 60000) {
    problems.push(
      `the run has been active for ${minutes(u.activeMs)} of ${l.max_active_minutes} minutes (limits.max_active_minutes)`
    );
  }
  const cur = currentCheckpoint(state);
  if (cur && (u.cpActiveMs[cur.id] || 0) >= l.max_checkpoint_minutes * 60000) {
    problems.push(
      `checkpoint ${cur.id} has been active for ${minutes(u.cpActiveMs[cur.id])} of ${l.max_checkpoint_minutes} minutes (limits.max_checkpoint_minutes)`
    );
  }
  if (u.diff !== null && u.diff > l.max_diff_lines) {
    problems.push(
      `DIFF ALARM: ${u.diff} lines changed since the run started, over the ${l.max_diff_lines}-line budget (limits.max_diff_lines)`
    );
  }
  if (cur && u.cpDiff !== null && u.cpDiff > l.max_checkpoint_diff_lines) {
    problems.push(
      `DIFF ALARM: the diff grew by ${u.cpDiff} lines while checkpoint ${cur.id} was current, over the ${l.max_checkpoint_diff_lines}-line budget (limits.max_checkpoint_diff_lines)`
    );
  }
  if (u.blocks >= l.max_iterations) {
    problems.push(`the stop-gate has blocked ${u.blocks} of ${l.max_iterations} times (max_iterations)`);
  }
  return { limits: l, usage: u, problems };
}

// One line of usage against limits, for the stop-gate's messages.
function summary({ limits: l, usage: u }, state) {
  const cur = currentCheckpoint(state);
  const parts = [];
  if (cur) {
    parts.push(`checkpoint ${cur.id}: ${u.rounds[cur.id] || 0}/${l.max_review_rounds} rounds, ${minutes(u.cpActiveMs[cur.id] || 0)}/${l.max_checkpoint_minutes} min`);
    if (u.cpDiff !== null) parts.push(`checkpoint diff ${u.cpDiff}/${l.max_checkpoint_diff_lines} lines`);
  }
  parts.push(`run: ${u.totalRounds}/${l.max_total_review_rounds} rounds, ${minutes(u.activeMs)}/${l.max_active_minutes} min, ${u.blocks}/${l.max_iterations} blocked stops`);
  if (u.diff !== null) parts.push(`run diff ${u.diff}/${l.max_diff_lines} lines`);
  return `Budget: ${parts.join("; ")}.`;
}

const HALT_HELP =
  'To continue, raise the limit in the "limits" object of .claude/helix/state.json by hand (or "max_iterations" for blocked stops) and run /orchestrator. ' +
  "Or narrow the plan, or stop here: the agent cannot raise its own limits.";

module.exports = {
  DEFAULTS,
  ACTIVE,
  RECHECK,
  badLimits,
  limitsFor,
  isRecheck,
  currentCheckpoint,
  readEvents,
  record,
  resetLedger,
  tick,
  tickDue,
  usage,
  evaluate,
  summary,
  HALT_HELP,
};

if (require.main === module) {
  const projectDir = process.env.CLAUDE_PROJECT_DIR || process.cwd();
  const state = JSON.parse(fs.readFileSync(path.join(helixDir(projectDir), "state.json"), "utf8"));
  const result = evaluate(projectDir, state);
  console.log(`status: ${state.status}${state.halt_reason ? ` (${state.halt_reason})` : ""}`);
  if (!result.usage.started) {
    console.log("no usage recorded yet (the run has not started)");
  } else {
    console.log(summary(result, state));
    for (const cp of cps(state)) {
      const m = minutes(result.usage.cpActiveMs[cp.id] || 0);
      console.log(`checkpoint ${cp.id}: ${cp.passed ? "passed" : "not passed"}, ${result.usage.rounds[cp.id] || 0} review rounds, ${m} active min`);
    }
  }
  console.log(result.problems.length ? `OVER BUDGET:\n- ${result.problems.join("\n- ")}` : "within budget");
}
```

## Sources & confidence

All capability claims below are **Verified** against the vendors' official documentation,
read directly (not from search summaries), on 2026-09-30.

- Helix (origin of the pattern) — Shopify Engineering's agent-driven migration system: "Helix breaks it into checkpoints of increasing complexity"; "a subagent reads the reference code and generates test cases for each checkpoint" and "the test cases generated for the checkpoint define what 'proven' means"; "Two independent, context-isolated reviewer agents check the new code" and the loop repeats "until both reviewers approve"; a failed gate can be retried but the agent "can't override a failed check": <https://shopify.engineering/helix>
- Claude Code hooks — a `PreToolUse` hook on exit code 2 "Blocks the tool call"; a `Stop` hook on exit code 2 "Prevents Claude from stopping, continues the conversation": <https://code.claude.com/docs/en/hooks>
- Claude Code subagents — "each subagent runs in its own context window"; project `.claude/agents/`, user `~/.claude/agents/`: <https://code.claude.com/docs/en/sub-agents>
- Claude Code skills — a `SKILL.md` creates a matching slash command: <https://code.claude.com/docs/en/slash-commands>

Helix is the name of Shopify Engineering's published agent-driven migration system. What this guide provides is an independent, general-purpose implementation of that pattern on Claude Code primitives — not Shopify's tool itself, and not affiliated with or endorsed by Shopify.
