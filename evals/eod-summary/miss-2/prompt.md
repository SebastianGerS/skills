---
name: eod-summary-miss-2
description: A near-miss prompt eod-summary should stay out of
tags: [trigger, eod-summary]
max_turns: 3
allowed_tools: [Skill]
expected_outcome: The Skill tool is not called with eod-summary; other skills may fire
---

Give me a summary of this PR's changes before I review it.
