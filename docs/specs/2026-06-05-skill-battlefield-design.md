# Skill Battlefield — Implementation Design

Date: 2026-06-05
Status: APPROVED
Repo: waynewangyuxuan/skill-battlefield

## Philosophy

**AI tests, humans write.** Skill Battlefield is a Claude Code plugin that stress-tests AI agent skills across diverse scenarios, produces diagnostic evidence, and proposes targeted improvements — but the human skill author makes the final call on what to adopt.

## Plugin Structure

```
skill-battlefield/
├── plugin.json
├── skills/
│   ├── battle/
│   │   ├── SKILL.md               # /battle — full loop orchestrator
│   │   ├── phases/
│   │   │   ├── 01-analyze.md      # Parse skill, classify, extract sections
│   │   │   ├── 02-generate.md     # Generate test scenarios
│   │   │   ├── 03-baseline.md     # Simulate + judge all scenarios
│   │   │   ├── 04-sharpen.md      # Rewrite loop (runs internally)
│   │   │   └── 05-report.md       # Emit report + skill-sharpened.md
│   │   └── references/
│   │       ├── contracts.md       # Scenario, eval-result, report schemas
│   │       ├── taxonomy.md        # D1-D6 dimensions, F1-F8 failure modes
│   │       └── rubric-guide.md    # Binary per-criterion rubric design
│   ├── diagnose/
│   │   └── SKILL.md               # /diagnose — Phase 2
│   ├── ablate/
│   │   └── SKILL.md               # /ablate — Phase 2
│   └── scenarios/
│       └── SKILL.md               # /scenarios — Phase 2
└── research/
    └── 00-synthesis.md
```

## Invocation

```
/battle <skill-path> [--scenarios 20] [--iterations 5]
```

- `skill-path` (required): path to a SKILL.md file
- `--scenarios N` (default: 20): number of scenarios to generate
- `--iterations N` (default: 5): max sharpening iterations

## Output Directory

All output lives in `~/.skill-battlefield/runs/<skill-name>/<timestamp>/`:

```
├── analysis.json          # Phase 01 output
├── scenarios.jsonl        # Phase 02 output
├── baseline-evals.jsonl   # Phase 03 output
├── iterations/
│   └── iter-N.json        # Phase 04 per-iteration results
├── progress.jsonl         # Live iteration log
├── spot-check.json        # Real execution validation
├── report.md              # Phase 05 final report
├── skill-sharpened.md     # Proposed improved skill
└── skill-original.md      # Copy of original
```

## Implementation: Approach C — Subagent-per-Phase

SKILL.md is a pure orchestrator. Each phase is dispatched as a subagent with its phase file as instructions. This provides context isolation — each subagent only loads what it needs, preventing context window blowout during 20+ scenarios x 5 iterations.

## Orchestrator Flow (SKILL.md)

```
1. Parse arguments: skill-path, --scenarios, --iterations
2. Validate: skill-path exists, is readable
3. Create run dir: ~/.skill-battlefield/runs/<skill-name>/<timestamp>/
4. Copy original skill to run dir as skill-original.md

Phase 01 — Analyze
   Dispatch subagent with 01-analyze.md
   Input: skill-original.md
   Output: analysis.json

Phase 02 — Generate
   Dispatch subagent with 02-generate.md
   Input: analysis.json
   Output: scenarios.jsonl

Phase 03 — Baseline
   Dispatch subagent with 03-baseline.md
   Input: skill-original.md + scenarios.jsonl
   Output: baseline-evals.jsonl

Phase 04 — Sharpen (loop, internal to subagent)
   Dispatch subagent with 04-sharpen.md
   Input: skill-original.md + scenarios.jsonl + baseline-evals.jsonl
   Config: max iterations, early-stop criteria
   Output: iterations/iter-N.json, skill-current.md, progress.jsonl

Spot-Check
   Dispatch subagent to run 3-5 scenarios via real `claude -p`
   Compare simulated vs real scores
   Output: spot-check.json

Phase 05 — Report
   Dispatch subagent with 05-report.md
   Input: all artifacts in run dir
   Output: report.md + skill-sharpened.md

5. Print summary: score before/after, top 3 findings, path to report
```

## Phase 01 — Analyze

Subagent parses the target skill and outputs structured analysis.

**Input**: `skill-original.md`

**Output**: `analysis.json`

```json
{
  "skill_name": "investigate",
  "skill_type": "multi-phase",
  "sections": [
    {
      "id": "s1",
      "name": "Trigger conditions",
      "line_range": [3, 15],
      "content_hash": "abc123",
      "role": "activation"
    }
  ],
  "intent": "Systematic debugging with root cause investigation",
  "trigger_conditions": ["debug", "fix bug", "why is this broken"],
  "negative_triggers": ["build feature", "refactor"],
  "constraints": ["no fixes without root cause"],
  "gates": ["investigate before hypothesize"],
  "estimated_complexity": "high"
}
```

**Section roles** (5 types):
- **activation**: description, when_to_use, trigger conditions
- **instruction**: actual steps, behavioral directives
- **gate**: verification checkpoints, hard gates
- **reference**: examples, templates, supporting material
- **meta**: frontmatter, metadata, comments

