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

Give each reviewer a restricted toolset so it can read, run tests, and record its verdict,
but is clearly a separate role. Minimal frontmatter:

```markdown
---
name: reviewer-a
description: Independent reviewer A. Runs the checkpoint's tests, then records only its own verdict in reviews.reviewer_a and writes critique-a.md. Never sees reviewer-b's review.
tools: Read, Grep, Glob, Bash, Write, Edit
---

(system prompt: how to review a checkpoint against its acceptance criteria, how to run the
generated tests, and how to record its verdict in its own state.json slot and findings in
critique-a.md — never touching passed, status, or the other reviewer's slot)
```

`reviewer-b` is identical except it writes `reviews.reviewer_b` and `critique-b.md`. A
checkpoint passes only when both have approved; the stop-gate derives that.

### The orchestrator skill

A skill is a `SKILL.md` file that becomes a slash command; a skill at
`.claude/skills/orchestrator/SKILL.md` creates `/orchestrator`. [VERIFIED] Its body is
the workflow: delegate to the planner, present the plan for approval, then loop over
checkpoints — for the first unapproved one, delegate to the **test-author** to generate
its tests, implement it, run the tests yourself, then delegate to **reviewer-a and
reviewer-b independently and in parallel**. The checkpoint advances only when the tests
pass _and_ both reviewers approve; on any rejection the builder fixes the findings and
re-reviews. Repeat until every checkpoint is approved by both reviewers.

### Run it

```
/orchestrator build an interactive tutor that teaches AI agent evaluations
```

Approve the plan, then let the loop run. The stop-gate keeps the agent working until
both reviewers have approved every checkpoint.

## Sources & confidence

All capability claims below are **Verified** against the vendors' official documentation,
read directly (not from search summaries), on 2026-09-30.

- Helix (origin of the pattern) — Shopify Engineering's agent-driven migration system: "Helix breaks it into checkpoints of increasing complexity"; "a subagent reads the reference code and generates test cases for each checkpoint" and "the test cases generated for the checkpoint define what 'proven' means"; "Two independent, context-isolated reviewer agents check the new code" and the loop repeats "until both reviewers approve"; a failed gate can be retried but the agent "can't override a failed check": <https://shopify.engineering/helix>
- Claude Code hooks — a `PreToolUse` hook on exit code 2 "Blocks the tool call"; a `Stop` hook on exit code 2 "Prevents Claude from stopping, continues the conversation": <https://code.claude.com/docs/en/hooks>
- Claude Code subagents — "each subagent runs in its own context window"; project `.claude/agents/`, user `~/.claude/agents/`: <https://code.claude.com/docs/en/sub-agents>
- Claude Code skills — a `SKILL.md` creates a matching slash command: <https://code.claude.com/docs/en/slash-commands>

Helix is the name of Shopify Engineering's published agent-driven migration system. What this guide provides is an independent, general-purpose implementation of that pattern on Claude Code primitives — not Shopify's tool itself, and not affiliated with or endorsed by Shopify.
