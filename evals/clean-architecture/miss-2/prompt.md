---
name: clean-architecture-miss-2
description: A near-miss prompt clean-architecture should stay out of
tags: [trigger, clean-architecture]
max_turns: 3
allowed_tools: [Skill]
expected_outcome: The Skill tool is not called with clean-architecture; other skills may fire
---

Refactor this function to reduce nesting; it has four levels of if statements.
