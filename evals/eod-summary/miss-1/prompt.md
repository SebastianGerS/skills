---
name: eod-summary-miss-1
description: A near-miss prompt eod-summary should stay out of
tags: [trigger, eod-summary]
max_turns: 3
allowed_tools: [Skill]
expected_outcome: The Skill tool is not called with eod-summary; other skills may fire
---

Summarize what happened in #nexus-incidents this week.
