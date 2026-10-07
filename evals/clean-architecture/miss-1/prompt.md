---
name: clean-architecture-miss-1
description: A near-miss prompt clean-architecture should stay out of
tags: [trigger, clean-architecture]
max_turns: 3
allowed_tools: [Skill]
expected_outcome: The Skill tool is not called with clean-architecture; other skills may fire
---

Audit the nordic-progress repo for clean architecture violations against its real import graph.
