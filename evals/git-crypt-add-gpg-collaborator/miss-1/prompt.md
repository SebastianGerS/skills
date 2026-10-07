---
name: git-crypt-add-gpg-collaborator-miss-1
description: A near-miss prompt git-crypt-add-gpg-collaborator should stay out of
tags: [trigger, git-crypt-add-gpg-collaborator]
max_turns: 3
allowed_tools: [Skill]
expected_outcome: The Skill tool is not called with git-crypt-add-gpg-collaborator; other skills may fire
---

Generate a new GPG key for me and upload it to my GitHub account.
