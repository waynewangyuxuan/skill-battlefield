# Contracts — Shared Data Schemas

All phases read/write JSON files in the run directory (`~/.skill-battlefield/runs/<skill-name>/<timestamp>/`). These schemas define the shape of each file.

## analysis.json (Phase 01 output)

Written by 01-analyze. Read by all subsequent phases.

```json
{
  "skill_name": "string — name from frontmatter or first heading",
  "skill_type": "string — prose | checklist | multi-phase",
  "sections": [
    {
      "id": "string — s1, s2, ... sequential",
      "name": "string — heading text or 'Frontmatter' / 'Preamble'",
      "line_range": [number, number],
      "content_hash": "string — first 8 chars of md5 of section text",
      "role": "string — activation | instruction | gate | reference | meta"
    }
  ],
  "intent": "string — one sentence summarizing what the skill does",
  "trigger_conditions": ["string — phrases that should activate this skill"],
  "negative_triggers": ["string — phrases that should NOT activate this skill"],
  "constraints": ["string — explicit rules the skill enforces"],
  "gates": ["string — verification checkpoints or hard gates"],
  "estimated_complexity": "string — low | medium | high"
}
```

## scenarios.jsonl (Phase 02 output)

One JSON object per line:

```json
{
  "id": "string — scenario-001, scenario-002, ...",
  "description": "string — one sentence describing the test scenario",
  "user_message": "string — the exact message the simulated user sends",
  "context": {
    "files_present": ["string — file paths in the simulated project"],
    "recent_error": "string | null",
    "project_type": "string — e.g. typescript, python, rust"
  },
  "expected_behaviors": [
    {
      "check": "string — short identifier like asks_clarifying_question",
      "description": "string — natural language rubric the judge uses"
    }
  ],
  "difficulty": "string — easy | medium | hard | adversarial",
  "dimension": "string — D1 | D2 | D3 | D4 | D5 | D6",
  "failure_mode_target": "string — F1-F8 | null",
  "tags": ["string"]
}
```

## baseline-evals.jsonl (Phase 03 output)

One eval result per line. Also used by Phase 04 for re-evaluation results.

```json
{
  "scenario_id": "string — matches scenario.id",
  "skill_version": "string — v0 for original, v1/v2/... for rewrites",
  "simulated_response": "string — what the simulator described the agent would do",
  "checks": [
    {
      "check": "string — matches expected_behaviors[].check",
      "pass": "boolean",
      "reasoning": "string — why this check passed or failed"
    }
  ],
  "score": "number — 1-10 overall score",
  "dimension_scores": {
    "D1": "number | null",
    "D2": "number | null",
    "D3": "number | null",
    "D4": "number | null",
    "D5": "number | null",
    "D6": "number | null"
  },
  "judge_reasoning": "string — overall assessment"
}
```

## iterations/iter-N.json (Phase 04 output)

One file per sharpening iteration:

```json
{
  "iteration": "number — 1-indexed",
  "target_section": "string — section id from analysis.json",
  "target_section_role": "string — section role",
  "failure_pattern": "string — description of the failure cluster",
  "diff": "string — unified diff of the rewrite",
  "kept": "boolean",
  "score_before": "number",
  "score_after": "number",
  "regressed_scenarios": ["string — scenario ids that regressed"],
  "reasoning": "string — why kept or discarded"
}
```

## progress.jsonl (Phase 04, append-only)

```json
{
  "timestamp": "string — ISO 8601",
  "event": "string — iteration_start | iteration_end | early_stop | rewrite_discarded",
  "iteration": "number",
  "score": "number",
  "detail": "string"
}
```

## spot-check.json (Spot-check phase output)

```json
{
  "checks": [
    {
      "scenario_id": "string",
      "simulated_score": "number",
      "real_score": "number",
      "delta": "number",
      "flag": "string — OK | SIMULATION-OPTIMISTIC | SIMULATION-PESSIMISTIC"
    }
  ],
  "summary": "string — overall assessment of simulation reliability"
}
```
