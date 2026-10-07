---
name: write-a-prd-miss-1
description: A near-miss prompt write-a-prd should stay out of
tags: [trigger, write-a-prd]
max_turns: 3
allowed_tools: [Skill]
expected_outcome: The Skill tool is not called with write-a-prd; other skills may fire
---

Create a ticket for the flaky test in segments.
