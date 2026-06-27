# loop-designer

Autonomous agent loop design in 10 steps.

[![Version](https://img.shields.io/badge/version-0.5.1-blue)](SKILL.md)

A [Hermes Agent](https://github.com/nousresearch/hermes-agent) skill that distills [Alex (@de1lymoon)'s 10-step roadmap](https://x.com/de1lymoon/status/2069726411724673077) into concrete Hermes Agent invocations — from `/goal` with `goal_judge` to unattended cron loops with memory, verification, and fail-safe guardrails.

This is NOT about better prompting or model selection. The model stays the same — what improves is the infrastructure wrapped around it.

## What it covers

| Step | Concept | Mechanism |
|------|---------|-----------|
| 1 | Agent is a loop | `/goal` loops via `goal_judge` |
| 2 | Harness first | Dedicated profile with tool-config pruning |
| 3 | System improves, not model | Memory + skills compound across runs |
| 4 | Independent grader | `goal_judge` + kanban decomposition |
| 5 | Maker/checker split | `delegate_task` verifier subagent |
| 6 | Timer, then cloud | `cronjob` seeds kanban on 60s tick |
| 7 | Compose with workflows | `delegate_task` batch + kanban deps |
| 8 | Memory | Memory provider + state log + memory tool |
| 9 | Distill into skills | `skill_manage` patch-on-fail lifecycle |
| 10 | Fail safe | Restricted profile + `enabled_toolsets` + repeat cap |

## Prerequisites

- Hermes Agent with `/goal` and `/subgoal` commands
- `goal_judge` auxiliary model configured
- Kanban dispatcher, cronjob, skill_manage, and memory tools
- A dedicated Hermes profile for running the loop

## Installation

Clone the repo into your Hermes skills directory:

```bash
git clone https://github.com/groktopus/loop-designer.git ~/.hermes/skills/thinking/loop-designer
```

Verify it's installed:

```bash
hermes skills list | grep loop-designer
```

## Quick start

Once installed, load the skill with the slash command:

```
/loop-designer
```

Or reference it naturally in conversation — Hermes discovers and loads skills automatically when their triggers match.

Then work through the 10 steps sequentially — each step in [SKILL.md](SKILL.md) translates one move from the article into a concrete Hermes configuration or tool invocation. The canonical output is a profile-bound cron job with `goal_judge` as the grader and a skill-update lifecycle.

## Documentation

The full methodology — all 10 steps with configuration examples and pitfalls — lives in [SKILL.md](SKILL.md).

## License

MIT © 2026 groktopus — see [LICENSE](LICENSE).
