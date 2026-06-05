# Skill Battlefield `/battle` v1 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `/battle` command as a Claude Code plugin — a multi-phase skill that stress-tests other skills via scenario generation, simulated evaluation, targeted rewriting, spot-check validation, and evidence reporting.

**Architecture:** Pure skill orchestration (zero code dependencies). A `SKILL.md` orchestrator dispatches one subagent per phase. Each phase reads/writes files in a run directory (`~/.skill-battlefield/runs/`). Reference files provide shared contracts, taxonomy, and rubric guidance.

**Tech Stack:** Claude Code plugin (plugin.json + SKILL.md files), markdown instructions only, Agent tool for subagent dispatch, Bash tool for file I/O within phases.

**Meta-constraint:** We are building skills to optimize skills. Every skill file we write must itself follow the best practices we researched. Before writing each phase file, research how similar skill instructions are written in the wild (superpowers, Anthropic official skills, top community skills). Apply our own taxonomy (D1-D6) as a self-check. Dogfood relentlessly.

---

## Research-Before-Writing Protocol

For EACH task that creates a skill file (Tasks 4-9), the implementer MUST:

1. **Study reference skills** — Read 2-3 existing skills that do something similar:
   - For orchestrator (SKILL.md): study `superpowers:brainstorming`, `superpowers:writing-plans`, or any multi-phase skill with subagent dispatch
   - For phase files: study how `robin:robin-executor`, `robin:robin-reviewer`, or similar subagent instruction files are structured
   - For the evaluator/judge role: look at how `code-review` or `superpowers:requesting-code-review` frame evaluation criteria
   - Search GitHub for any published skill that does scenario generation, LLM-as-judge, or iterative optimization

