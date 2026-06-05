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
