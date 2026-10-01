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

A checkpoint's `passed` flag is never written by a tool call at all: the stop-gate
**derives** it from the two reviewers' verdicts and flips it to `true` only when _both_
have approved. This mirrors Shopify's design, where "two independent, context-isolated
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
| A blocking pre-tool hook | The `guard`                                                                                              | Confines each role: the builder can't approve or edit findings; each reviewer writes only its own verdict/file; the test-author writes only tests |
| A blocking stop hook     | The `stop-gate`                                                                                          | Forces the loop to continue until every checkpoint is approved by both reviewers                                                                  |

The state of a run lives, per project, in the state file plus one findings file per reviewer:

```
.claude/helix/state.json     # the plan: checkpoints, acceptance criteria, per-reviewer verdicts, passed flags, status
.claude/helix/critique-a.md  # reviewer-a's latest findings or approval
.claude/helix/critique-b.md  # reviewer-b's latest findings or approval
```

No agent writes a checkpoint's `passed` flag. Each reviewer records only its own verdict
in its own slot (`reviews.reviewer_a` / `reviews.reviewer_b`); the stop-gate **derives**
`passed = (reviewer-a approved AND reviewer-b approved)` and persists it. The main agent
may only change the top-level `status` (to `in_progress` or `paused`). The guard hook
enforces exactly that separation of roles.

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
  settings.json                  # registers the two hooks
  helix/state.json               # per-run plan + status (created when a run starts)
  helix/critique-a.md            # reviewer-a's review notes
  helix/critique-b.md            # reviewer-b's review notes
