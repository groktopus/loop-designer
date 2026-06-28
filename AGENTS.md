# loop-designer

Autonomous agent loop design in 10 steps. See [SKILL.md](SKILL.md) for full methodology.

This skill distills Alex (@de1lymoon)'s 10-step roadmap into concrete Hermes Agent invocations — from `/goal` with `goal_judge` to unattended cron loops with memory, verification, and fail-safe guardrails.

## Quick Reference

| Step | Article Move | Hermes Mechanism |
|------|-------------|------------------|
| 1 | Agent is a loop | /goal loops via goal_judge |
| 2 | Harness first | Dedicated profile with tool-config pruning |
| 3 | System improves, not model | Memory + skills compound across runs |
| 4 | Independent grader | goal_judge + kanban tasks with acceptance criteria |
| 5 | Maker/checker split | delegate_task verifier subagent |
| 6 | Timer, then cloud | profile-created cronjob seeds kanban on 60s tick |
| 7 | Compose with workflows | delegate_task batch + kanban deps |
| 8 | Memory | State log + durable memory + skills |
| 9 | Distill into skills | skill_manage patch-on-fail lifecycle |
| 10 | Fail safe | Restricted profile + enabled_toolsets + repeat cap |

## Prerequisites

- Hermes Agent with `/goal` and `/subgoal` commands
- `goal_judge` auxiliary model configured
- Kanban dispatcher, cronjob, skill_manage, and memory tools
- A dedicated Hermes profile for running the loop

## Loading

```bash
skill_view(name="loop-designer")
```
