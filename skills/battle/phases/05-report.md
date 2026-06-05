# Phase 05 — Report Generation

Compile all run artifacts into a human-readable battle report and produce the final sharpened skill file.

## Input / Output

| Item | Path |
|------|------|
| Analysis | `{run_dir}/analysis.json` |
| Scenarios | `{run_dir}/scenarios.jsonl` |
| Baseline evals | `{run_dir}/baseline-evals.jsonl` |
| Iterations | `{run_dir}/iterations/iter-*.json` |
| Progress log | `{run_dir}/progress.jsonl` |
| Spot-check | `{run_dir}/spot-check.json` (optional) |
| Original skill | `{run_dir}/skill-original.md` |
| Current skill | `{run_dir}/skill-current.md` |
| Output: report | `{run_dir}/report.md` |
| Output: sharpened | `{run_dir}/skill-sharpened.md` |

All input schemas defined in `references/contracts.md`. Dimension names defined in `references/taxonomy.md`.

## Steps

### 5.1 Load artifacts

Read every input file. Parse JSON/JSONL. If `spot-check.json` does not exist, set `spot_check = null`.

### 5.2 Compute scores

**Before (per dimension):** For each D1–D6, collect scenarios tagged with that dimension. Average their `score` from `baseline-evals.jsonl`. No scenarios for a dimension = "N/A".

**After (per dimension):** Find the highest `iteration` where `kept: true` in `iter-*.json`. Average scores from that version's evals per dimension. No kept iteration = After equals Before.

**Overall:** Mean of all scenario scores (Before from baseline, After from final kept evals).

**Deltas:** After minus Before. Explicit sign (`+X.X` / `-X.X` / `+0.0`). One decimal place throughout.

### 5.3 Build tables

**Difficulty counts:** Count scenarios per difficulty tier from `scenarios.jsonl`.

**Iteration log:** One row per `iter-*.json` file. Columns: Iter, Section (target_section), Role (target_section_role), Kept (yes/no), Before (score_before), After (score_after), Summary (first sentence of reasoning).

**Failure map:** One row per scenario. Columns: Scenario (id), Difficulty, Dimension, Failure Mode (failure_mode_target or "general"), Before (baseline score), After (final score), Status. Status rules:
- FIXED: before < 7 AND after >= 9
- IMPROVED: after > before AND not FIXED
- OPEN: after <= before OR after < 7

**Spot-check table:** If `spot_check` is not null, one row per check entry (Scenario, Simulated, Real, Delta, Flag). Otherwise "Not performed".

### 5.4 Identify findings

Select the top 1–5 most impactful changes. Each finding is one sentence stating: what was found, which dimension it affects, what was done, and the measured impact. Prioritize findings with the largest positive delta. Findings are actionable and specific — no vague summaries.

### 5.5 Remaining weaknesses

For each scenario with Status = OPEN in the failure map: describe the scenario, explain why it still fails (cite judge reasoning from the final evals), and suggest a concrete fix.

### 5.6 Write report.md

Assemble the report using this exact template structure:

```
# Battle Report: {skill_name}

**Date:** {timestamp}
**Skill:** {original_path}
**Scenarios:** {count} ({easy} easy / {medium} medium / {hard} hard / {adversarial} adversarial)
**Iterations:** {count} ({kept} kept, {discarded} discarded)

## Score Card
| Dimension | Before | After | Delta |
|-----------|--------|-------|-------|
| D1: Activation Reliability | X.X | X.X | +X.X |
| D2: Execution Compliance | X.X | X.X | +X.X |
| D3: Behavioral Alignment | X.X | X.X | +X.X |
| D4: Instruction Clarity | X.X | X.X | +X.X |
| D5: Architecture & Composability | X.X | X.X | +X.X |
| D6: Evolvability | X.X | X.X | +X.X |
| **Overall** | **X.X** | **X.X** | **+X.X** |

## Top Findings
1. {finding}
...

## Iteration Log
| Iter | Section | Role | Kept | Before | After | Summary |
|------|---------|------|------|--------|-------|---------|
...

## Spot-Check Validation
{table or "Not performed"}

## Failure Map
| Scenario | Difficulty | Dimension | Failure Mode | Before | After | Status |
|----------|------------|-----------|--------------|--------|-------|--------|
...

## Remaining Weaknesses
{For each OPEN scenario: description, why it fails, suggested fix}

## How to Apply
Review `skill-sharpened.md`, diff against `skill-original.md`, copy to the original path to adopt, re-run battle to verify.
```

### 5.7 Write skill-sharpened.md

Copy `{run_dir}/skill-current.md` to `{run_dir}/skill-sharpened.md`. No modifications.

## Output Checklist

- [ ] `report.md` exists and follows the exact template structure
- [ ] All scores formatted to one decimal place
- [ ] All deltas show explicit `+` or `-` sign
- [ ] Dimension names match `references/taxonomy.md` exactly
- [ ] Iteration log has one row per iteration file
- [ ] Failure map has one row per scenario
- [ ] `skill-sharpened.md` is byte-identical to `skill-current.md`
- [ ] Findings are specific and actionable (no vague summaries)
