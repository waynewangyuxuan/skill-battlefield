# Phase 02 — Generate Scenarios

Create diverse test scenarios from the skill analysis.

## Input / Output

| Item | Path |
|------|------|
| Analysis | `{run_dir}/analysis.json` |
| Ref: dimensions & failures | `references/taxonomy.md` (D1-D6, F1-F8) |
| Ref: check design | `references/rubric-guide.md` |
| Output file | `{run_dir}/scenarios.jsonl` |
| Config | `{num_scenarios}` (default 20) |

Output must conform to the `scenarios.jsonl` schema in `references/contracts.md`.

## Procedure

### 1. Load analysis

Read `{run_dir}/analysis.json`. Extract these fields for use in later steps:
- `intent`, `trigger_conditions`, `negative_triggers`
- `constraints`, `gates`
- `skill_type`, `estimated_complexity`

### 2. Plan difficulty distribution

Distribute `{num_scenarios}` across tiers (round to nearest integer):

| Tier | Share | Description |
|------|-------|-------------|
| easy | 25% | Clear happy path, unambiguous input |
| medium | 35% | Typical real-world usage, some ambiguity |
| hard | 25% | Edge cases, conflicting instructions, partial info |
| adversarial | 15% | Deliberately break the skill, exploit gaps |

### 3. Ensure dimension coverage

Cover all 6 dimensions (D1-D6) from `references/taxonomy.md`. Allocate at least 2 scenarios per dimension. Tag each scenario with its primary dimension.

Suggested mapping to difficulty tiers:

| Dimension | Natural tier fit |
|-----------|-----------------|
| D1 (Activation) | easy + adversarial (true/false positive) |
| D2 (Execution) | medium + hard (step skipping, gate adherence) |
| D3 (Behavioral) | hard + adversarial (edge cases, ambiguity) |
| D4 (Clarity) | medium (instruction following fidelity) |
| D5 (Architecture) | medium + hard (context bloat, composability) |
| D6 (Evolvability) | hard (drift, gotcha coverage) |

### 4. Target failure modes

For each scenario, set `failure_mode_target` to one of F1-F8 from `references/taxonomy.md` when the scenario specifically probes that failure. Set `null` when no specific failure mode is targeted.

Prioritize failure modes relevant to the skill's `constraints` and `gates`.

### 5. Design expected_behaviors checks

Write 2-4 checks per scenario following `references/rubric-guide.md`:
- Each check is binary (MET/UNMET)
- One criterion per check
- Use observable actions: tool calls, questions asked, outputs produced
- Include at least one achievable-by-baseline check and one challenging check
- Use snake_case identifiers (e.g., `reads_file_before_edit`)

### 6. Build context fields

For each scenario, populate realistic `context` fields:

| Field | Guidance |
|-------|----------|
| `files_present` | 2-5 plausible paths matching `project_type` (e.g., `["src/index.ts", "package.json"]`) |
| `project_type` | Match the scenario domain: typescript, python, rust, go, etc. |
| `recent_error` | Include a realistic error string for debugging scenarios; `null` otherwise |

### 7. Self-critique pass

Review all generated scenarios for:
- Redundancy: merge or differentiate scenarios testing the same thing
- Coverage gaps: verify all 6 dimensions have 2+ scenarios
- Tier balance: confirm distribution matches step 2
- Check quality: verify checks follow rubric-guide.md principles

### 8. Write output

Write each scenario as one JSON object per line to `{run_dir}/scenarios.jsonl` using Bash heredoc. Use sequential IDs: `scenario-001`, `scenario-002`, etc.

```bash
cat > "{run_dir}/scenarios.jsonl" << 'SCENARIOS_EOF'
{"id":"scenario-001", ...}
{"id":"scenario-002", ...}
SCENARIOS_EOF
```

Validate with:

```bash
python3 -c "
import json, sys
lines = open('{run_dir}/scenarios.jsonl').readlines()
assert len(lines) == {num_scenarios}, f'Expected {num_scenarios} lines, got {len(lines)}'
dims = set()
for i, line in enumerate(lines, 1):
    obj = json.loads(line)
    assert obj['id'] == f'scenario-{i:03d}', f'Bad ID on line {i}'
    assert len(obj['expected_behaviors']) >= 2, f'Too few checks on line {i}'
    dims.add(obj['dimension'])
assert len(dims) == 6, f'Only {len(dims)} dimensions covered: {dims}'
print('VALID')
"
```

## Output Checklist

- [ ] `scenarios.jsonl` exists at `{run_dir}/scenarios.jsonl`
- [ ] File has exactly `{num_scenarios}` lines
- [ ] All difficulty tiers represented (easy, medium, hard, adversarial)
- [ ] All 6 dimensions covered (at least 2 scenarios each)
- [ ] Every scenario has 2-4 expected_behaviors checks
- [ ] IDs are sequential: scenario-001 through scenario-{num_scenarios}
