---
name: create-a-plan-miss-2
description: A near-miss prompt create-a-plan should stay out of
tags: [trigger, create-a-plan]
max_turns: 3
allowed_tools: [Skill]
expected_outcome: The Skill tool is not called with create-a-plan; other skills may fire
---

What's the plan for the 3pm deploy? Just list the order of steps you'd run.
