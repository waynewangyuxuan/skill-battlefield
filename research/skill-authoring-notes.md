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