Section roles inform downstream phases: rewriter knows which type of section to target, evaluator knows which dimensions each section affects.

## Phase 02 — Generate Scenarios

**Input**: `analysis.json`

**Output**: `scenarios.jsonl`

```json
{
  "id": "scenario-007",
  "description": "User asks to debug but provides no error message",
  "user_message": "fix my test",
  "context": {
    "files_present": ["src/app.ts", "tests/app.test.ts"],
    "recent_error": null,
    "project_type": "typescript"
  },
  "expected_behaviors": [
    {"check": "asks_clarifying_question", "description": "Should ask which test or for error output"},
    {"check": "does_not_blindly_edit", "description": "Should not modify code without understanding failure"}
  ],
  "difficulty": "medium",
  "dimension": "D3",
  "failure_mode_target": "F4",
  "tags": ["ambiguous-input", "safety"]
}
```

**Generation strategy**:
1. Extract intent from analysis
2. Distribute across 4 difficulty tiers: Easy (25%), Medium (35%), Hard (25%), Adversarial (15%)
3. Cover all 6 dimensions — at least 2 scenarios per dimension
4. Self-critique pass: check for redundancy and coverage gaps

The `context` field provides project state (files, language, git status) because many skills behave differently based on environment, not just user message.

## Phase 03 — Baseline Eval

Two-step separation: Simulator and Judge are independent LLM calls.

**Input**: `skill-original.md` + `scenarios.jsonl`

**Output**: `baseline-evals.jsonl`

**Step 1 — Simulator**:
- Receives: skill text + scenario.user_message + scenario.context
- Does NOT receive: expected_behaviors (prevents rubric contamination)
- Role: "You are a Claude Code agent that just received this skill and user message. Describe what you would do — which tools, what order, what output."
- Outputs: structured simulated response

**Step 2 — Judge**:
- Receives: simulated_response + expected_behaviors
- Does NOT receive: skill text (prevents judge inferring correct answer)
- Evaluates each check: binary pass/fail + reasoning
- Aggregates to 1-10 score

```json
{
  "scenario_id": "scenario-007",
  "skill_version": "v0",
  "simulated_response": "Agent would first Read tests/app.test.ts...",
  "checks": [
    {"check": "asks_clarifying_question", "pass": true, "reasoning": "Asked 'which test file?'"},
    {"check": "does_not_blindly_edit", "pass": true, "reasoning": "No Edit before gathering info"}
  ],
  "score": 9,
  "dimension_scores": {"D1": null, "D2": 9, "D3": 9, "D4": null, "D5": null, "D6": null},
  "judge_reasoning": "Both checks met. Minor: could be more specific."
}
```

**Consistency**: Judge uses temperature 0. Scores within ±1 of pass/fail threshold get 3x majority vote.

**Parallelism**: Subagent can dispatch multiple simulator+judge pairs in parallel via Agent tool.

## Phase 04 — Sharpen (Loop)

Single subagent runs the full iteration loop internally.

**Input**: `skill-original.md` + `scenarios.jsonl` + `baseline-evals.jsonl` + config

**Output**: `iterations/iter-N.json` + `skill-current.md` + `progress.jsonl`

**Per-iteration steps**:

```
1. Analyze failures — find failing checks, cluster by section
2. Identify pattern — "Section s3 causes 3/20 scenarios to skip step 2"
3. Select target — pick section with highest failure impact, one section per iteration
4. Rewrite — propose targeted diff with reasoning
5. Re-eval — rerun ALL scenarios with rewritten skill (regression check)
6. Decision:
   - Overall score up + easy cases don't regress → KEEP
   - Score down or easy regress → DISCARD, feed failure to next attempt
7. Log to progress.jsonl
```

**Rewrite feedback loop**: On discard, next rewrite attempt receives the failed diff + judge reasoning. Max 2 retries per iteration; if all fail, mark "no improvement found" and move to next iteration targeting a different section.

**Early stop** (any triggers):
- Score delta < 0.2 for 2 consecutive iterations
- Cumulative diff > 40% of original skill length
- Max iterations reached

**Per-iteration output** (`iter-N.json`):
```json
{
  "iteration": 1,
  "target_section": "s3",
  "target_section_role": "instruction",
  "failure_pattern": "3 scenarios: agent skips step 2 in ambiguous input cases",
  "diff": "- Follow the four phases\n+ Follow the four phases. Do NOT skip to implementation before completing investigation.",
  "kept": true,
  "score_before": 6.2,
  "score_after": 7.0,
  "regressed_scenarios": [],
  "reasoning": "Added explicit gate language. 3 previously failing scenarios now pass."
}
```

## Spot-Check Validation

After Phase 04 converges, before Phase 05 report.

