# Skill Authoring Notes

## Phase 01 — analyze.md (2026-06-05)

### Reference skills studied
- **Robin executor pattern**: Clear Input/Output table at top, numbered procedural phases, file-path contracts. Applied the same I/O table + sequential numbered steps structure.
- **Anthropic skill-creator**: Pushy directive style ("Read the file" not "You should read the file"). Explicit classification tables with "first match wins" semantics.

### Principles applied
- **Positive framing**: All instructions use "do X" form. No "don't do Y" phrasing.
- **Structure for scanning**: Tables for classification rules (roles, types, complexity, edge cases). No prose paragraphs explaining decision logic.
- **Only non-default knowledge**: Omitted general markdown parsing advice. Focused on the specific role taxonomy and type classification rules.
- **Hard gates**: Output checklist at the end with concrete validation (python3 JSON check).
- **Under 150 lines**: Final file is ~100 lines.
- **Reference, don't inline**: Schema referenced via contracts.md, not duplicated.

### D1-D6 self-assessment
- **D1 (Activation)**: N/A — this is a phase instruction, not a user-facing skill.
- **D2 (Execution Compliance)**: Strong. Seven sequential steps, each producing concrete output. No skippable middle steps — step 2 (edge cases) is a hard branch that either exits early or continues. The output checklist catches missed fields.
- **D3 (Behavioral Alignment)**: Good. Edge cases table covers empty, binary, massive, no-frontmatter, no-headings. The "first matching rule" tables prevent ambiguity in classification.
- **D4 (Instruction Clarity)**: Strong. Directive voice throughout. Tables over prose. Every classification rule is a concrete string match, not vague guidance.
- **D5 (Architecture)**: Good. References contracts.md for schema. Lean at ~100 lines. Single responsibility (parse + classify).
- **D6 (Evolvability)**: Moderate. Role and type tables are easy to extend. Adding a new role = one table row. No gotchas section yet (none observed — this is the first iteration).

## Phase 02 — generate.md (2026-06-05)

### Patterns applied
- **Robin executor pattern**: I/O table at top, 8 numbered steps, output checklist at bottom.
- **Reference, don't inline**: Points to taxonomy.md for D1-D6 and F1-F8 definitions, rubric-guide.md for check design rules. No duplication of those tables.
- **Positive framing**: "Cover all 6 dimensions" not "Don't leave dimensions uncovered." "Include at least one achievable check" not "Don't make all checks hard."
- **Structure for scanning**: Distribution table for difficulty tiers, dimension-to-tier mapping table, context field guidance table. No prose blocks.
- **Under 150 lines**: Final file is ~100 lines.

### Design decisions
- **Difficulty tier percentages** (25/35/25/15): Weighted toward medium because most real-world usage is typical, but adversarial tier at 15% ensures stress-testing without drowning the results in failure cases.
- **Dimension-to-tier mapping**: Advisory, not enforced. Helps the subagent distribute dimensions naturally rather than randomly assigning.
- **Self-critique step** (step 7): Added as an explicit step before writing, because LLMs tend to generate redundant scenarios if not prompted to review. This is a lightweight alternative to a separate validation phase.
- **Inline validation script**: The python3 validator checks line count, ID sequencing, minimum checks per scenario, and dimension coverage — the four most likely failure modes of generation.

## Phase 03 — baseline.md (2026-06-05)

### Core design constraint: information separation
- This is the most nuanced phase because it requires strict firewalling between simulator and judge. The simulator sees the skill + user message but never the expected_behaviors. The judge sees the simulated response + expected_behaviors but never the skill text. Without this separation, the eval is meaningless — the simulator would optimize for the rubric, and the judge would infer pass criteria from the skill instead of evaluating the response on its own merits.
- Marked with "HARD GATE" labels and bold warnings. This is stronger language than any other phase uses, because the consequence of violation is silent corruption (results look valid but are inflated).

### Patterns applied
- **Robin executor pattern**: I/O table, numbered steps, output checklist.
- **Two-step decomposition**: Simulate and judge are explicitly separated sub-steps (2A and 2B), not just described in prose. Horizontal rules visually separate them.
- **Autorubric findings**: Binary per-criterion checks, majority vote for borderlines (scores 4-6), temperature 0 for judge determinism. These come from research on LLM-as-judge reliability.
- **Echo append over heredoc**: For JSONL output, echo-append is safer than a single heredoc because it avoids shell escaping issues with JSON containing quotes and special characters.

