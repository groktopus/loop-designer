---
name: loop-designer
description: Autonomous agent loop design in 10 steps.
version: 0.5.2
metadata:
  hermes:
    tags: [Loop Design, Goal Judge, Profiles, Cron, Autonomous Agents]
---

# From Prompter to Loop Designer

A loop designer builds a system that prompts itself -- runs on a timer, checks its own work against a goal, spawns helpers when needed, and writes down what it learned so the next run starts smarter. This skill distills Alex (@de1lymoon)'s [10-step roadmap](https://x.com/de1lymoon/status/2069726411724673077) into concrete Hermes Agent invocations.

This is NOT about better prompting. It does NOT cover model selection, fine-tuning, or prompt engineering. The model stays the same throughout -- what improves is the Hermes infrastructure wrapped around it.

## When to Use

- You are building an agent that should run unattended on a schedule
- You have a working manual workflow and want to automate it
- You want the system to remember past runs and improve over time
- You need to build a fail-safe autonomous loop
- You are debugging why your automated agent stops improving after initial gains

## When Not to Use

- **The task takes under 5 minutes and runs less than weekly.** The setup cost of a dedicated profile, goal_judge wiring, state log, and cron job exceeds the manual effort for months or years. A simple cron job or calendar reminder is a better fit.
- **The task requires irreversible production writes without a human gate.** Deployments, database migrations, and user-facing content changes must have a human approval step. The loop can prepare and verify, but should not execute alone.
- **The task's environment changes faster than the loop's value compounds.** If the target (dashboard, API, website) changes layout or contract weekly, the loop spends more time debugging failures than the manual task would take. The compounding benefit of memory and skills only helps when the environment is stable enough for lessons to accumulate.
- **You can't define a clear "done" condition.** Without a goal_judge verdict or measurable acceptance criteria, the loop has no way to know when to stop. Loops without a stop condition drift into token-wasting oscillation.
- **You're building it to learn loop design, not to solve a problem.** That's valid — build a throwaway loop on a synthetic task first. The Verification section spells out what graduating looks like.

## Prerequisites

- Hermes Agent with `/goal` and `/subgoal` commands
- `goal_judge` auxiliary model configured in `~/.hermes/config.yaml` -- this IS the independent grader the article calls for
- Kanban dispatcher (runs on 60s tick) for autonomous task decomposition and promotion
- `cronjob` tool for scheduling recurring loops
- `skill_manage` tool for distilling learned patterns into skills
- `memory` tool for cross-session persistence
- A dedicated Hermes profile for running the loop (see Step 2)

## How to Run

Work through the tiers sequentially. Each step in the Procedure translates one of the article's 10 moves into a concrete Hermes configuration or tool invocation. The canonical output is a profile-bound cron job with goal_judge as the grader and a skill-update lifecycle.

## Quick Reference

| Step | Article Move | Hermes Mechanism |
|------|-------------|------------------|
| 1 | Agent is a loop | /goal loops via goal_judge, which IS the independent grader |
| 2 | Harness first | Dedicated profile with profile.yaml + tool-config pruning |
| 3 | System improves, not model | Memory provider auto-extracts; skills compound across runs |
| 4 | Independent grader | goal_judge + kanban dispatcher decomposes goal into cards with per-card acceptance criteria |
| 5 | Maker/checker split | delegate_task with verifier subagent + distinct profile |
| 6 | Timer, then cloud | cronjob seeds kanban; dispatcher eats through cards on 60s tick |
| 7 | Compose with workflows | delegate_task batch + kanban dependency graph (parallel by card dependency) |
| 8 | Memory | Memory provider auto-extracts; memory tool for manual facts |
| 9 | Distill into skills | skill_manage patch-on-fail lifecycle |
| 10 | Fail safe | Restricted profile + enabled_toolsets + per-model routing |

## Procedure

### Tier 1 -- See the loop

**1. See the agent as a loop.**

Hermes already has loop infrastructure. `/goal` sets an objective. `goal_judge` evaluates completion after every turn. Together they form the core loop: work until the judge says done.