```

Put the skill, agents, hooks, and `settings.json` in your user directory
(`~/.claude/`) to make Helix available in **every** project, or in a project's
`.claude/` to scope it to one repo. The `helix/` state directory stays **per project**
either way — each project runs its own workflow.

### Register the hooks

Claude Code reads hook registrations from `settings.json`. The two Helix hooks attach to
the `PreToolUse` and `Stop` events:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit|Bash|PowerShell",
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
- The `"matcher"` limits the guard to the tools that could touch state: the file editors and the shells.
- The guard `"command"` runs the guard script through Node.
- The `"Stop"` array holds hooks that run **when the agent tries to end its turn**.
- The stop `"command"` runs the stop-gate script.

On Windows, use an absolute path such as
`node "C:\\Users\\you\\.claude\\hooks\\helix-guard.js"` if the tilde is not expanded in
your environment.

### What each hook must do (the contract)

The **stop-gate** (`Stop` event):

- If the run's `status` is not `in_progress`, allow the stop (planning, paused, and complete states hand control back to the user).
- **Derive** each checkpoint's `passed` from its two verdicts (`passed = reviewer-a approved AND reviewer-b approved`) and persist it. This — not any tool call — is what flips `passed`. (The hook writes state via `fs`, which does not pass through the guard.)
- If every checkpoint is now `passed`, set `status` to `complete` and allow the stop.
- Otherwise **block** the stop (exit code 2) and print, to standard error, which checkpoint is next, which reviewers are still pending/rejected, and to read both critique files. Claude Code feeds that text back to the agent as the reason it cannot stop.
- Keep an iteration counter and a cap, so a stuck run pauses instead of looping forever.

The **guard** (`PreToolUse` event) — enforced per role (the subagent's name; "main" is the builder):

- **planner**: may write the whole plan to `state.json` (only while `status` is `awaiting_approval`, with every checkpoint unapproved and both verdicts `pending`) and create the empty `critique-a.md` / `critique-b.md`. No source files.
- **reviewer-a / reviewer-b**: each may change _only_ its own verdict slot (`reviews.reviewer_a` or `reviews.reviewer_b`) in `state.json`, and write _only_ its own findings file (`critique-a.md` or `critique-b.md`). Any attempt to touch `passed`, `status`, the other reviewer's slot, another checkpoint, or a source file is denied. This keeps the two reviews **independent** and un-forgeable.
- **test-author**: may write _only_ test files (a file in a `tests/` directory, or `test_*.py` / `*_test.py` / `*.spec.*` / `conftest.py`). Every write under `.claude/helix/` is denied, so it can never approve anything or touch state.
- **main (the builder)**: during the planning phase (`status` is `awaiting_approval`, or before a state file exists) it may edit the plan freely _as long as_ nothing starts out approved (all `passed` false, `review_notes` empty, both verdicts `pending`). Once `status` is `in_progress` the plan is frozen: the only change allowed to `state.json` is the top-level `status` (to `in_progress` or `paused`). Any Write/Edit to a `critique-*.md`, or any other write under `.claude/helix/`, is denied — this is the rule that stops the builder from approving its own work or rewriting a review.
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

- **`planner`** — turns the request into checkpoints (each with a `test_target` and two
  `pending` reviewer slots).
- **`test-author`** — for a checkpoint, generates tests from its acceptance criteria into
  `test_target`, _before_ implementation. Restricted so it can write only test files.
- **`reviewer-a`** and **`reviewer-b`** — two independent reviewers with the same restricted
  toolset. Each runs in its own fresh context, judges the code against the acceptance
  criteria, runs the checkpoint's tests, and records **only its own** verdict and findings
  file. Neither is ever shown the other's notes — that independence is the whole point of
  running two.

`reviewer-b` is identical to `reviewer-a` except that it writes `reviews.reviewer_b` and
`critique-b.md`. A checkpoint passes only when both have approved; the stop-gate derives
that. The full system prompt of every subagent is in [Complete files](#complete-files).

### The orchestrator skill

A skill is a `SKILL.md` file that becomes a slash command; a skill at
`.claude/skills/orchestrator/SKILL.md` creates `/orchestrator`. [VERIFIED] Its body is
the workflow: delegate to the planner, present the plan for approval, then loop over
checkpoints — for the first unapproved one, delegate to the **test-author** to generate
its tests, implement it, run the tests yourself, then delegate to **reviewer-a and
reviewer-b independently and in parallel**. The checkpoint advances only when the tests
pass _and_ both reviewers approve; on any rejection the builder fixes the findings and
re-reviews. Repeat until every checkpoint is approved by both reviewers. The full skill is
in [Complete files](#complete-files).

### Run it

```
/orchestrator build an interactive tutor that teaches AI agent evaluations
```

Approve the plan, then let the loop run. The stop-gate keeps the agent working until
both reviewers have approved every checkpoint.

## Complete files

These are the exact files of the working implementation. Save each one at the path shown,
under `~/.claude/` (every project) or a project's `.claude/` (one project), then register
the hooks as shown in [Register the hooks](#register-the-hooks).

### The `/orchestrator` skill

`.claude/skills/orchestrator/SKILL.md`

````markdown
---
name: orchestrator
description: Run the Helix plan/test/implement/review loop for a feature. Plans checkpoints, waits for user approval, then for each checkpoint generates tests, implements, and gets it independently approved by two reviewers before moving on.
argument-hint: <feature request> | approve | (empty to resume)
disable-model-invocation: true
---

# Helix Orchestrator

You are the master coordinator of the Helix workflow. State lives in `.claude/helix/state.json`; the two reviewers' findings live in `.claude/helix/critique-a.md` and `.claude/helix/critique-b.md`. A Stop hook blocks you from ending your turn while `status` is `"in_progress"` and any checkpoint is unpassed. A checkpoint's `passed` is derived by that hook and becomes true only when **both** independent reviewers approve it — you cannot set it yourself.

Arguments: $ARGUMENTS

## Decide what to do

- **Arguments are a feature request** → Phase 1.
- **Arguments are `approve`**, or the user approves the plan in conversation → Phase 2.
- **No arguments** → read the state file and resume: `awaiting_approval` → show the plan and ask for approval; `paused` or `in_progress` → Phase 2; `complete` or missing → ask the user for a feature request.

## Phase 1: Plan

1. Delegate to the `planner` subagent with the full feature request.
2. Read `.claude/helix/state.json` and present the checkpoints to the user as a numbered list with their acceptance criteria.
3. Ask the user to review, edit the file if they wish, and reply "approve". Then end your turn. (The hook allows this because status is `awaiting_approval`.)

## Phase 2: Implement loop

1. Set `"status": "in_progress"` in the state file. From now on you cannot stop until every checkpoint passes.
2. Take the first checkpoint with `"passed": false`. Read `.claude/helix/critique-a.md` **and** `.claude/helix/critique-b.md`; if either has findings for this checkpoint, address every one of them.
3. **Generate tests first.** Set `"status": "paused"`, delegate to the `test-author` subagent with the checkpoint id, and set `"status": "in_progress"` again when it returns. It writes tests derived from the acceptance criteria to the checkpoint's `test_target`. (On a re-attempt where tests already exist and still fit, you may skip this, but never delete or weaken those tests yourself.)
4. **Implement** the checkpoint so those tests — and every acceptance criterion — are satisfied.
5. **Run the tests yourself.** Run `test_command` and any commands named in the acceptance criteria. Do not ask for review until they pass; the reviewers will reject a red or missing test.
6. **Review independently, in parallel.** Set `"status": "paused"` first (see "Waiting on subagents" below). Then, in a single message, delegate to **both** `reviewer-a` and `reviewer-b`, giving each only the checkpoint id and the list of files you changed. Never pass one reviewer the other's notes, verdict, or critique file — their independence is the point. When both have returned, set `"status": "in_progress"` again before doing anything else.
7. Read both verdicts from `state.json` (`reviews.reviewer_a` and `reviews.reviewer_b`) and both critique files. The checkpoint advances **only when the tests pass AND both reviewers have `"verdict": "approved"`.** If either reviewer rejected, go back to step 2 for the **same** checkpoint: fix the findings, re-run tests, and re-delegate to both reviewers. One approval is never enough.
8. Move to the next unpassed checkpoint and repeat. When every checkpoint is passed (both reviewers approved each), the Stop hook marks the run complete; give the user a short summary of what was built and the reviewers' notes.

## Rules

- Never edit a `passed` flag, a `review_notes` field, or any `reviews.*` verdict yourself. Only the reviewers record verdicts, and only the Stop hook derives `passed` (both-approved). A PreToolUse hook enforces this: from the main session you may only change the top-level `status` (to `in_progress` or `paused`) with the Edit tool, you may not write any `critique-*.md`, and shell commands may not touch `.claude/helix/`. Read state with the Read tool.
- Never weaken acceptance criteria, delete/skip generated tests, or edit a reviewer's findings to get an approval.
- If you are truly blocked (missing credentials, an ambiguous requirement, a decision only the user can make), set `"status": "paused"`, explain exactly what you need, and end your turn. Do not use this to escape hard work.

## Waiting on subagents

The Stop hook counts every blocked stop toward `max_iterations`. If the run is `in_progress` while a subagent works and you end your turn to wait for it, each wait burns an iteration, and the cap can run out before the work is done. So:

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
3. Give every checkpoint concrete, verifiable acceptance criteria (a command that passes, a behavior that can be observed, a file that exists). Avoid vague criteria like "code is clean". These criteria are what the test-author turns into tests and what both reviewers check against, so make them testable.
4. Choose a `test_target` for each checkpoint: the path where the test-author will write that checkpoint's tests, following the project's convention (for a Python/pytest project, e.g. `tests/test_<slug>.py`).
5. Write the plan to `.claude/helix/state.json` (create the directory if needed), overwriting any previous plan, using exactly this shape:

```json
{
  "feature": "<the original request>",
  "status": "awaiting_approval",
  "max_iterations": 25,
  "test_command": "<command that runs the project's tests, or null>",
  "checkpoints": [
    {
      "id": 1,
      "title": "<short name>",
      "description": "<what to build and where>",
      "acceptance_criteria": ["<verifiable criterion>", "..."],
      "test_target": "tests/test_<slug>.py",
      "reviews": {
        "reviewer_a": { "verdict": "pending", "tests_verified": false, "notes": "" },
        "reviewer_b": { "verdict": "pending", "tests_verified": false, "notes": "" }
      },
      "passed": false,
      "review_notes": ""
    }
  ]
}
```

6. Also create an empty `.claude/helix/critique-a.md` and `.claude/helix/critique-b.md` (one per reviewer).

## Rules

- `status` must be `"awaiting_approval"`. Never set it to anything else; the orchestrator does that after the user approves.
- Every checkpoint starts with `"passed": false`, empty `review_notes`, and both reviewer verdicts `"pending"`. The Helix guard will reject a plan where anything starts out approved.
- Do not modify any file outside `.claude/helix/`.
- Finish by returning a short numbered summary of the checkpoints.
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
  "reviewer_a": { "verdict": "approved", "tests_verified": true, "notes": "<one line>" }
}
```

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
  "reviewer_b": { "verdict": "approved", "tests_verified": true, "notes": "<one line>" }
}
```

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
//                  critique-a.md. Never passed/status/other slots/source.
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

const fs = require("fs");
const path = require("path");

const REVIEWERS = {
  "reviewer-a": { slot: "reviewer_a", file: "critique-a.md" },
  "reviewer-b": { slot: "reviewer_b", file: "critique-b.md" },
};
const MAIN_STATUSES = new Set(["awaiting_approval", "in_progress", "paused"]);
const VERDICTS = new Set(["pending", "approved", "rejected"]);
const SLOT_KEYS = new Set(["verdict", "tests_verified", "notes"]);

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
  const bad = notAllUnapproved(after);
  if (bad) deny(bad);
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

if (!MAIN_STATUSES.has(after.status)) {
  deny(
    `status may only be set to ${[...MAIN_STATUSES].join(", ")}. The Stop hook marks the workflow complete.`
  );
}

if (!before || before.status === "awaiting_approval") {
  // Planning phase: the builder may edit the plan and flip status to in_progress, as long
  // as nothing starts out approved.
  const bad = notAllUnapproved(after);
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
      return `reviews.${slot} may contain only verdict, tests_verified, and notes.`;
    }
    if (!VERDICTS.has(s.verdict)) {
      return `reviews.${slot}.verdict must be one of: ${[...VERDICTS].join(", ")}.`;
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
// when BOTH reviewer-a and reviewer-b have approved it. This hook writes state.json via
// fs (it does not go through the PreToolUse guard), so it is the single authority that
// flips "passed" and marks the run complete.
//
// Exit 0 -> Claude may stop.
// Exit 2 -> Claude is blocked; stderr is fed back to it as the reason.

const fs = require("fs");
const path = require("path");

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

// A checkpoint passes only when BOTH reviewers have approved it.
function bothApproved(cp) {
  const r = cp.reviews || {};
  const a = r.reviewer_a || {};
  const b = r.reviewer_b || {};
  return a.verdict === "approved" && b.verdict === "approved";
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

// Only enforce while the loop is actively running. Planning/approval and paused/complete
// states must let Claude hand control back to the user.
if (state.status !== "in_progress") {
  resetCounter();
  allow();
}

const checkpoints = Array.isArray(state.checkpoints) ? state.checkpoints : [];

// Derive "passed" for every checkpoint from the two reviewers' verdicts, and persist it.
let dirty = false;
for (const cp of checkpoints) {
  const should = bothApproved(cp);
  if (cp.passed !== should) {
    cp.passed = should;
    dirty = true;
    if (should) {
      const r = cp.reviews || {};
      const an = (r.reviewer_a && r.reviewer_a.notes) || "approved";
      const bn = (r.reviewer_b && r.reviewer_b.notes) || "approved";
      cp.review_notes = `Passed: reviewer-a and reviewer-b both approved. a: ${an} | b: ${bn}`;
    }
  }
}

const pending = checkpoints.filter((c) => c.passed !== true);

if (checkpoints.length > 0 && pending.length === 0) {
  state.status = "complete";
  fs.writeFileSync(statePath, JSON.stringify(state, null, 2) + "\n");
  resetCounter();
  allow("Helix: every checkpoint approved by BOTH reviewers. Workflow complete.");
}

if (dirty) fs.writeFileSync(statePath, JSON.stringify(state, null, 2) + "\n");

const max = Number.isInteger(state.max_iterations) ? state.max_iterations : DEFAULT_MAX_ITERATIONS;
const n = readCounter() + 1;
if (n > max) {
  state.status = "paused";
  fs.writeFileSync(statePath, JSON.stringify(state, null, 2) + "\n");
  resetCounter();
  allow(`Helix: hit the ${max}-iteration cap with ${pending.length} checkpoint(s) not passed. Loop paused; run /orchestrator to resume.`);
}
fs.writeFileSync(counterPath, String(n));

const next = pending[0] || {};
const lines = [
  `Helix gate failed (iteration ${n}/${max}): ${pending.length} of ${checkpoints.length} checkpoint(s) not passed.`,
  `Next: checkpoint ${next.id} - ${next.title || "(untitled)"} [${reviewerState(next)}].`,
  "A checkpoint passes only when BOTH reviewer-a and reviewer-b approve it.",
  "Read .claude/helix/critique-a.md and .claude/helix/critique-b.md for the latest findings, fix every one, re-run the tests, then delegate to reviewer-a AND reviewer-b again.",
  'Do not edit "passed" flags or reviewer verdicts yourself; only the reviewers record verdicts and only this gate derives "passed".',
  'If you are genuinely blocked and need the user, set "status" to "paused" in .claude/helix/state.json and explain why.',
];
block(lines.join("\n"));
```

