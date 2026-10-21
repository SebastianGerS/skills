---
name: local-secrets-miss-2
description: A near-miss prompt local-secrets should stay out of
tags: [trigger, local-secrets]
max_turns: 3
allowed_tools: [Skill]
expected_outcome: The Skill tool is not called with local-secrets; other skills may fire
---

Move the metadata-ingest database password into AWS Secrets Manager in the CDKTF stack.
