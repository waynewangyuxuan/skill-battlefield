# Phase 01 — Analyze Skill

Parse a skill file into structured analysis for downstream phases.

## Input / Output

| Item | Path |
|------|------|
| Skill directory | `{skill_dir}` (contains SKILL.md and supporting files) |
| Run directory | `{run_dir}` (write output here) |
| Output file | `{run_dir}/analysis.json` |

Output must conform to the `analysis.json` schema defined in `references/contracts.md`.

## Procedure

### 1. Read the skill file

Read `{skill_dir}/SKILL.md`. If that file does not exist, scan `{skill_dir}` for the first `.md` file alphabetically.

Store the raw content and total line count.

### 2. Handle edge cases

| Condition | Action |
|-----------|--------|
| File is empty (0 bytes) | Write analysis.json with `skill_type: "unknown"`, empty arrays for all list fields, `estimated_complexity: "low"` |
| File is binary (contains null bytes in first 512 bytes) | Same as empty |
| File exceeds 1000 lines | Analyze normally, set `estimated_complexity: "high"` |
| No YAML frontmatter | Derive `skill_name` from the first markdown heading; if no heading, use the filename without extension |
| No markdown headings | Treat the entire body (after any frontmatter) as one section named "Preamble" |

For empty or binary files, skip to step 7 (write output).

### 3. Extract frontmatter

If the file starts with `---`, parse the YAML frontmatter block. Extract:
- `skill_name` from the `name` field
- `intent` from the `description` field
- `trigger_conditions` from `when_to_use` or phrases containing "Use when", "Trigger when"
- `negative_triggers` from `when_not_to_use` or phrases containing "Do NOT use", "Not for"

### 4. Segment into sections

Walk the file line by line. Start a new section at each:
- YAML frontmatter block (lines between `---` delimiters)
- Markdown heading (`#`, `##`, `###`, etc.)

For each section, record:

| Field | Value |
|-------|-------|
| `id` | Sequential: `s1`, `s2`, `s3`... |
| `name` | Heading text (without `#` prefix), or `"Frontmatter"` / `"Preamble"` |
| `line_range` | `[start_line, end_line]` (1-indexed, inclusive) |
| `content_hash` | First 8 characters of the MD5 hex digest of the section text |

### 5. Classify section roles

Assign each section a `role` using the first matching rule:

| Role | Match when section contains |
|------|-----------------------------|
| `meta` | YAML frontmatter (between `---` delimiters) |
| `activation` | "when_to_use", "Use when", "Trigger when", "description:" with trigger language |
| `gate` | "HARD-GATE", "Do NOT proceed until", "verify before", "must pass before" |
| `reference` | Code blocks with examples, template literals, lookup tables, "Example:" |
| `instruction` | "always", "must", "do X before Y", numbered steps, imperative verbs |

If no rule matches, assign `instruction` as the default.

### 6. Classify skill type and complexity

**Skill type** — apply the first matching rule:

| Type | Condition |
|------|-----------|
| `multi-phase` | Has 2+ sections with distinct named phases, subagent dispatch, or phase transitions |
| `checklist` | Has numbered steps or bullet-point procedures |
| `prose` | Declarative conventions or style guide with no steps |

When ambiguous: named phases with distinct I/O = `multi-phase`. Flat numbered list = `checklist`.

**Estimated complexity:**

| Complexity | Condition |
|------------|-----------|
| `low` | Fewer than 5 sections |
| `medium` | 5 to 15 sections |
| `high` | More than 15 sections, or skill_type is `multi-phase` |

Override: files exceeding 1000 lines are always `high`.

**Constraints and gates:**
- Scan all sections for explicit rules: sentences with "always", "never", "must", "required". Collect these as `constraints`.
- Scan for verification checkpoints: "HARD-GATE", "verify", "check before", "do not proceed until". Collect these as `gates`.

### 7. Write output

Compute the final JSON object with all fields from the `analysis.json` schema in `references/contracts.md`.

Write the JSON to `{run_dir}/analysis.json` using Bash:

```bash
cat > "{run_dir}/analysis.json" << 'ANALYSIS_EOF'
{ ... the JSON ... }
ANALYSIS_EOF
```

Validate that the file is valid JSON by running:

```bash
python3 -c "import json; json.load(open('{run_dir}/analysis.json'))"
```

If validation fails, fix the JSON and rewrite.

## Output Checklist

- [ ] `analysis.json` exists at `{run_dir}/analysis.json`
- [ ] File is valid JSON
- [ ] All schema fields from `references/contracts.md` are present
- [ ] `sections` array has at least one entry (unless empty/binary input)
- [ ] Every section has a valid `role` from: activation, instruction, gate, reference, meta
- [ ] `skill_type` is one of: prose, checklist, multi-phase, unknown
