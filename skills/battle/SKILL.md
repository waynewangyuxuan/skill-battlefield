---
name: battle
description: "Stress-test a Claude Code skill across diverse scenarios. Generates test scenarios (easy/medium/hard/adversarial), evaluates via simulated execution, proposes targeted improvements, and produces an evidence report. Use when: user wants to test a skill, evaluate skill quality, find weaknesses, sharpen a skill, or run skill diagnostics. Trigger on: 'battle', 'test this skill', 'evaluate my skill', 'sharpen', 'find weaknesses', 'skill quality'. DO NOT use for: writing new skills from scratch, general code review, non-skill markdown files."
user-invocable: true
argument-hint: "<skill-path> [--scenarios 20] [--iterations 5]"
---

# Skill Battlefield

Stress-test any Claude Code skill: generate scenarios, evaluate, sharpen, and report.

## Argument Parsing

Parse `$ARGUMENTS`:
- First positional argument = `skill_path` (required)
- `--scenarios N` = `num_scenarios` (default 20)
- `--iterations N` = `max_iterations` (default 5)

If no `skill_path` provided, print usage and stop:
> Usage: /battle <skill-path> [--scenarios 20] [--iterations 5]

Resolve `~` in `skill_path` to the absolute home directory path.

## Validation

1. Verify `skill_path` exists and is readable
2. Verify the file contains markdown content (not binary, not empty)

If invalid, print an error message describing the problem and stop.

## Setup

1. Extract `skill_name` from the parent directory name (e.g., `my-skill/SKILL.md` -> `my-skill`)
2. Set `skill_dir` to the parent directory of `skill_path`
3. Create timestamp via `date +%Y%m%d-%H%M%S`
4. Create run directory: `~/.skill-battlefield/runs/{skill_name}/{timestamp}/`
5. Create `{run_dir}/iterations/` subdirectory
6. Copy the skill file to `{run_dir}/skill-original.md`
7. Print: `Starting battle for {skill_name}. Run directory: {run_dir}`

Set `phases_dir` to the `phases/` directory relative to this skill file.
Set `refs_dir` to the `references/` directory relative to this skill file.

## Phase 01 — Analyze

Read `{phases_dir}/01-analyze.md` and `{refs_dir}/contracts.md`.

Dispatch a subagent with:
- Full content of `01-analyze.md` as instructions
- Content of `contracts.md` (analysis.json schema)
- `run_dir` and `skill_dir` paths

**HARD GATE**: Verify `{run_dir}/analysis.json` exists after completion. If missing, stop with error.

Print: `Phase 01 complete: analysis.json written`

## Phase 02 — Generate Scenarios

Read `{phases_dir}/02-generate.md`, `{refs_dir}/contracts.md`, `{refs_dir}/taxonomy.md`, and `{refs_dir}/rubric-guide.md`.

Dispatch a subagent with:
- Full content of `02-generate.md` as instructions
- Content of `contracts.md`, `taxonomy.md`, and `rubric-guide.md`
- `run_dir`, `skill_dir`, and `num_scenarios`

**HARD GATE**: Verify `{run_dir}/scenarios.jsonl` exists. Count lines and confirm it matches `num_scenarios`. If invalid, stop with error.

Print: `Phase 02 complete: {num_scenarios} scenarios generated`

## Phase 03 — Baseline Evaluation

Read `{phases_dir}/03-baseline.md`, `{refs_dir}/contracts.md`, and `{refs_dir}/rubric-guide.md`.

Dispatch a subagent with:
- Full content of `03-baseline.md` as instructions
- Content of `contracts.md` and `rubric-guide.md`
- `run_dir`, `skill_dir`, and `skill_version = "v0"`

**HARD GATE**: Verify `{run_dir}/baseline-evals.jsonl` exists. If missing, stop with error.

Calculate baseline score: mean of all `score` fields from `baseline-evals.jsonl`.

Print: `Phase 03 complete: baseline score = {baseline_score}`

## Phase 04 — Sharpen

Read `{phases_dir}/04-sharpen.md`, `{refs_dir}/contracts.md`, `{refs_dir}/taxonomy.md`, and `{refs_dir}/rubric-guide.md`.

Dispatch a subagent with:
- Full content of `04-sharpen.md` as instructions
- Content of `contracts.md`, `taxonomy.md`, and `rubric-guide.md`
- `run_dir`, `skill_dir`, and `max_iterations`

**HARD GATE**: Verify `{run_dir}/skill-current.md` exists. If missing, stop with error.

Read `{run_dir}/progress.jsonl`. Extract the final score from the last entry. Compute delta from baseline.

Print: `Phase 04 complete: score {baseline_score} -> {final_score} (delta {delta})`

## Spot-Check Validation

Select 3-5 scenarios from `{run_dir}/scenarios.jsonl` for live validation:
- 1x scenario with the highest simulated score
- 1x scenario with the lowest simulated score
- 1-2x scenarios with scores near the threshold (score 5 or 6)
- 1x adversarial scenario (difficulty = "adversarial")

Read the final `{run_dir}/skill-current.md`.

For each selected scenario:
1. Run `claude -p` with the skill content prepended to the user_message
2. Judge the real response against the scenario's `expected_behaviors` using the same binary check rubric
3. Compute a real_score using the same formula as Phase 03
4. Compare `simulated_score` vs `real_score`, compute delta
5. Flag: `|delta| <= 1` = OK, `delta > 1` = SIMULATION-OPTIMISTIC, `delta < -1` = SIMULATION-PESSIMISTIC

Write `{run_dir}/spot-check.json` conforming to the schema in `references/contracts.md`.

Print average divergence: `Spot-check: avg divergence = {avg_delta}, {num_ok}/{total} checks within tolerance`

## Phase 05 — Report

Read `{phases_dir}/05-report.md`, `{refs_dir}/contracts.md`, and `{refs_dir}/taxonomy.md`.

Dispatch a subagent with:
- Full content of `05-report.md` as instructions
- Content of `contracts.md` and `taxonomy.md`
- `run_dir` and `skill_dir`

**HARD GATE**: Verify `{run_dir}/report.md` and `{run_dir}/skill-sharpened.md` both exist. If either is missing, stop with error.

## Summary

After all phases complete, read `{run_dir}/report.md` and extract the top findings.

Print to user:

```
=== Battle Complete ===
Skill: {skill_name}
Score: {baseline_score} -> {final_score} ({delta})
Top findings:
  1. {finding_1}
  2. {finding_2}
  ...
Report: {run_dir}/report.md
Sharpened skill: {run_dir}/skill-sharpened.md
Diff: diff {run_dir}/skill-original.md {run_dir}/skill-sharpened.md
```