A /goal runs as a persistent loop with automatic pass/fail checks. This IS the article's "while not goal_met" -- no extra infrastructure needed.

```
/goal All tests pass and lint is clean.
```

Add subgoals for multi-condition loops with `/subgoal`.

**2. Lock down the harness first -- as a profile.**

The harness is the environment the agent boots into. In Hermes, that is a **profile**: a dedicated directory at `~/.hermes/profiles/<name>/` with:

Build one before the loop touches any real work. Create it with:

```
hermes profile create <name>
```

Replace `<name>` with a short identifier for the loop's purpose (e.g., `nightly-build`, `scraper-loop`, `code-review`). This creates the profile directory at `~/.hermes/profiles/<name>/` with a starter `profile.yaml`.

Then populate it with the loop's specific configuration:

```
profile.yaml          # description, model override, allowed toolsets
.env                  # credentials scoped to this loop's tasks
skills/               # symlinks to only the skills the loop needs
SOUL.md               # identity/persona for the loop's run context
```

Hardware the harness by disabling tools the loop should never use. In `profile.yaml`:

```yaml
disabled_toolsets:
  - homeassistant
  - spotify
  - image_gen
```

The test for a good harness: run one manual cycle under the profile and verify every tool call the loop will need works on the first try.

**3. Understand what self-improving actually means.**

The model's weights do not change between runs. In Hermes, what compounds is:

- **Memory provider**: auto-extracts facts and relations from every turn across sessions. The loop inherits all prior extracted knowledge without manual state management.
- **Skills**: versioned via skill_manage, patched on failure. Every run starts with the hardened version.
- **Memory tool**: manual curated facts via the `memory` tool -- injected every turn.

The honest framing: you are not waiting for the model to get smarter. You are building a Hermes environment that gets smarter, run over run, with the same model.

### Tier 2 -- Build the loop

**4. Set a goal with an independent grader.**

The loop needs a stop condition the agent does not control. Hermes provides this natively with the `goal_judge` auxiliary model -- a separate model (configured in `config.yaml` under `auxiliary.goal_judge`) that:

- Runs in its own context window
- Sees the cumulative conversation and the original goal
- Evaluates whether the goal is met or not
- Has no stake in the work produced

Configure it:

```yaml
auxiliary:
  goal_judge:
    provider: openrouter
    model: qwen/qwen3-8b
```

Then invoke:

```
/goal All tests pass and lint is clean.
Triage failures, draft fixes, repeat until the goal holds.
```

The `goal_judge` fires after every turn. When it says done, the loop stops. This IS the article's "independent grader" -- no separate subagent needed for single-task loops.

**Physical circuit breaker: repeat cap.** goal_judge provides the logical stop. For unattended loops, add a physical one: the `repeat` parameter on the cronjob that seeds the loop. This limits how many times the loop fires regardless of goal_judge's verdict -- the hard stop that prevents a runaway loop when the grader itself is wrong, broken, or unreachable.

```
cronjob(repeat=10)  # hard stop after 10 iterations no matter what goal_judge says
```

The article's mistake is "no stop condition." goal_judge is one stop. `repeat` is the other. Both are needed because goal_judge is a model and can be wrong. The repeat cap is the physical guarantee.

**Kanban as the goal decomposition engine.** A monolithic goal works for simple pass/fail loops. For complex goals that decompose into parallel work items, seed the kanban board instead. The kanban dispatcher (60s tick) breaks the goal into cards with per-card acceptance criteria, promotes them through ready/in-progress/done lanes autonomously, and the kanban worker agents execute the cards as they become ready. The dispatcher handles fan-out by card dependency -- cards that are independent run in parallel, cards that depend on each other wait for their parent. This IS the article's "system that operates itself" at the task level, not just the loop level.

**5. Split the maker from the checker.**

For high-stakes loops where the goal_judge's binary pass/fail is too coarse, add a verifier subagent via `delegate_task`. Give it its own profile with limited tools:

