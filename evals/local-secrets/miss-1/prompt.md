---
name: local-secrets-miss-1
description: A near-miss prompt local-secrets should stay out of
tags: [trigger, local-secrets]
max_turns: 3
allowed_tools: [Skill]
expected_outcome: The Skill tool is not called with local-secrets; other skills may fire
---

My AWS SSO session expired, set up the prodcommon1 profile again.