### Design decisions
- **Parallelism cap at 5**: Enough to speed up a 20-scenario run (4 batches) without overwhelming context windows. Each sub-evaluator is self-contained (one scenario's full cycle).
- **Realism guidance bullets**: Four specific examples of how real agents deviate from ideal behavior. Without these, simulators tend toward optimistic "I would do everything perfectly" responses that inflate scores.
- **Score adjustment of +/-1**: Allows the judge to account for quality factors not captured by binary checks (e.g., depth of explanation, specificity of tool calls) without making scoring subjective. Clamped to 1-10.
- **Dimension scoring strategy**: Only score the primary dimension to avoid noise. Secondary dimensions scored only when there's clear evidence — prevents the judge from speculating.

### D1-D6 self-assessment
- **D2 (Execution Compliance)**: Strong. Four numbered steps with clear sequencing. The two HARD GATE markers are the most critical gates in the entire project. Validation script checks all schema fields.
- **D3 (Behavioral Alignment)**: Strong. Realism guidance prevents optimistic simulation. Borderline retry mechanism prevents noisy scores near the pass/fail boundary.
- **D4 (Instruction Clarity)**: Strong. Information boundaries spelled out twice each (what IS received, what is NOT received). No ambiguity about which data flows where.
- **D5 (Architecture)**: Good. References rubric-guide.md and contracts.md. ~90 lines. Single responsibility (evaluate scenarios).
- **D6 (Evolvability)**: Good. Adding new realism guidance = one bullet. Changing the parallelism cap = one number. The simulate/judge structure is extensible to multi-round evaluation if needed later.

## Phase 04 — sharpen.md (2026-06-05)

### Core design constraint: isolated variable per iteration
- Each iteration rewrites exactly one section. This makes score changes attributable — if the score goes up after rewriting the instruction section, we know instruction clarity was the bottleneck. Multi-section rewrites would confound the signal.
- The "one section" rule also limits blast radius: a bad rewrite can only damage one section, and the discard mechanism reverts it cleanly.

### Patterns applied
- **Memento-Skills loop**: Analyze failures, rewrite, re-evaluate, decide. The loop state (current_evals, attempted_sections, consecutive_no_improvement) carries forward without needing external memory.
- **Robin executor pattern**: I/O table, numbered steps (4.1-4.7), output checklist. Sub-steps within the loop mirror Phase 03's structure.
- **Rewrite feedback on discard**: When a rewrite is discarded, the failed diff and judge reasoning are passed to the next attempt. This prevents the rewriter from proposing the same change twice and gives it specific failure signal to learn from.
- **Positive framing in rewrites**: The rewrite rules enforce "Add X" over "Remove ambiguity about X" — this applies the same principle we use in skill authoring to the automated rewriting itself.

### Design decisions
- **Tie-breaking order** (instruction > activation > gate > reference > meta): Instruction sections have the highest surface area for behavioral change. Activation sections are second because they control whether the skill fires at all. Meta is last because metadata changes rarely affect behavior.
- **Easy-tier regression guard**: A rewrite that improves hard cases but breaks easy ones is net negative — easy cases represent the core use case. This prevents optimization for edge cases at the cost of basic functionality.
- **Max 2 retries per iteration**: Enough to recover from a bad first attempt (especially with the discard feedback), but not so many that the loop spends all its budget on one stubborn section.
- **Drift tracking by character count**: Simple proxy for semantic drift. A skill that doubles in length has likely drifted from its original scope. The 40% default is generous enough to allow meaningful additions but catches runaway expansion.
- **Re-eval procedure inlined**: The subagent running Phase 04 does not have Phase 03 loaded, so the re-eval procedure is summarized inline with all critical rules (information boundaries, borderline retry, temperature 0). This duplicates some content from Phase 03 but is necessary for correct execution.

### D1-D6 self-assessment
- **D2 (Execution Compliance)**: Strong. Seven sub-steps per iteration with clear sequencing. The decide step has explicit KEEP/DISCARD criteria with no ambiguity. Early stop conditions are enumerated as a checklist.
- **D3 (Behavioral Alignment)**: Strong. Easy-tier regression guard, drift limit, and exhaustion check prevent common optimization failure modes. Retry mechanism with feedback handles the stochastic nature of LLM rewrites.
- **D4 (Instruction Clarity)**: Good. Re-eval procedure is self-contained. Rewrite rules are concrete. However, mapping failures to sections (step 4.1) requires judgment — this is the least mechanical step.
- **D5 (Architecture)**: Good. References contracts.md for all schemas. 128 lines despite being the most complex phase. Config params have sensible defaults.
- **D6 (Evolvability)**: Good. Adding new early stop conditions = one bullet. Changing retry limit = one number. The loop structure supports adding new sub-steps (e.g., a "simplify" step after rewrite) without restructuring.
