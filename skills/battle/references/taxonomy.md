# Taxonomy — Evaluation Dimensions, Failure Modes, Section Roles

## Dimensions

### D1: Activation Reliability
Does the skill trigger when it should, and NOT trigger when it shouldn't?

| Sub-criterion | What to measure |
|---|---|
| True positive rate | % of relevant scenarios where skill activates |
| False positive rate | % of irrelevant scenarios where skill incorrectly activates |
| Description quality | Directive language, positive/negative triggers, front-loaded intent |
| Trigger specificity | when_to_use, paths, file-pattern scoping |

### D2: Execution Compliance
Once loaded, does the agent actually follow the skill's steps?

| Sub-criterion | What to measure |
|---|---|
| Step completion rate | % of prescribed steps actually executed |
| Step ordering | Were steps followed in correct sequence? |
| Gate adherence | Did the agent stop at verification gates? |
| Skip detection | Which steps get skipped most? (especially middle steps) |

### D3: Behavioral Alignment
Does the skill produce correct behavior on edge cases, adversarial inputs, and ambiguous scenarios?

| Sub-criterion | What to measure |
|---|---|
| Edge case handling | Score across easy/medium/hard/adversarial tiers |
| Safety boundary respect | Does it refuse/escalate when appropriate? |
| Ambiguity handling | Does it ask for clarification vs. guess? |
| Regression resistance | Easy cases still pass after optimizing for hard ones? |

### D4: Instruction Clarity
How well does the skill's text communicate intent to the LLM?

| Sub-criterion | What to measure |
|---|---|
| Positive framing | % of instructions using "do X" vs "don't do Y" |
| Structure | Headers, bullets, code blocks vs. prose paragraphs |
| Conciseness | Token count; information density; no restating defaults |
| Non-default knowledge ratio | % of content that pushes Claude beyond default behavior |
| Example quality | Few-shot examples present and correctly formatted? |
| Gotchas section | Present? Built from real failures? |

### D5: Architecture & Composability
Is the skill well-structured as a software artifact?

| Sub-criterion | What to measure |
|---|---|
| Scope focus | One skill, one job? Or mega-skill? |
| Context efficiency | SKILL.md under 500 lines? Progressive disclosure? |
| Context utilization | Behavioral ablation: remove section, score unchanged = dead text |
| Modularity | Can be composed with other skills without conflict? |
| File organization | SKILL.md + references/ + examples/ + scripts/? |

### D6: Evolvability
How well does the skill support iteration and improvement?

| Sub-criterion | What to measure |
|---|---|
| Version tracking | Changelog, semver, diff history? |
| Failure feedback loop | Gotchas updated from real failures? |
| Optimization headroom | Current score vs. theoretical max |
| Drift resistance | Does optimization preserve original intent? |

---

## Failure Modes

| ID | Name | Dimension | Description | Fix Pattern |
|---|---|---|---|---|
| F1 | Never activates | D1 | Skill doesn't trigger on valid input | Directive description, negative triggers |
| F2 | False positive | D1 | Activates on wrong input | DO NOT TRIGGER conditions, paths scoping |
| F3 | Skips middle steps | D2 | Agent loads skill but skips steps | Hard gates, verification steps |
| F4 | Letter not spirit | D3 | Follows rules but misses intent | Context over rigid rules |
| F5 | Pink elephant | D4 | Negative instructions backfire | Positive reframing |
| F6 | Context bloat | D5 | Skill too long, gets compacted | Progressive disclosure, split sub-skills |
| F7 | Instruction erosion | D2+D4 | Later instructions override earlier ones | Reduce to essentials only |
| F8 | Gotcha amnesia | D6 | Known failure patterns not documented | Living gotchas section |

---

## Section Roles

| Role | What it covers | Primary dimensions affected |
|---|---|---|
| activation | description, when_to_use, trigger conditions | D1 |
| instruction | steps, behavioral directives, commands | D2, D3 |
| gate | verification checkpoints, hard gates | D2 |
| reference | examples, templates, supporting material | D4, D5 |
| meta | frontmatter, metadata, comments | D5 |
