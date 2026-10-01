# Helix in Claude Code

This is an independent implementation of Shopify's Helix pattern in Claude Code: the agent that writes the code never gets the final word. Two independent reviewer agents must both approve every slice of work, and hooks enforce it.

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