```
delegate_task(
    goal="Verify the output against the goal. You did not produce this work. Run the tests yourself. Report pass or fail with concrete reasons.",
    context="Goal: {{goal}}. Artifact: {{path}}",
    toolsets=["file", "terminal"]
)
```

The key properties of a proper checker:
- Its own context window (delegate_task guarantees this)
- No stake in the maker's choices
- Sees only the artifact and the standard
- Reports pass or fail with concrete reasons

For simple loops, goal_judge is sufficient. For loops producing artifacts (code, documents, configs), add the verifier.

**6. Put it on a timer.**

A goal-driven run still waits for you to type /goal. The next move is cadence. `cronjob` provides this with profile targeting so the loop runs in the right environment:

```
cronjob(
    action="create",
    name="nightly-build",
    schedule="0 2 * * *",
    profile="build-verifier",
    prompt="Seed the kanban with a 'nightly build' goal card. Let the dispatcher eat through the backlog.",
    deliver="telegram"
)
```

The cron seeds the kanban with a goal card. The dispatcher's 60s tick handles the rest -- picks up the card, decomposes it into work items, promotes through lanes, and runs worker agents. The cron starts the cycle; the kanban executes it. This IS the article's "timer turns a run into a habit."

The `profile` parameter routes the cron job to run under that profile's config, credentials, and tool restrictions — so the loop inherits the hardened environment you built in Step 2.

Key parameters specific to loop cadence:
- `profile`: runs the loop under the dedicated profile (Step 2)
- `deliver`: results surface where you can act on them
- `enabled_toolsets`: restrict to `["terminal", "file", "search"]` to avoid token waste on tools the loop never needs

**7. Compose complex work with workflows.**

Three Hermes-native shapes cover the article's workflow patterns:

**Fan out and synthesize** -- parallel subagents via delegate_task batch, or seed a kanban board with multiple independent cards that the dispatcher runs concurrently:

```
kanban_board: 'build-fixes'
cards:
  - 'Fix test_a: ...'
  - 'Fix test_b: ...'
  - 'Fix test_c: ...'
# dispatcher runs them in parallel — no card dependency
```

**Adversarial verification** -- maker card + verifier card where the verifier depends on the maker:
```
kanban dependency: verifier_card depends_on maker_card
dispatcher holds verifier until maker passes goal_judge
```
```
delegate_task(
    tasks=[
        {"goal": "Fix test_a", "toolsets": ["terminal", "file"]},
        {"goal": "Fix test_b", "toolsets": ["terminal", "file"]},
        {"goal": "Fix test_c", "toolsets": ["terminal", "file"]}
    ]
)
```

**Adversarial verification** -- maker subagent + checker subagent:
```
Delegate maker produces artifact.
Delegate verifier (with its own profile) checks it.
Only merge if both complete and verifier passes.
```

**Loop-until-condition** -- /goal with a cronjob that re-triggers itself until the goal_judge passes. Use `repeat` on the cronjob to limit iterations.

### Tier 3 -- Make it compound

**8. Give the loop memory -- append-only.**

The agent forgets everything between runs. The loop does not have to. But a single mutable state file corrupts when two concurrent runs write at the same time -- a real risk with multiple cron jobs or parallel kanban workers targeting the same project.

The fix is an append-only state log:

```
First run:    STATE_LOG += "2026-06-24T02:00|run-001|tried:fix-auth|result:failed|lesson:needs mock db"
Second run:   STATE_LOG += "2026-06-24T02:30|run-002|tried:fix-auth|result:passed|lesson:mock at module scope"
Third run:    read last 5 lines of STATE_LOG sees both entries
```

Each run appends a structured line (timestamp + run-id + what was tried + result + lesson). The next run reads the tail (`tail -n 20` or `read_file` with offset). Appends are atomic on most filesystems -- no clobbering, no locks.

Use the right layer for the job:
- **Append-only state log** (manual, structured): for tabular state -- what was tried, what failed, what the lesson was. Concurrency-safe by append.
- **Memory provider** (automatic, unstructured): for cross-session facts -- auto-extracts without manual writes. Already concurrency-safe.
- **`memory` tool** (manual curated): for facts that must persist verbatim -- pin a finding across sessions.

