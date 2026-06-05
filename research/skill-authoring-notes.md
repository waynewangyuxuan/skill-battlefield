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
