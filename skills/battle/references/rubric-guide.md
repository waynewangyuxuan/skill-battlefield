# Rubric Design Guide

How to write expected_behaviors checks (Phase 02) and evaluate them (Phase 03).

## Principles

1. **Binary (MET/UNMET).** No ordinal scales. Each check is pass or fail.
2. **One criterion per check.** Never bundle multiple behaviors into one check.
3. **Behavioral anchors, not quality judgments.** Good: "Agent calls Read before Edit." Bad: "Agent handles the situation well."
4. **Observable actions only.** Each check references something the simulator can describe: a tool call, a question asked, an output produced, a step skipped.
5. **Balance difficulty.** 2-4 checks per scenario. At least one achievable by baseline (prevents floor). At least one challenging (prevents ceiling).

---

## Good Checks

| check | description |
|---|---|
| reads_file_before_edit | Agent uses Read tool on the target file before any Edit call |
| asks_which_test | Agent asks the user to specify which test file or test name |
| does_not_modify_unrelated | Agent does not Edit any file not mentioned in the error output |
| presents_table_format | Agent outputs a markdown table matching the specified format |
| follows_priority_order | Agent sorts results by the priority criteria specified in the skill |

## Bad Checks

| check | Why bad |
|---|---|
| handles_well | "Well" is subjective — not binary |
| good_workflow | Too vague — what specific step? |
| asks_and_reads | Bundled — split into two checks |
| appropriate_response | No observable anchor — what makes it "appropriate"? |

---

## Judge Instructions

- Evaluate each check independently. Do not let one check influence another.
- Use temperature 0 for determinism.
- For scores of 4, 5, or 6 (near the pass/fail threshold): evaluate 3 times, take majority.
- Cite specific evidence from the simulated response for every pass/fail decision.
- Score formula: (passed_checks / total_checks) × 10, adjusted ±1 for quality factors.
