---
name: local-secrets-fires-2
description: A prompt local-secrets should fire on
tags: [trigger, local-secrets]
max_turns: 3
allowed_tools: [Skill]
expected_outcome: The Skill tool is called with local-secrets
---

Hit the tv4 metadata-ingest raw endpoint for vimond-asset 1234567 and tell me the title. Use my local key.
