# Helix in Claude Code

An independent implementation of Shopify's Helix pattern in Claude Code: the agent that writes the code never gets the final word. Two independent reviewer agents must both approve every slice of work, and hooks enforce it.

> Helix is [Shopify Engineering's agent-driven migration system](https://shopify.engineering/helix). This repository describes my own independent implementation of the pattern in Claude Code. It is not Shopify's tool and is not affiliated with or endorsed by Shopify.

**Setup:** see [HELIX_SETUP.md](HELIX_SETUP.md).

## Why

When an AI coding agent finishes a task, the agent that wrote the code is usually the one that decides it's done. For quick experiments that's ok, and I work that way all the time.

I also wanted to see what changes when the builder doesn't get the final word.

## Where the idea comes from

Shopify Engineering built Helix so AI agents could migrate their mobile app from React Native to native Swift and Kotlin. The work is split into checkpoints. Two reviewer agents, each working in isolation, must both approve a checkpoint before the next one starts, and the agent can't override a failed check.

This version is an independent implementation for everyday coding.

## The moving parts

- **The command.** /orchestrator runs the loop: plan, write tests, build, review, repeat.
- **Four helper agents.** A planner, a test writer and two reviewers. Each starts fresh with its own context, and the reviewers never see each other's notes.
- **The guard** (a PreToolUse hook). It checks every file change before it happens. The builder can't write a verdict. Each reviewer can write only its own. The test writer can write only tests.
- **The gate** (a Stop hook). It stops the builder from ending its turn while any checkpoint is unapproved. It's also the only thing that can mark a checkpoint "passed", and it does so only when both reviewers have said yes.

Full details, including the hook contracts and the settings.json registration, are in [HELIX_SETUP.md](HELIX_SETUP.md).

## Case study: a browser-based tutor

I asked for a browser-based tutor to teach a Python beginner (me) how AI agents are evaluated. The planner produced 8 checkpoints (slices of work) with 64 acceptance criteria. The finished app came to about 3,700 lines of Python and 850 lines of HTML, CSS and JavaScript, with 882 automated tests.

The reviewers rejected work 9 times, across 5 of the 8 checkpoints. Each time, the full test suite was passing.

Two examples of rejected work:

- **A demo that could freeze the server.** The tutor has a demo where a learner types in a text-matching pattern (a regular expression). One of the reviewers found that one well-known pattern made the server's work grow explosively: every 2 extra characters roughly quadrupled the time, and one probe froze it for 88 seconds. The fix runs each pattern in a separate process with a time limit.
- **An exercise runner that took three rounds.** The exercise runner runs the learner's own code. In round one, a superscript "²" slipped past a number check and crashed it, a flood of printed output pushed memory to 3.8 GB in one probe, and a background process could outlive the time limit. In round two, a function returning 50 million characters produced a 50 MB web response. In round three, both reviewers approved.

Every fix came with a new test for that exact failure. That's much of how the suite grew from 67 tests to 882.

## What broke in the harness

- **The gate ran out of patience.** It allows 25 attempts, so a stuck run can't loop forever. The builder was using them up while it waited for reviewers. The fix was to mark the run "paused" while reviewers work, then resume.
- **A reviewer contradicted itself.** It wrote approval notes but left its verdict saying "rejected". Because "passed" is calculated from the verdicts, the checkpoint didn't pass, and the mismatch was easy to spot.
- **A reviewer got locked out.** A permission check once stopped a reviewer from recording its own approval. The builder isn't allowed to write verdicts, so it didn't. It ran a fresh review instead.

## Lessons for building your own

- **Enforce the rules in the tooling, not the prompt.** An agent can skip an instruction to "review carefully". It can't skip a blocked file write.
- **Calculate "done" from the reviewers' verdicts**, and let nobody set it directly, the builder included.
- **Keep the reviewers apart.** On checkpoint 1, one reviewer approved while the other found a test file that ran on Python 3.12 but broke on 3.10, the version the README promised to support. On checkpoint 3, the roles flipped. Either reviewer working alone would have let one of those through.
- **Budget for it.** It takes more time and more tokens.

I still let agents run freely when I'm exploring. Helix is what I add when I get serious about a project.

---

All figures come from a single run on one project.