2. **Apply our own 12 principles** (from research/00-synthesis.md):
   - Positive framing (no pink elephants)
   - Structure for scanning (headers, bullets, tables — not prose)
   - Only non-default knowledge (don't tell the model what it already knows)
   - Hard gates where compliance matters
   - Directive description (pushy, with negative triggers)
   - Progressive disclosure (keep phase file lean, reference files for details)

3. **Self-evaluate against D1-D6** before committing:
   - D1: Will the orchestrator reliably dispatch this phase? Is the prompt clear enough?
   - D2: Will the subagent follow all steps? Are there skippable middle steps?
   - D3: What happens with weird input? Empty skill? Massive skill? Binary file?
   - D4: Is every instruction actionable? Any vague language?
   - D5: Is this file under 200 lines? Does it reference rather than inline?
   - D6: Can this phase be improved independently without breaking others?

4. **Log research findings** — After each research step, append a brief note to `research/skill-authoring-notes.md` with what you learned and how it influenced the file you wrote.

---

## File Structure

```
skill-battlefield/
├── .claude-plugin/
│   └── plugin.json                          # Plugin manifest
├── skills/
│   └── battle/
│       ├── SKILL.md                         # /battle orchestrator — dispatches subagents
│       ├── phases/
│       │   ├── 01-analyze.md                # Parse skill, classify type, extract sections
│       │   ├── 02-generate.md               # Generate test scenarios across difficulty tiers
│       │   ├── 03-baseline.md               # Simulate execution + judge (two-step, per scenario)
│       │   ├── 04-sharpen.md                # Rewrite loop: analyze failures → rewrite → re-eval
│       │   └── 05-report.md                 # Emit report.md + skill-sharpened.md
│       └── references/
│           ├── contracts.md                 # JSON schemas for scenario, eval-result, iter-result, report
│           ├── taxonomy.md                  # D1-D6 dimensions, F1-F8 failure modes, section roles
│           └── rubric-guide.md              # How to design binary per-criterion rubrics
├── test-fixtures/
│   └── simple-task-skill.md                 # A small test skill for validating each phase
├── docs/
│   ├── specs/
│   │   └── 2026-06-05-skill-battlefield-design.md
│   └── plans/
│       └── 2026-06-05-battle-skill-v1.md    # This file
└── research/
    └── 00-synthesis.md
```

---

### Task 1: Plugin Scaffold + Test Fixture

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `test-fixtures/simple-task-skill.md`

- [ ] **Step 1: Create plugin.json**

```json
{
  "name": "skill-battlefield",
  "version": "0.1.0",
  "description": "AI tests skills, humans write them. Stress-test AI agent skills across diverse scenarios, get diagnostic evidence and improvement proposals.",
  "author": {
    "name": "Wayne Wang"
  },
  "homepage": "https://github.com/waynewangyuxuan/skill-battlefield",
  "repository": "https://github.com/waynewangyuxuan/skill-battlefield",
  "license": "MIT",
  "keywords": [
    "skill-testing",
    "skill-optimization",
    "evaluation",
    "diagnostics",
    "scenario-generation"
  ]
}
```

Write this to `.claude-plugin/plugin.json`.

- [ ] **Step 2: Create a simple test fixture skill**

This is a deliberately simple skill with known weaknesses for testing the pipeline. It has: a vague trigger, missing negative constraints, a skippable step, and some dead text.

```markdown
---
name: daily-task
description: Use when the user wants to organize their daily tasks
---

# Daily Task Organizer

Help the user organize their daily tasks into a prioritized list.

## Background

Task management has been studied extensively in productivity literature. The Eisenhower Matrix, Getting Things Done (GTD), and time-blocking are all popular approaches. This skill draws on these methodologies to help users organize their day.

## Steps

1. Ask the user what tasks they need to complete today
2. For each task, determine urgency and importance
3. Sort tasks by priority (urgent+important first, then important, then urgent, then neither)
4. Present the prioritized list in a markdown table
5. Ask if the user wants to set time estimates

## Output Format

Present tasks as a markdown table:

| Priority | Task | Urgency | Importance | Time Estimate |
|----------|------|---------|------------|---------------|
| 1 | ... | High | High | ... |

## Notes

- Don't use emojis unless the user asks for them
- Keep the language professional
- If the user provides fewer than 3 tasks, still create the table
```

Write this to `test-fixtures/simple-task-skill.md`.

- [ ] **Step 3: Commit**

```bash
cd /Users/waynewang/skill-battlefield
git add .claude-plugin/plugin.json test-fixtures/simple-task-skill.md
git commit -m "feat: add plugin manifest and test fixture skill"
```

---

### Task 2: Reference Files — Contracts

**Files:**
- Create: `skills/battle/references/contracts.md`

- [ ] **Step 1: Write contracts.md**

This file defines the JSON schemas that all phases use to communicate via the run directory. Each phase reads the previous phase's output and writes its own.

```markdown
# Contracts — Shared Data Schemas

All phases read/write JSON files in the run directory. These schemas define the shape of each file.

## analysis.json (Phase 01 → Phase 02, 03, 04)

Written by 01-analyze. Read by all subsequent phases.

```json
{
  "skill_name": "string — name extracted from frontmatter or first heading",
  "skill_type": "string — one of: prose | checklist | multi-phase",
  "sections": [
    {
      "id": "string — s1, s2, ... sequential",
      "name": "string — heading text or 'Frontmatter' / 'Preamble'",
      "line_range": [number, number],
      "content_hash": "string — first 8 chars of md5 of section text",
      "role": "string — one of: activation | instruction | gate | reference | meta"
    }
  ],
  "intent": "string — one sentence summarizing what the skill does",
  "trigger_conditions": ["string — phrases/keywords that should activate this skill"],
  "negative_triggers": ["string — phrases/keywords that should NOT activate this skill"],
  "constraints": ["string — explicit rules the skill enforces"],
  "gates": ["string — verification checkpoints or hard gates"],
  "estimated_complexity": "string — one of: low | medium | high"
}
```

## scenarios.jsonl (Phase 02 → Phase 03, 04)

Written by 02-generate. Each line is one JSON object:

```json
{
  "id": "string — scenario-001, scenario-002, ...",
  "description": "string — one sentence describing the test scenario",
  "user_message": "string — the exact message the simulated user sends",
  "context": {
    "files_present": ["string — file paths in the simulated project"],
    "recent_error": "string | null — recent error output if relevant",
    "project_type": "string — e.g. typescript, python, rust"
  },
  "expected_behaviors": [
    {
      "check": "string — short identifier like asks_clarifying_question",
      "description": "string — natural language rubric the judge uses"
    }
  ],
  "difficulty": "string — one of: easy | medium | hard | adversarial",
  "dimension": "string — primary dimension tested: D1 | D2 | D3 | D4 | D5 | D6",
  "failure_mode_target": "string — which failure mode this scenario probes: F1-F8 | null",
  "tags": ["string — freeform tags for grouping"]
}
```

## baseline-evals.jsonl / re-eval results (Phase 03, 04)

Written by 03-baseline and by 04-sharpen during re-evaluation. Each line:

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

## iterations/iter-N.json (Phase 04)

One file per sharpening iteration:

```json
{
  "iteration": "number — 1-indexed",
  "target_section": "string — section id from analysis.json",
  "target_section_role": "string — section role",
  "failure_pattern": "string — description of the failure cluster",
  "diff": "string — unified diff of the rewrite",
  "kept": "boolean — was this rewrite accepted",
  "score_before": "number — overall score before this iteration",
  "score_after": "number — overall score after this iteration",
  "regressed_scenarios": ["string — scenario ids that regressed"],
  "reasoning": "string — why kept or discarded"
}
```

## progress.jsonl (Phase 04)

Append-only log, one line per event:

```json
{
  "timestamp": "string — ISO 8601",
  "event": "string — iteration_start | iteration_end | early_stop | rewrite_discarded",
  "iteration": "number",
  "score": "number — current overall score",
  "detail": "string — human-readable summary"
}
```

## spot-check.json (Spot-Check phase)

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
```

Write this to `skills/battle/references/contracts.md`.

- [ ] **Step 2: Commit**

```bash
cd /Users/waynewang/skill-battlefield
git add skills/battle/references/contracts.md
git commit -m "feat: add contracts reference — shared JSON schemas for all phases"
```

---

### Task 3: Reference Files — Taxonomy + Rubric Guide

**Files:**
- Create: `skills/battle/references/taxonomy.md`
- Create: `skills/battle/references/rubric-guide.md`

- [ ] **Step 1: Write taxonomy.md**

Extract the taxonomy from the research synthesis into a compact, phase-readable reference. Include D1-D6 dimensions with sub-criteria, F1-F8 failure modes with fix patterns, and section role definitions.

Source content: copy the "Evaluation Taxonomy" section from the design spec (`docs/specs/2026-06-05-skill-battlefield-design.md` lines 345-369), and enrich with the sub-criteria tables from `research/00-synthesis.md` (the D1-D6 sections with their sub-criterion tables). Keep it under 200 lines — phases only need the structured reference, not the full research narrative.

Structure:

```markdown
# Taxonomy — Evaluation Dimensions & Failure Modes

## Dimensions

### D1: Activation Reliability
[sub-criteria table]

### D2: Execution Compliance
[sub-criteria table]

...

## Failure Modes

| ID | Name | Dimension | Description | Fix Pattern |
...

## Section Roles

| Role | What it covers | Affects dimensions |
...
```

Write to `skills/battle/references/taxonomy.md`.

- [ ] **Step 2: Write rubric-guide.md**

A compact guide for how Phase 02 (scenario generation) should design `expected_behaviors` checks, and how Phase 03 (judge) should evaluate them. Based on Autorubric findings from the research.

```markdown
# Rubric Design Guide

## Principles (from Autorubric, Stanford 2026)

1. **Binary criteria (MET/UNMET)** yield highest inter-rater reliability. Avoid ordinal scales in expected_behaviors checks.

2. **One criterion per evaluation.** The judge evaluates each check in expected_behaviors independently. Never bundle multiple behaviors into one check.

3. **Behavioral anchors, not quality judgments.** Good: "Agent calls Read tool before Edit tool." Bad: "Agent handles the situation well."

4. **Observable actions only.** Each check must reference something the simulator can describe: a tool call, a question asked, an output produced, a step skipped. Not internal reasoning.

5. **Balance difficulty.** Each scenario should have 2-4 checks. At least one should be achievable by a baseline skill (prevents floor effects). At least one should challenge the skill (prevents ceiling effects).

## Writing Good Checks

### Good checks (observable, binary, specific):
- `{"check": "reads_file_before_edit", "description": "Agent uses Read tool on the target file before any Edit call"}`
- `{"check": "asks_which_test", "description": "Agent asks the user to specify which test file or test name"}`
- `{"check": "does_not_modify_unrelated", "description": "Agent does not Edit any file not mentioned in the error output"}`

### Bad checks (vague, subjective, bundled):
- `{"check": "handles_well", "description": "Agent handles the edge case appropriately"}` — "appropriately" is subjective
- `{"check": "good_workflow", "description": "Agent follows a logical debugging workflow"}` — too vague, not binary
- `{"check": "asks_and_reads", "description": "Agent asks clarifying question AND reads the file"}` — bundled, split into two

## For the Judge

- Evaluate each check independently. Do not let one check's result influence another.
- Use temperature 0 for determinism.
- For scores within ±1 of 5 (the pass/fail threshold), run 3 evaluations and take majority vote.
- Provide specific reasoning citing the simulated response for every pass/fail decision.
```

Write to `skills/battle/references/rubric-guide.md`.

- [ ] **Step 3: Commit**

```bash
cd /Users/waynewang/skill-battlefield
git add skills/battle/references/taxonomy.md skills/battle/references/rubric-guide.md
git commit -m "feat: add taxonomy and rubric-guide references"
```

---

### Task 4: Phase 01 — Analyze

**Files:**
- Create: `skills/battle/phases/01-analyze.md`

- [ ] **Step 1: Write 01-analyze.md**

This is the instruction file for the Phase 01 subagent. The subagent receives the path to `skill-original.md` in the run directory and must produce `analysis.json`.

```markdown
# Phase 01 — Analyze

You are a skill analysis agent. Your job is to parse a Claude Code skill file and produce a structured analysis.

## Input

Read the file at: `{run_dir}/skill-original.md`

## Output

Write a JSON file to: `{run_dir}/analysis.json`

The JSON must conform to the analysis.json schema in references/contracts.md.

## How to Analyze

### 1. Extract metadata

- **skill_name**: From YAML frontmatter `name` field. If no frontmatter, use the first `#` heading. If neither, use the filename without extension.
- **skill_type**: Classify as one of:
  - `prose` — Conventions, style guides, reference knowledge. No numbered steps or phases. Short, declarative.
  - `checklist` — Numbered steps or bullet-point procedures. May have verification steps. Task-oriented.
  - `multi-phase` — Multiple named phases or stages with transitions between them. May dispatch subagents. Workflow-oriented.
  - When ambiguous between checklist and multi-phase: if the skill has named phases with distinct inputs/outputs or dispatches subagents, it is multi-phase. If it has a flat numbered list, it is checklist.

### 2. Segment into sections

Walk through the skill file and identify logical sections. A section boundary is:
- A markdown heading (`#`, `##`, `###`, etc.)
- YAML frontmatter (always its own section)
- A clear topic shift in prose (rare — prefer heading-based boundaries)

For each section:
- Assign sequential id: s1, s2, s3, ...
- Record the heading text as `name` (use "Frontmatter" for YAML, "Preamble" for text before first heading)
- Record `line_range` as [first_line, last_line] (1-indexed)
- Compute `content_hash`: take the section text, strip whitespace, take first 8 characters of its md5 hash
- Assign `role` based on content:

| Role | How to identify |
|------|----------------|
| **activation** | Contains trigger conditions, `when_to_use`, description field, "Use when", "Trigger when" |
| **instruction** | Contains steps, commands, behavioral directives ("always", "must", "do X before Y") |
| **gate** | Contains verification checkpoints, "HARD-GATE", "Do NOT proceed until", "verify before continuing" |
| **reference** | Contains examples, templates, code samples, lookup tables, supporting material |
| **meta** | YAML frontmatter, comments, metadata, authorship, version info |

### 3. Extract intent and triggers

- **intent**: One sentence summarizing what the skill does. Derive from description field, first heading, or opening paragraph.
- **trigger_conditions**: Phrases from the description or body that indicate when the skill should activate.
- **negative_triggers**: Phrases that explicitly say when NOT to use the skill. Look for "DO NOT use when", "DO NOT TRIGGER when".
- **constraints**: Explicit rules the skill enforces (e.g., "no fixes without root cause", "one question at a time").
- **gates**: Verification checkpoints (e.g., "verify tests pass before committing", "get user approval before proceeding").

### 4. Estimate complexity

- `low`: < 5 sections, prose or simple checklist, no subagent dispatch
- `medium`: 5-15 sections, checklist with gates, or prose with multiple topics
- `high`: > 15 sections, multi-phase, subagent dispatch, complex state management

## Important

- Output valid JSON only — no markdown wrapping, no commentary.
- Use Bash to write the JSON file: `cat > {run_dir}/analysis.json << 'ENDJSON' ... ENDJSON`
- If the skill file cannot be parsed (binary, empty, not markdown), write an analysis.json with `"skill_type": "unknown"` and empty arrays for sections/triggers.
```

Write to `skills/battle/phases/01-analyze.md`.

- [ ] **Step 2: Test Phase 01 against the test fixture**

Manually verify: read `test-fixtures/simple-task-skill.md`, mentally trace the analysis rules, and confirm the expected output would be:
- `skill_name`: "daily-task"
- `skill_type`: "checklist"
- Sections: Frontmatter (meta), heading "Daily Task Organizer" (activation/instruction), "Background" (reference), "Steps" (instruction), "Output Format" (reference), "Notes" (instruction)
- `trigger_conditions`: ["organize daily tasks"]
- `negative_triggers`: [] (none specified — this is a known weakness)
- `gates`: [] (none — another known weakness)
- `estimated_complexity`: "low"

If the expected output looks correct against the contract schema, proceed.

- [ ] **Step 3: Commit**

```bash
cd /Users/waynewang/skill-battlefield
git add skills/battle/phases/01-analyze.md
git commit -m "feat: add Phase 01 — analyze (skill parser)"
```

---

### Task 5: Phase 02 — Generate Scenarios

**Files:**
- Create: `skills/battle/phases/02-generate.md`

- [ ] **Step 1: Write 02-generate.md**

Instruction file for the Phase 02 subagent. Reads `analysis.json`, writes `scenarios.jsonl`.

```markdown
# Phase 02 — Generate Test Scenarios

You are a scenario generation agent. Your job is to create diverse, realistic test scenarios that probe whether a skill works correctly across a range of situations.

## Input

Read: `{run_dir}/analysis.json`

Also read the taxonomy reference at `{skill_dir}/references/taxonomy.md` for the D1-D6 dimensions and F1-F8 failure modes.

Also read the rubric guide at `{skill_dir}/references/rubric-guide.md` for how to write good expected_behaviors checks.

## Output

Write: `{run_dir}/scenarios.jsonl` — one JSON object per line, conforming to the scenario schema in references/contracts.md.

Generate `{num_scenarios}` scenarios total (passed as config, default 20).

## Generation Strategy

### Step 1: Understand the skill

From analysis.json, extract:
- What the skill claims to do (intent)
- When it should trigger (trigger_conditions)
- When it should NOT trigger (negative_triggers)
- What rules it enforces (constraints)
- What verification points it has (gates)
- Its structural type (prose/checklist/multi-phase)

### Step 2: Distribute across difficulty tiers

| Tier | Percentage | Description |
|------|-----------|-------------|
| easy | 25% | Clear, unambiguous happy path. A reasonable skill should handle this. |
| medium | 35% | Typical real-world usage. Some ambiguity, incomplete info, or multiple valid approaches. |
| hard | 25% | Edge cases: unusual inputs, conflicting instructions, boundary conditions. |
| adversarial | 15% | Deliberately trying to break the skill: prompt injection, off-topic requests, requests that look like triggers but aren't. |

Round to whole numbers. For 20 scenarios: 5 easy, 7 medium, 5 hard, 3 adversarial.

### Step 3: Cover all 6 dimensions

Ensure at least 2 scenarios per dimension (D1-D6). Assign each scenario a primary dimension. Distribution:
- D1 (Activation): at least 2 — test true positive and false positive triggers
- D2 (Execution): at least 3 — test step-following, especially middle steps
- D3 (Behavioral): at least 3 — test edge cases and ambiguous inputs
- D4 (Clarity): at least 2 — test if the instructions are clear enough to follow
- D5 (Architecture): at least 2 — test context efficiency, scope boundaries
- D6 (Evolvability): at least 2 — test gotcha scenarios, failure recovery

### Step 4: Target specific failure modes

Where possible, assign a `failure_mode_target` to scenarios:
- F1 (Never activates): scenario where the skill should trigger but the description is weak
- F2 (False positive): scenario where the skill should NOT trigger
- F3 (Skips steps): scenario that requires following a middle step
- F4 (Letter vs spirit): scenario where rigid rule-following would miss the point
- F5 (Pink elephant): scenario involving a negative instruction in the skill
- F6 (Context bloat): scenario testing if unnecessary skill content causes confusion
- F7 (Instruction erosion): scenario testing if later instructions override earlier ones
- F8 (Gotcha amnesia): scenario testing a common mistake the skill should prevent

### Step 5: Design expected_behaviors

For each scenario, write 2-4 checks following the rubric guide:
- Binary (pass/fail), observable, specific
- At least one "should be easy" check
- At least one "should be challenging" check
- Use the check naming convention: `verb_object` (e.g., `asks_clarifying_question`, `reads_file_before_edit`)

### Step 6: Self-critique

After generating all scenarios, review them as a batch:
- Are any two scenarios testing the same thing? → merge or replace one
- Is any dimension underrepresented? → add a scenario
- Are the adversarial scenarios actually challenging? → strengthen if too easy
- Do the context fields make sense? → ensure files_present and project_type are realistic

## Context Field Design

The `context` field simulates the project state. Make it realistic:
- `files_present`: plausible file paths for the `project_type`
- `recent_error`: include for debugging scenarios, null otherwise
- `project_type`: match to the scenario's domain

## Important

- Write valid JSONL — one JSON object per line, no trailing commas, no wrapping.
- Use Bash to write: `cat > {run_dir}/scenarios.jsonl << 'ENDJSONL' ... ENDJSONL`
- Scenario ids must be sequential: scenario-001, scenario-002, ...
- Every scenario must have at least 2 expected_behaviors checks.
```

Write to `skills/battle/phases/02-generate.md`.

- [ ] **Step 2: Commit**

```bash
cd /Users/waynewang/skill-battlefield
git add skills/battle/phases/02-generate.md
git commit -m "feat: add Phase 02 — generate scenarios"
```

---

### Task 6: Phase 03 — Baseline Eval

**Files:**
- Create: `skills/battle/phases/03-baseline.md`

- [ ] **Step 1: Write 03-baseline.md**

Instruction file for the baseline evaluation subagent. Two-step process: simulate, then judge. The key design constraint is **information separation** to prevent contamination.

```markdown
# Phase 03 — Baseline Evaluation

You are an evaluation agent. Your job is to evaluate how well a skill performs on each test scenario using a two-step simulate-then-judge process.

## Input

Read:
- `{run_dir}/skill-original.md` (or `{run_dir}/skill-current.md` if re-evaluating a rewrite)
- `{run_dir}/scenarios.jsonl`
- `{skill_dir}/references/rubric-guide.md`

## Output

Write: `{run_dir}/baseline-evals.jsonl` — one eval result per line, conforming to the eval-result schema in references/contracts.md.

## Two-Step Evaluation Process

For each scenario, you perform TWO separate reasoning steps. These MUST be kept separate to prevent rubric contamination.

### Step 1: Simulate

You role-play as a Claude Code agent that has received the skill and the user's message.

**You receive:**
- The full skill text
- The scenario's `user_message`
- The scenario's `context` (files present, project type, recent error)

**You do NOT receive:**
- The scenario's `expected_behaviors` — you must NOT look at them during simulation

**Your task:** Describe in detail what the agent would do:
- Which tools it would call, in what order
- What questions it would ask the user
- What output it would produce
- What steps it would follow from the skill
- What it would skip or miss
- How it would handle ambiguity

Write a detailed paragraph (100-200 words) as the `simulated_response`.

**Be realistic, not optimistic.** A real agent might:
- Skip steps that seem redundant
- Miss nuance in instructions
- Default to its base behavior when skill instructions are vague
- Get distracted by irrelevant context
- Follow the letter of instructions while missing the spirit

### Step 2: Judge

Now evaluate the simulated response against the expected behaviors.

**You receive:**
- The `simulated_response` from Step 1
- The scenario's `expected_behaviors` (checks + descriptions)

**You do NOT receive:**
- The skill text — judge purely based on what the agent did, not what the skill says

**For each check:**
1. Read the check description carefully
2. Look for specific evidence in the simulated response that the behavior occurred or did not occur
3. Mark as `pass` (true) or `fail` (false)
4. Write a one-sentence `reasoning` citing the specific part of the simulated response

**Scoring:**
- Count passed checks: `passed / total × 10` gives the base score
- Adjust ±1 for quality factors: was the agent thorough? efficient? appropriate in tone?
- Final score: integer 1-10

**Dimension scores:**
- Score the primary dimension (from scenario.dimension) using the same 1-10 scale
- Leave other dimensions as null unless the simulated response clearly touches them

### Consistency Measures

- When scoring, imagine you are scoring 100 similar responses. Be consistent.
- For scores of 4, 5, or 6 (near the pass/fail boundary of 5): repeat the judge step 3 times with slight reframing and take the majority result.

## Parallelism

You may evaluate multiple scenarios in parallel using the Agent tool to dispatch sub-evaluators. Each sub-evaluator handles one scenario's simulate + judge cycle.

When parallelizing:
- Dispatch up to 5 scenarios at a time
- Each sub-evaluator writes its result as a single JSONL line
- Collect all results and write the final baseline-evals.jsonl

If not parallelizing, evaluate scenarios sequentially and append each result to the file.

## Important

- NEVER look at expected_behaviors during simulation. This is the most critical rule.
- Write valid JSONL — one JSON object per line.
- Use Bash to write: append each line to the file with `echo '...' >> {run_dir}/baseline-evals.jsonl`
- The `skill_version` field should be "v0" for baseline evaluation.
```

Write to `skills/battle/phases/03-baseline.md`.

- [ ] **Step 2: Commit**

```bash
cd /Users/waynewang/skill-battlefield
git add skills/battle/phases/03-baseline.md
git commit -m "feat: add Phase 03 — baseline eval (simulator + judge)"
```

---

### Task 7: Phase 04 — Sharpen

**Files:**
- Create: `skills/battle/phases/04-sharpen.md`

- [ ] **Step 1: Write 04-sharpen.md**

This is the most complex phase. A single subagent runs the full iteration loop internally.

```markdown
# Phase 04 — Sharpen

You are a skill sharpening agent. Your job is to iteratively improve a skill by analyzing failure patterns, proposing targeted rewrites, and verifying improvements.

## Input

Read:
- `{run_dir}/skill-original.md`
- `{run_dir}/analysis.json`
- `{run_dir}/scenarios.jsonl`
- `{run_dir}/baseline-evals.jsonl`
- `{skill_dir}/references/taxonomy.md`
- `{skill_dir}/references/rubric-guide.md`

Config (from orchestrator):
- `max_iterations`: default 5
- `early_stop_delta`: default 0.2
- `max_drift`: default 0.4 (40% of original skill length)

## Output

Write:
- `{run_dir}/iterations/iter-N.json` — one per iteration
- `{run_dir}/skill-current.md` — best version so far (updated each iteration)
- `{run_dir}/progress.jsonl` — append-only event log

## Setup

1. Copy `skill-original.md` to `skill-current.md` as the starting point.
2. Calculate baseline overall score: average of all scores in baseline-evals.jsonl.
3. Record original skill length (character count) for drift tracking.
4. Create `iterations/` directory.
5. Log to progress.jsonl: `{"event": "sharpen_start", "iteration": 0, "score": <baseline_score>}`

## Iteration Loop

For each iteration (1 to max_iterations):

### 4.1: Analyze failures

Read the current evaluation results (baseline-evals.jsonl for iter 1, or the most recent re-eval results for later iterations).

Find all failing checks (pass: false). Group them by which section of the skill likely caused the failure:
- Use analysis.json to map each failure to a section
- A failure maps to a section if:
  - The section contains instructions relevant to the failed check
  - The section's role matches the type of failure (e.g., activation failures → activation sections, step-skipping → instruction sections)

Produce a failure cluster: `{"section_id": "s3", "failure_count": 4, "pattern": "Agent skips step 2 when input is ambiguous"}`

### 4.2: Select target

Pick the section with the highest failure count. Ties: prefer instruction > activation > gate > reference > meta.

If no section has failures, or all failures are in sections already rewritten in previous iterations with no improvement: trigger early stop.

### 4.3: Rewrite

Read the target section from skill-current.md. Propose a targeted rewrite:

**Rules:**
- Change ONLY the target section. Do not touch other sections.
- The rewrite should address the specific failure pattern identified.
- Prefer additions over deletions (adding clarity, adding a gate, adding an example).
- Use positive framing ("do X") not negative ("don't do Y") — Pink Elephant principle.
- Keep the section's original intent. Do not change what the skill does, only how clearly it communicates.

**Output the rewrite as a unified diff:**
```
- old line
+ new line
```

Apply the diff to skill-current.md, producing a candidate version.

### 4.4: Re-evaluate

Run the FULL evaluation process (same as Phase 03) on the candidate skill version against ALL scenarios. Not just the failing ones — we need regression detection.

Use the same two-step simulate-then-judge process. Set `skill_version` to "v{iteration}".

Calculate the new overall score.

### 4.5: Decide

Compare new score to previous score:

**KEEP** if:
- New overall score > previous overall score
- No easy-tier scenarios regressed (went from pass to fail)

**DISCARD** if:
- New overall score ≤ previous overall score
- OR any easy-tier scenario regressed

On DISCARD:
- Revert skill-current.md to the previous version
- Log the failed attempt with reasoning
- If this is retry attempt 1: try a DIFFERENT rewrite approach for the same section, passing the failed diff + judge reasoning as context
- If this is retry attempt 2: mark "no improvement found for section {id}", move to next iteration targeting the next-highest failure section
- Max 2 retries per iteration

On KEEP:
- Update skill-current.md with the new version
- Update current_evals to the new evaluation results

### 4.6: Log

Write `iterations/iter-N.json` conforming to the iter-result schema in contracts.md.

Append to progress.jsonl:
```json
{"timestamp": "<ISO 8601>", "event": "iteration_end", "iteration": N, "score": <new_score>, "detail": "Section s3 rewritten, score 6.2 → 7.0, KEPT"}
```

### 4.7: Check early stop

Stop the loop if ANY of these conditions are met:
- Score delta < `early_stop_delta` for 2 consecutive iterations (including discarded rewrites)
- Cumulative character diff between skill-original.md and skill-current.md exceeds `max_drift` × original length
- All sections have been attempted with no improvement

On early stop, log:
```json
{"timestamp": "<ISO 8601>", "event": "early_stop", "iteration": N, "score": <current_score>, "detail": "Reason for stopping"}
```

## Important

- The re-evaluation in step 4.4 follows the EXACT same rules as Phase 03 (simulator does not see expected_behaviors, judge does not see skill text).
- Track cumulative diff carefully — the 40% drift guard prevents the skill from losing its identity.
- skill-current.md must always be a complete, valid skill file — not a diff or partial.
- Write valid JSON for iter-N.json and valid JSONL for progress.jsonl.
```

Write to `skills/battle/phases/04-sharpen.md`.

- [ ] **Step 2: Commit**

```bash
cd /Users/waynewang/skill-battlefield
git add skills/battle/phases/04-sharpen.md
git commit -m "feat: add Phase 04 — sharpen (iteration loop)"
```

---

### Task 8: Phase 05 — Report

**Files:**
- Create: `skills/battle/phases/05-report.md`

- [ ] **Step 1: Write 05-report.md**

```markdown
# Phase 05 — Report

You are a reporting agent. Your job is to synthesize all battle artifacts into a human-readable report and produce the final sharpened skill file.

## Input

Read all files in `{run_dir}/`:
- `analysis.json`
- `scenarios.jsonl`
- `baseline-evals.jsonl`
- `iterations/iter-*.json` (all iteration files)
- `progress.jsonl`
- `spot-check.json` (if exists)
- `skill-original.md`
- `skill-current.md`

Also read `{skill_dir}/references/taxonomy.md` for dimension names.

## Output

Write:
- `{run_dir}/report.md`
- `{run_dir}/skill-sharpened.md` (copy of skill-current.md, renamed for clarity)

## Report Structure

Generate report.md with exactly this structure:

```markdown
# Battle Report: {skill_name}

**Date:** {timestamp}
**Skill:** {original_path}
**Scenarios:** {count} ({easy}/{medium}/{hard}/{adversarial})
**Iterations:** {count} ({kept} kept, {discarded} discarded)

## Score Card

| Dimension | Before | After | Delta |
|-----------|--------|-------|-------|
| D1: Activation Reliability | X.X | X.X | +/-X.X |
| D2: Execution Compliance | X.X | X.X | +/-X.X |
| D3: Behavioral Alignment | X.X | X.X | +/-X.X |
| D4: Instruction Clarity | X.X | X.X | +/-X.X |
| D5: Architecture & Composability | X.X | X.X | +/-X.X |
| D6: Evolvability | X.X | X.X | +/-X.X |
| **Overall** | **X.X** | **X.X** | **+/-X.X** |

Calculate dimension scores by averaging all eval results for scenarios tagged with that dimension, both before (baseline) and after (final iteration). Overall is the mean of all scenario scores.

## Top Findings

List the 3-5 most significant findings. Each finding should be one sentence with:
- What was found
- What dimension/failure mode it relates to
- What was done about it (if anything)
- Quantified impact

## Iteration Log

| Iter | Section | Role | Kept | Before | After | Summary |
|------|---------|------|------|--------|-------|---------|
| ... | | | | | | |

## Spot-Check Validation

If spot-check.json exists, include:

| Scenario | Simulated | Real | Delta | Flag |
|----------|-----------|------|-------|------|
| ... | | | | |

And a one-sentence summary of simulation reliability.

If no spot-check was performed, write: "Spot-check validation was not performed in this run."

## Failure Map

| Scenario | Difficulty | Dimension | Failure Mode | Before | After | Status |
|----------|-----------|-----------|--------------|--------|-------|--------|
| ... | | | | | | FIXED / IMPROVED / OPEN |

Status:
- FIXED: was failing, now passing
- IMPROVED: score improved but still below threshold
- OPEN: no improvement or new failure

## Remaining Weaknesses

For each OPEN failure, write:
- The scenario description
- Why it fails
- Suggested fix (what a human skill author should consider changing)

## How to Apply

```
Review the proposed changes:
  diff {run_dir}/skill-original.md {run_dir}/skill-sharpened.md

To adopt all changes:
  cp {run_dir}/skill-sharpened.md {original_path}

To adopt selectively:
  Review each iteration in iterations/ and copy specific diffs.
```
```

## Generating skill-sharpened.md

Copy `skill-current.md` to `skill-sharpened.md`. No modifications — it is the exact output of the sharpening loop.

## Important

- All numbers in tables should be formatted to one decimal place.
- Delta should show + or - sign explicitly.
- Findings should be actionable, not vague.
- Write report.md using Bash: `cat > {run_dir}/report.md << 'ENDREPORT' ... ENDREPORT`
- Copy skill-current to skill-sharpened: `cp {run_dir}/skill-current.md {run_dir}/skill-sharpened.md`
```

Write to `skills/battle/phases/05-report.md`.

- [ ] **Step 2: Commit**

```bash
cd /Users/waynewang/skill-battlefield
git add skills/battle/phases/05-report.md
git commit -m "feat: add Phase 05 — report generation"
```

---

### Task 9: SKILL.md — Orchestrator

**Files:**
- Create: `skills/battle/SKILL.md`

- [ ] **Step 1: Write SKILL.md**

This is the entrypoint for `/battle`. It parses arguments, creates the run directory, and dispatches one subagent per phase sequentially.

```markdown
---
name: battle
description: "Stress-test a Claude Code skill across diverse scenarios. Generates test scenarios (easy/medium/hard/adversarial), evaluates the skill via simulated execution, proposes targeted improvements, and produces an evidence report. Use when: user wants to test a skill, evaluate a skill, sharpen a skill, battle-test a skill, find weaknesses in a skill, or optimize a skill. Use when user says 'battle', 'test this skill', 'evaluate my skill', 'sharpen', 'find weaknesses'. DO NOT use for: writing new skills from scratch, general code review, or non-skill files."
user-invocable: true
argument-hint: "<skill-path> [--scenarios 20] [--iterations 5]"
---

# /battle — Skill Battlefield

Stress-test a skill file through scenario generation, simulated evaluation, targeted rewriting, and evidence reporting.

## Argument Parsing

Parse `$ARGUMENTS`:
- First positional argument: `skill_path` (required). Resolve to absolute path. If it starts with `~`, expand it.
- `--scenarios N`: number of scenarios to generate (default: 20)
- `--iterations N`: max sharpening iterations (default: 5)

If no skill_path is provided, print usage and stop:
```
Usage: /battle <skill-path> [--scenarios 20] [--iterations 5]
Example: /battle ~/.claude/skills/my-skill/SKILL.md
```

## Validation

1. Check that skill_path exists and is a readable file. If not, print error and stop.
2. Check that the file looks like a skill (contains markdown). If it's binary or empty, print error and stop.

## Setup

1. Extract skill name from the filename (e.g., `SKILL.md` in `my-skill/` → `my-skill`).
2. Create timestamp: `date +%Y%m%d-%H%M%S`
3. Create run directory: `~/.skill-battlefield/runs/{skill_name}/{timestamp}/`
4. Create `iterations/` subdirectory.
5. Copy the skill file to `{run_dir}/skill-original.md`.

Print: `Starting battle for {skill_name}. Run directory: {run_dir}`

## Phase Execution

Execute phases sequentially. Each phase is dispatched as a subagent via the Agent tool. The subagent reads its phase instruction file and the relevant reference files.

**Important for all subagent dispatches:**
- Tell the subagent the `run_dir` path so it knows where to read/write.
- Tell the subagent the `skill_dir` path (the directory containing this SKILL.md) so it can read reference files.
- Wait for each subagent to complete before dispatching the next.
- After each phase, verify the expected output file exists. If it doesn't, print an error and stop.

### Phase 01 — Analyze

Dispatch a subagent. In the prompt, include:
- The full content of `phases/01-analyze.md` (read it first)
- `run_dir = {run_dir}`
- `skill_dir = {skill_dir}`

After completion, verify `{run_dir}/analysis.json` exists.

Print: `Phase 01 complete. Skill type: {skill_type}, {section_count} sections identified.`

### Phase 02 — Generate Scenarios

Dispatch a subagent. In the prompt, include:
- The full content of `phases/02-generate.md`
- The full content of `references/taxonomy.md`
- The full content of `references/rubric-guide.md`
- `run_dir = {run_dir}`
- `num_scenarios = {scenarios_arg}`

After completion, verify `{run_dir}/scenarios.jsonl` exists and count the lines.

Print: `Phase 02 complete. {N} scenarios generated.`

### Phase 03 — Baseline Evaluation

Dispatch a subagent. In the prompt, include:
- The full content of `phases/03-baseline.md`
- The full content of `references/rubric-guide.md`
- `run_dir = {run_dir}`

After completion, verify `{run_dir}/baseline-evals.jsonl` exists.

Calculate and print baseline score: read baseline-evals.jsonl, average all `score` fields.

Print: `Phase 03 complete. Baseline score: {score}/10`

### Phase 04 — Sharpen

Dispatch a subagent. In the prompt, include:
- The full content of `phases/04-sharpen.md`
- The full content of `references/taxonomy.md`
- The full content of `references/rubric-guide.md`
- The full content of `references/contracts.md`
- `run_dir = {run_dir}`
- `max_iterations = {iterations_arg}`
- `early_stop_delta = 0.2`
- `max_drift = 0.4`

After completion, verify `{run_dir}/skill-current.md` exists.

Read progress.jsonl and print a summary:
```
Phase 04 complete. {N} iterations run.
Score: {baseline} → {final} ({delta})
```

### Spot-Check Validation

Read scenarios.jsonl and the final evals. Select 3-5 scenarios for real execution:
- 1x highest simulated score
- 1x lowest simulated score
- 1-2x scores closest to 5.0 (threshold)
- 1x adversarial difficulty

For each selected scenario, run:
```bash
claude -p "You have been given this skill to follow:

$(cat {run_dir}/skill-current.md)

A user sends you this message: {scenario.user_message}

The project context is: {scenario.context}

Respond as you would as a Claude Code agent following the skill."
```

Judge each real response using the same expected_behaviors from the scenario.

Write results to `{run_dir}/spot-check.json` conforming to the contract schema.

Print: `Spot-check complete. Avg divergence: {avg_delta}`

### Phase 05 — Report

Dispatch a subagent. In the prompt, include:
- The full content of `phases/05-report.md`
- The full content of `references/taxonomy.md`
- `run_dir = {run_dir}`
- `original_path = {skill_path}` (the original path the user provided)

After completion, verify `{run_dir}/report.md` and `{run_dir}/skill-sharpened.md` exist.

## Summary

Print the final summary to the user:

```
=== Battle Complete ===

Skill: {skill_name}
Score: {baseline_score} → {final_score} ({delta})

Top findings:
1. {finding_1}
2. {finding_2}
3. {finding_3}

Report: {run_dir}/report.md
Sharpened skill: {run_dir}/skill-sharpened.md
Diff: diff {run_dir}/skill-original.md {run_dir}/skill-sharpened.md
```
```

Write to `skills/battle/SKILL.md`.

- [ ] **Step 2: Commit**

```bash
cd /Users/waynewang/skill-battlefield
git add skills/battle/SKILL.md
git commit -m "feat: add /battle orchestrator SKILL.md"
```

---

### Task 10: End-to-End Test

**Files:**
- No new files — testing existing files against the test fixture.

- [ ] **Step 1: Install the plugin locally**

Register the plugin with Claude Code so `/battle` becomes available:

```bash
claude plugins add /Users/waynewang/skill-battlefield
```

Verify with `claude plugins list` that `skill-battlefield` appears.

- [ ] **Step 2: Run /battle against the test fixture**

```bash
/battle /Users/waynewang/skill-battlefield/test-fixtures/simple-task-skill.md --scenarios 5 --iterations 2
```

Use a small run (5 scenarios, 2 iterations) for the first test to keep cost and time low.

- [ ] **Step 3: Verify outputs**

Check that all expected files exist in `~/.skill-battlefield/runs/simple-task-skill/<timestamp>/`:

```bash
ls -la ~/.skill-battlefield/runs/simple-task-skill/*/
```

Expected files:
- `analysis.json` — should show skill_type: "checklist"
- `scenarios.jsonl` — should have 5 lines
- `baseline-evals.jsonl` — should have 5 lines
- `iterations/` — should have iter-1.json, iter-2.json (or fewer if early stop)
- `progress.jsonl` — should have events
- `spot-check.json` — should have 3 entries
- `report.md` — should follow the report structure
- `skill-sharpened.md` — should be a valid skill file
- `skill-original.md` — should match the test fixture

- [ ] **Step 4: Review the report**

Read `report.md` and verify:
- Score Card has all 6 dimensions
- Findings are actionable and reference specific sections
- Failure Map shows specific scenarios
- Remaining Weaknesses section is present
- Diff between original and sharpened shows reasonable changes

- [ ] **Step 5: Fix any issues found**

If any phase failed or produced malformed output, trace back to the specific phase file and fix the instructions. Re-run the test after each fix.

- [ ] **Step 6: Commit any fixes**

```bash
cd /Users/waynewang/skill-battlefield
git add -A
git commit -m "fix: address issues found during e2e test"
```

- [ ] **Step 7: Full test run**

Once the small run works, do a full test:

```bash
/battle /Users/waynewang/skill-battlefield/test-fixtures/simple-task-skill.md --scenarios 20 --iterations 5
```

Verify the report, review the sharpened skill, confirm the pipeline runs end-to-end.

- [ ] **Step 8: Self-battle (ultimate dogfood)**

Run `/battle` against our own SKILL.md:

```bash
/battle /Users/waynewang/skill-battlefield/skills/battle/SKILL.md --scenarios 10 --iterations 3
```

This tests whether our skill-testing skill can test itself. Review the report:
- Does it find real weaknesses in our orchestrator?
- Are the generated scenarios realistic for a meta-skill?
- Do the rewrite proposals make sense?

If the self-battle report reveals issues, fix them and re-run. This is the strongest validation — if /battle can improve itself, it works.

- [ ] **Step 9: Push to remote**

```bash
cd /Users/waynewang/skill-battlefield
git push origin main
```
