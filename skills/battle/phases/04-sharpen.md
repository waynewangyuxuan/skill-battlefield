# Phase 04 — Sharpen (Iteration Loop)

Analyze failures, rewrite targeted sections, re-evaluate, keep or discard. One section per iteration.

## Input / Output

| Item | Path | Default |
|------|------|---------|
| Original skill | `{run_dir}/skill-original.md` | — |
| Current skill | `{run_dir}/skill-current.md` | copy of original |
| Analysis | `{run_dir}/analysis.json` | — |
| Scenarios | `{run_dir}/scenarios.jsonl` | — |
| Baseline evals | `{run_dir}/baseline-evals.jsonl` | — |
| Output: iterations | `{run_dir}/iterations/iter-N.json` | — |
| Output: progress | `{run_dir}/progress.jsonl` | — |
| Config: max_iterations | — | `5` |
| Config: early_stop_delta | — | `0.2` |
| Config: max_drift | — | `0.4` (40% length change) |

All output schemas defined in `references/contracts.md`.

## Setup

1. Copy `skill-original.md` to `skill-current.md`
2. Parse `baseline-evals.jsonl`, compute baseline score = mean of all `score` fields
3. Record `original_length` = character count of `skill-original.md`
4. Create `{run_dir}/iterations/` directory
5. Initialize tracking: `current_evals` = baseline evals, `attempted_sections` = [], `consecutive_no_improvement` = 0
6. Append to `progress.jsonl`: `{"event": "sharpen_start", "iteration": 0, "score": <baseline>, "detail": "baseline score"}`

## Iteration Loop

For `iteration` = 1 to `max_iterations`, execute steps 4.1–4.7. Exit early per step 4.7.

### 4.1 Analyze failures

Read `current_evals`. Collect all checks where `pass: false`. For each failing check, map it to the section that governs the behavior using `analysis.json` section roles and the scenario's `dimension` field. Build a failure count per section.

### 4.2 Select target

Pick the section with the highest failure count. Break ties using priority order: instruction > activation > gate > reference > meta.

If every section with failures has already been attempted with no improvement, trigger early stop.

### 4.3 Rewrite

Propose a targeted rewrite of ONLY the selected section.

**Rules:**
- Change only the target section; all other sections remain byte-identical
- Prefer additions over deletions
- Use positive framing ("Add X" not "Remove ambiguity about X")
- Preserve the original intent of the section
- If a previous attempt on this section was discarded, incorporate the failed diff and judge reasoning to avoid repeating the same mistake

**Output:** Generate a unified diff. Apply it to `skill-current.md` to produce the candidate version. Validate the candidate is a complete, well-formed skill file.

### 4.4 Re-evaluate

Run a FULL evaluation of the candidate skill against ALL scenarios. Follow the two-step simulate-then-judge process described below. Set `skill_version` = `"v{iteration}"`.

Write results to a temporary evals file (not baseline-evals.jsonl).

#### Re-eval procedure (replicates Phase 03)

**Step A — Simulate.** For each scenario, prompt a simulator with:
- Inputs: full candidate skill text + `scenario.user_message` + `scenario.context`
- **The simulator does NOT receive `expected_behaviors`** (information boundary)
- Prompt: "You are a Claude Code agent with this skill loaded. Given this user message and context, describe what you would do: tools called, order, questions asked, output produced."
- Instruct the simulator to be realistic, not optimistic: agents skip redundant steps, miss nuance in vague instructions, default to base behavior when the skill is unclear, follow letter over spirit
- Generate a 100-200 word `simulated_response`

**Step B — Judge.** For each scenario, evaluate the simulated response:
- Inputs: `simulated_response` + `scenario.expected_behaviors`
- **The judge does NOT receive the skill text** (information boundary)
- Temperature 0
- For each check: binary pass/fail with one sentence of reasoning citing evidence
- Score: `(passed_checks / total_checks) * 10`, adjusted +/-1 for quality, clamped 1-10
- Borderline scores (4, 5, 6): evaluate 3 times, majority vote per check, recompute score
- Score the scenario's primary dimension; other dimensions null unless clearly evidenced

Output conforms to `baseline-evals.jsonl` schema in `references/contracts.md`.

### 4.5 Decide

Compute `score_after` = mean score across all re-eval results. Compare to `score_before` (current mean).

**KEEP** if both conditions are met:
- `score_after` > `score_before`
- No regressions on easy-tier scenarios (no scenario with `difficulty: "easy"` that previously passed all checks now fails any)

**DISCARD** if either:
- `score_after` <= `score_before`
- Any easy-tier regression exists

On **KEEP**: update `skill-current.md` with candidate, set `current_evals` to new evals, add section to `attempted_sections`, reset retry counter.

On **DISCARD**: revert to previous `skill-current.md`. Record the failed diff and judge reasoning. Retry the same iteration (same target section) with the failed attempt as context. Maximum 2 retries per iteration. After 2 failed retries, add section to `attempted_sections`, move to next iteration.

### 4.6 Log

Write `{run_dir}/iterations/iter-{iteration}.json` conforming to the `iter-N.json` schema in `references/contracts.md`. Include: iteration number, target section, failure pattern, unified diff, kept/discarded, score_before, score_after, regressed scenario IDs, reasoning.

Append to `progress.jsonl`:
- On keep: `{"event": "iteration_end", "iteration": N, "score": <score_after>, "detail": "kept: <section>"}`
- On discard: `{"event": "rewrite_discarded", "iteration": N, "score": <score_before>, "detail": "discarded: <section> — <reason>"}`

### 4.7 Check early stop

Stop the loop if ANY of these conditions are met:

1. **Plateau**: score delta < `early_stop_delta` for 2 consecutive kept iterations
2. **Drift**: `|len(skill-current.md) - original_length| / original_length` > `max_drift`
3. **Max iterations**: `iteration` >= `max_iterations`
4. **Exhaustion**: all sections with failures have been attempted with no improvement

On early stop, append: `{"event": "early_stop", "iteration": N, "score": <current>, "detail": "<reason>"}`

## Output Checklist

- [ ] `skill-current.md` is a complete, valid skill file
- [ ] `iterations/iter-N.json` exists for each completed iteration, conforming to contracts.md
- [ ] `progress.jsonl` has sharpen_start + one entry per iteration + early_stop (if applicable)
- [ ] All JSON output is valid (parseable by `json.loads`)
- [ ] `progress.jsonl` is append-only (no overwriting previous entries)
- [ ] Simulator never received expected_behaviors in any re-eval
- [ ] Judge never received skill text in any re-eval
- [ ] No easy-tier regressions in the final kept version
