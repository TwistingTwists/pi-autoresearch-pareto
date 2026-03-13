---
name: autoresearch-create
description: Set up and run an autonomous experiment loop for any optimization target. Gathers what to optimize, then starts the loop immediately. Use when asked to "run autoresearch", "optimize X in a loop", "set up autoresearch for X", or "start experiments".
---

# Autoresearch

Autonomous experiment loop: try ideas, keep what works, discard what doesn't, never stop.

## Tools

- **`init_experiment`** — configure session. Supports three modes:
  - `single_objective` — one metric
  - `threshold_then_optimize` — metric A must pass a threshold, then optimize metric B
  - `frontier_exploration` — explore the non-dominated Pareto frontier across 2-3 primary metrics
  Call again to re-initialize with a new baseline when the optimization target changes.
- **`run_experiment`** — runs command, times it, captures output.
- **`log_experiment`** — records result. For successful runs, provide the metrics and let the tool compute `keep` vs `discard` from the active mode. `keep` auto-commits. `discard`/`crash` → `git checkout -- .` to revert. Always include secondary `metrics` dict. Dashboard: ctrl+x.

## Setup

1. Ask (or infer): **Goal**, **Command**, **Experiment mode**, **Files in scope**, **Constraints**.
2. If the user wants Pareto behavior, ask which Pareto mode:
   - `threshold_then_optimize`: "Which metric must cross a threshold first, what is the threshold, and which second metric should improve after that?"
   - `frontier_exploration`: "Which 2-3 metrics define the tradeoff surface we want to map?"
   Ask explicitly before designing the experiment unless the user already answered it.
3. `git checkout -b autoresearch/<goal>-<date>`
4. Read the source files. Understand the workload deeply before writing anything.
5. Write `autoresearch.md` and `autoresearch.sh` (see below). Commit both.
6. `init_experiment` → run baseline → `log_experiment` → start looping immediately.

### `autoresearch.md`

This is the heart of the session. A fresh agent with no context should be able to read this file and run the loop effectively. Invest time making it excellent.

```markdown
# Autoresearch: <goal>

## Objective
<Specific description of what we're optimizing and the workload.>

## Metrics
- **Mode**: <single_objective | threshold_then_optimize | frontier_exploration>
- **Primary**: <name> (<unit>, lower/higher is better)
- **Secondary**: <name>, <name>, ...
- **Threshold rule**: <only for threshold_then_optimize>

## How to Run
`./autoresearch.sh` — outputs `METRIC name=number` lines.

## Files in Scope
<Every file the agent may modify, with a brief note on what it does.>

## Off Limits
<What must NOT be touched.>

## Constraints
<Hard rules: tests must pass, no new deps, etc.>

## What's Been Tried
<Update this section as experiments accumulate. Note key wins, dead ends,
and architectural insights so the agent doesn't repeat failed approaches.>
```

Update `autoresearch.md` periodically — especially the "What's Been Tried" section — so resuming agents have full context.

### `autoresearch.sh`

Bash script (`set -euo pipefail`) that: pre-checks fast (syntax errors in <1s), runs the benchmark, outputs `METRIC name=number` lines. Keep it fast — every second is multiplied by hundreds of runs. Update it during the loop as needed.

## Loop Rules

**LOOP FOREVER.** Never ask "should I continue?" — the user expects autonomous work.

- **Single objective:** improved primary metric → `keep`. Worse/equal → `discard`.
- **Threshold then optimize:** `keep` only if the threshold is satisfied and the optimize metric improves among feasible kept runs.
- **Frontier exploration:** `keep` only if the result is non-dominated by previous kept runs in the current segment.
- **Simpler is better.** Removing code for equal perf = keep. Ugly complexity for tiny gain = probably discard.
- **Don't thrash.** Repeatedly reverting the same idea? Try something structurally different.
- **Crashes:** fix if trivial, otherwise log and move on. Don't over-invest.
- **Think longer when stuck.** Re-read source files, study the profiling data, reason about what the CPU is actually doing. The best ideas come from deep understanding, not from trying random variations.
- **Resuming:** if `autoresearch.md` exists, read it + git log, continue looping.

**NEVER STOP.** The user may be away for hours. Keep going until interrupted.

## Ideas Backlog

When you discover complex but promising optimizations that you won't pursue right now, **append them as bullets to `autoresearch.ideas.md`**. Don't let good ideas get lost.

On resume (context limit, crash), check `autoresearch.ideas.md` — prune stale/tried entries, experiment with the rest. When all paths are exhausted, delete the file and write a final summary.

## User Messages During Experiments

If the user sends a message while an experiment is running, finish the current `run_experiment` + `log_experiment` cycle first, then incorporate their feedback in the next iteration. Don't abandon a running experiment.