The two rules from the article apply:
- Write before walking away: the last action appends to the state log
- Read at the start: the first action reads the tail of it

Skip either and tomorrow restarts from zero.

**9. Distill lessons into skills.**

A state log dies with the project. The memory provider persists across sessions but the signal-to-noise ratio depends on what got extracted. The way to compound knowledge across projects is `skill_manage`.

When a loop hits a wall, the pattern is:

```
skill_manage(
    action="patch",
    name="<relevant-skill>",
    old_string="## Pitfalls\n\n- **Known failure mode**: ",
    new_string="## Pitfalls\n\n- **Known failure mode**: description of what failed\n- **Fix**: what resolved it"
)
```

This is the patch-on-fail lifecycle. Every time the loop encounters the same wall, the skill gets sharper. Run-level state goes in the state file; general lessons graduate into skills.

**Graduation threshold:** After 3 consecutive failures for the same issue (same goal, same queue item, or same failure mode), treat it as a repeat failure and call `skill_manage(action="patch")` to document the fix. The first two occurrences go to the state log as lessons; the third triggers a skill patch. This prevents over-patching on one-off failures while ensuring systematic issues get hardened.

The `curator` cron (if enabled) runs nightly and can identify skills that need updating from session patterns, but the primary mechanism is the loop patching itself after each block.

**10. Close the loop and make it fail safe.**

Now the parts lock together. The profile restricts what the loop can do. The goal_judge determines when it stops. cronjob provides cadence. Memory (auto-extraction + manual) compounds across runs. Skills harden over time.

An autonomous loop must also fail safely, because no one is watching each iteration. In Hermes, the guardrails are:

- **Profile-scoped toolsets**: `disabled_toolsets` or `enabled_toolsets` in profile.yaml prevents the loop from reaching for destructive tools. A build loop never needs write access to production configs.
- **Profile-scoped env**: credentials live in the profile's `.env`, not the global config. A loop cannot leak across trust boundaries.
- **Model routing by task cost**: configure per-loop in profile.yaml with a cheaper model for high-volume passes and a heavy model for orchestration/verification. The cronjob inherits the profile's model config.
- **Repeat limit on cron**: use the `repeat` parameter to prevent runaway loops that fail to converge.

A loop that runs unattended and cannot do anything irreversible is one you can actually leave alone.

## Pitfalls

- **Looping a thin harness**: A loop multiplies whatever is underneath it. A weak profile (full default toolset, no env scoping, no AGENTS.md) produces slop faster. Build Step 2 first.
- **Letting the maker grade itself**: goal_judge solves this for binary pass/fail. For artifact verification, delegate to a separate subagent. Never have the same agent produce and validate.
- **No stop condition**: Without a /goal and a working goal_judge, the loop halts at good enough. Verify goal_judge is configured and returns correct verdicts. Kanban cards without acceptance criteria in their goal have the same problem -- the dispatcher cannot know when a card is done.
- **No memory**: Without memory provider auto-extraction or explicit memory writes, every run starts from zero. This is where most of the compounding quietly leaks out.
- **Lessons that never leave the scope**: A general lesson patched into a project config file but not into a skill dies with the project. Graduate into skill_manage.
- **An unattended loop with full tool access**: No one is watching each step. Use enabled_toolsets and profile scoping to limit blast radius. Kanban workers inherit the profile's tool restrictions.
- **Top-tier model for every iteration**: Route cheap passes to a cheaper auxiliary model. An always-on loop on the flagship model bleeds tokens on work a smaller model handles fine.

## Verification

You have graduated from prompter to loop designer when: a dedicated profile exists with a restricted toolset and scoped env; a /goal with a configured goal_judge auxiliary evaluates completion independently; a cronjob with profile targeting and deliver provides cadence; the loop starts by loading prior state and ends by writing updated state back to memory or a state file; at least one failure-to-fix cycle has been patched into a skill; and the loop has run unattended through at least one full cycle without producing garbage or breaking things.
