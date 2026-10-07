---
name: git-crypt-add-gpg-collaborator-miss-2
description: A near-miss prompt git-crypt-add-gpg-collaborator should stay out of
tags: [trigger, git-crypt-add-gpg-collaborator]
max_turns: 3
allowed_tools: [Skill]
expected_outcome: The Skill tool is not called with git-crypt-add-gpg-collaborator; other skills may fire
---

Add erikw as a collaborator on the GitHub repo with write access.