**Select 3-5 scenarios**:
- 1x highest simulated score (check for optimism bias)
- 1x lowest simulated score (verify it's truly bad)
- 1-2x scores near threshold (borderline, most valuable to verify)
- 1x adversarial (simulation most likely to be optimistic here)

**Execution**: Subagent runs `claude -p` with the skill + scenario, then judges the real output.

**Divergence handling**:
- ≤ 2 points: normal, noted in report
- > 2 points: flagged as "simulation-optimistic" or "simulation-pessimistic" in report

**Cost**: ~$2 for 3-5 real executions. Acceptable.

## Phase 05 — Report

**Input**: all artifacts in run directory

**Output**: `report.md` + `skill-sharpened.md`

**`report.md` structure**:

```markdown
# Battle Report: <skill-name>
Date: <timestamp>
Skill: <original-path>

## Score Card
| Dimension | Before | After | Delta |
|-----------|--------|-------|-------|
| D1: Activation Reliability | 7.0 | 7.0 | — |
| D2: Execution Compliance | 5.5 | 8.0 | +2.5 |
| D3: Behavioral Alignment | 6.0 | 7.5 | +1.5 |
| ... | | | |
| **Overall** | **6.2** | **7.8** | **+1.6** |

## Top Findings
1. Section "Phase 2 instructions" caused step-skipping → rewritten, +2.5 on D2
2. Section "Trigger conditions" is dead text — ablation showed 0 score impact
3. 2 adversarial scenarios still fail — skill has no prompt injection handling

## Iteration Log
| Iter | Section | Kept | Delta | Summary |
|------|---------|------|-------|---------|
| 1 | s3 (instruction) | yes | +0.8 | Added gate language |
| 2 | s5 (reference) | no | -0.3 | Removing examples regressed easy cases |

## Spot-Check Validation
| Scenario | Simulated | Real | Delta | Flag |
|----------|-----------|------|-------|------|
| scenario-003 | 9 | 8 | -1 | OK |
| scenario-012 | 3 | 2 | -1 | OK |
| scenario-018 | 7 | 4 | -3 | SIMULATION-OPTIMISTIC |

## Failure Map
| Scenario | Difficulty | Dimension | Failure Mode | Status |
|----------|-----------|-----------|--------------|--------|
| scenario-007 | medium | D3 | F4 | FIXED |
| scenario-012 | adversarial | D3 | F5 | OPEN |

## Remaining Weaknesses
- [still-failing scenarios with reasoning]

## How to Apply
- Review skill-sharpened.md in this directory
- Diff: `diff skill-original.md skill-sharpened.md`
- Copy sections you agree with into your skill
```

**`skill-sharpened.md`**: Complete, usable skill file incorporating all kept rewrites. Original skill is never modified.

## Evaluation Taxonomy (Reference)

### 6 Dimensions (D1-D6)
- **D1: Activation Reliability** — triggers when it should, doesn't when it shouldn't
- **D2: Execution Compliance** — steps actually followed, gates respected
- **D3: Behavioral Alignment** — edge cases, ambiguity, adversarial handling
- **D4: Instruction Clarity** — positive framing, structure, non-default knowledge
- **D5: Architecture & Composability** — scope focus, context efficiency, context utilization
- **D6: Evolvability** — version tracking, gotchas, drift resistance

### Context Utilization (D5 sub-dimension)
Three layers of observability:
1. **KV cache**: which tokens entered context window after compaction (indirect inference only)
2. **Attention attribution**: which tokens influenced output (not available via API)
3. **Behavioral ablation** (actionable): remove section, re-eval, score unchanged → dead text

### 8 Failure Modes (F1-F8)
- F1: Never activates
- F2: Activates on wrong input
- F3: Skips middle steps
- F4: Follows letter, misses spirit
- F5: Pink elephant (negative instructions backfire)
- F6: Context bloat
- F7: Instruction stacking erosion
- F8: Gotcha amnesia

## Phase 2 Commands (Future)

| Command | What it does | Reuses from /battle |
|---------|-------------|-------------------|
| `/diagnose <skill-path>` | Static analysis only — no scenarios, no eval. Section roles, dead text candidates, activation description quality, structure score. | 01-analyze + subset of taxonomy scoring |
| `/ablate <skill-path>` | Behavioral ablation — remove each section, re-eval, report saliency. | 01-analyze + 02-generate + 03-baseline + ablation logic |
| `/scenarios <skill-path>` | Generate scenarios only, output for human review. | 01-analyze + 02-generate |

## Cost Model

For 20 scenarios, 5 iterations:
- Scenario generation: ~$0.06
- Baseline eval (20x simulator + judge): ~$0.24
- Per iteration (1 rewrite + 20 re-evals): ~$0.29
- 5 iterations: ~$1.45
- Spot-check (3-5 real executions): ~$2.00
- Report generation: ~$0.03
- **Total: ~$3.78**

Overnight mode (50 scenarios, 20 iterations): ~$17-22.

## Key Design Decisions

1. **Form factor**: Claude Code plugin with multiple skills
2. **Implementation**: Pure skill orchestration, zero code dependencies
3. **Approach**: Subagent-per-phase for context isolation
4. **Eval method**: Simulated execution + real spot-check validation
5. **Rewrite strategy**: One section per iteration, human reviews final result
6. **Output location**: `~/.skill-battlefield/runs/` (centralized, not in skill repo)
7. **Original skill**: Never modified — `skill-sharpened.md` is a proposal