## Sources & confidence

All capability claims below are **Verified** against the vendors' official documentation,
read directly (not from search summaries), on 2026-09-30.

- Helix (origin of the pattern) — Shopify Engineering's agent-driven migration system: "Helix breaks it into checkpoints of increasing complexity"; "a subagent reads the reference code and generates test cases for each checkpoint" and "the test cases generated for the checkpoint define what 'proven' means"; "Two independent, context-isolated reviewer agents check the new code" and the loop repeats "until both reviewers approve"; a failed gate can be retried but the agent "can't override a failed check": <https://shopify.engineering/helix>
- Claude Code hooks — a `PreToolUse` hook on exit code 2 "Blocks the tool call"; a `Stop` hook on exit code 2 "Prevents Claude from stopping, continues the conversation": <https://code.claude.com/docs/en/hooks>
- Claude Code subagents — "each subagent runs in its own context window"; project `.claude/agents/`, user `~/.claude/agents/`: <https://code.claude.com/docs/en/sub-agents>
- Claude Code skills — a `SKILL.md` creates a matching slash command: <https://code.claude.com/docs/en/slash-commands>

Helix is the name of Shopify Engineering's published agent-driven migration system. What this guide provides is an independent, general-purpose implementation of that pattern on Claude Code primitives — not Shopify's tool itself, and not affiliated with or endorsed by Shopify.
