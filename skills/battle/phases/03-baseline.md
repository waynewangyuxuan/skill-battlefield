# Phase 03 — Baseline Evaluation

Simulate agent behavior and judge it against expected behaviors. Two-step process with strict information separation.

## Input / Output

| Item | Path |
|------|------|
| Skill file | `{run_dir}/skill-original.md` (baseline) or `{run_dir}/skill-current.md` (re-eval) |
| Scenarios | `{run_dir}/scenarios.jsonl` |
| Ref: judge instructions | `references/rubric-guide.md` |
| Ref: eval schema | `references/contracts.md` (baseline-evals.jsonl) |
| Output file | `{run_dir}/baseline-evals.jsonl` |
| Config | `skill_version` — `"v0"` for baseline, `"v{N}"` for re-evals |

Output must conform to the `baseline-evals.jsonl` schema in `references/contracts.md`.

## Procedure

### 1. Load inputs

Read the skill file and `{run_dir}/scenarios.jsonl`. Parse each line of scenarios.jsonl into a scenario object.

### 2. Evaluate each scenario (simulate + judge)

Process scenarios in parallel batches of up to 5 using the Agent tool. Each sub-evaluator handles one scenario's full simulate-then-judge cycle.

For each scenario, execute Step 2A then Step 2B in sequence.

---

### 2A. Simulate

**HARD GATE — INFORMATION BOUNDARY**: The simulator receives ONLY these inputs:
- The full skill text
- `scenario.user_message`
- `scenario.context`

**HARD GATE**: The simulator does NOT receive `expected_behaviors`. Passing expected_behaviors to the simulator contaminates the evaluation and invalidates all results.

Prompt the simulator with:

> You are a Claude Code agent. You have this skill loaded and you received the following user message in the given project context. Describe in detail what you would do: which tools you would call, in what order, what questions you would ask the user, and what output you would produce.

Generate a 100-200 word structured description as `simulated_response`.

**Realism guidance** — instruct the simulator:
- A real agent might skip steps that seem redundant
- A real agent might miss nuance in vague or ambiguous instructions
- A real agent defaults to base behavior when the skill is unclear
- A real agent might follow the letter of a rule while missing its spirit
- Be realistic, not optimistic

---

### 2B. Judge

**HARD GATE — INFORMATION BOUNDARY**: The judge receives ONLY these inputs:
- `simulated_response` (from Step 2A)
- `scenario.expected_behaviors`

**HARD GATE**: The judge does NOT receive the skill text. Providing the skill text to the judge causes rubric contamination — the judge would infer passing criteria from the skill rather than evaluating purely against the rubric.

Use temperature 0 for determinism.

For each check in `expected_behaviors`:
1. Compare the check's `description` against the `simulated_response`
2. Assign `pass: true` or `pass: false`
3. Write one sentence of `reasoning` that cites specific evidence from the simulated response

**Score calculation:**
1. Base score: `(passed_checks / total_checks) * 10`, rounded to nearest integer
2. Adjust by +1 or -1 for overall quality factors (depth of reasoning, specificity of actions)
3. Clamp to range 1-10

**Dimension scores:**
- Score the scenario's primary `dimension` (from the scenario object) using the same 1-10 score
- Set all other dimensions to `null` unless the simulated response clearly demonstrates strength or weakness in another dimension

**Borderline retry:** If the score is 4, 5, or 6, repeat the full judge step 3 times and take the majority result for each check. Recompute the score from majority-voted checks.

### 3. Write output

Write each eval result as one JSON object per line to `{run_dir}/baseline-evals.jsonl`. Use echo append, not heredoc for the whole file:

```bash
echo '{"scenario_id":"scenario-001", ...}' >> "{run_dir}/baseline-evals.jsonl"
echo '{"scenario_id":"scenario-002", ...}' >> "{run_dir}/baseline-evals.jsonl"
```

Set `skill_version` to `"v0"` for baseline or `"v{N}"` for re-evaluations.

### 4. Validate output

```bash
python3 -c "
import json, sys
scenarios = open('{run_dir}/scenarios.jsonl').readlines()
evals = open('{run_dir}/baseline-evals.jsonl').readlines()
assert len(evals) == len(scenarios), f'Line count mismatch: {len(evals)} evals vs {len(scenarios)} scenarios'
for i, line in enumerate(evals, 1):
    obj = json.loads(line)
    assert 'scenario_id' in obj, f'Missing scenario_id on line {i}'
    assert 'simulated_response' in obj, f'Missing simulated_response on line {i}'
    assert isinstance(obj['checks'], list), f'checks not a list on line {i}'
    for c in obj['checks']:
        assert isinstance(c['pass'], bool), f'check.pass not boolean on line {i}'
        assert isinstance(c['reasoning'], str), f'check.reasoning not string on line {i}'
    assert 1 <= obj['score'] <= 10, f'Score out of range on line {i}'
print('VALID')
"
```

## Output Checklist

- [ ] `baseline-evals.jsonl` exists at `{run_dir}/baseline-evals.jsonl`
- [ ] Line count matches `scenarios.jsonl`
- [ ] Every line has all schema fields: scenario_id, skill_version, simulated_response, checks, score, dimension_scores, judge_reasoning
- [ ] Every check has `pass` (boolean) and `reasoning` (string citing evidence)
- [ ] Scores are integers 1-10
- [ ] Borderline scores (4, 5, 6) were evaluated 3 times with majority vote
- [ ] Simulator never received expected_behaviors
- [ ] Judge never received skill text
