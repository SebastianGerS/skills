# Agent Skills

Sebastian's personal [Claude Code](https://claude.com/claude-code) skills: planning,
day-to-day workflow and a few TV4-shaped utilities. Team-wide TV4/Nexus skills live in
[`TV4/nexus-skills`](https://github.com/TV4/nexus-skills).

## Install

As a Claude Code plugin from this directory (the marketplace manifest is in `.claude-plugin/`):

```bash
claude plugin marketplace add ~/Dev/Scripts/skills
claude plugin install sebastian-skills@sebastian-skills
```

Or one skill at a time, on any runtime that reads `~/.agents/skills/`:

```bash
npx skills@latest add SebastianGerS/skills/<skill>
```

## Skills

| Skill | What it's for |
|-------|---------------|
| `write-a-prd` | Write a PRD through an interview, a codebase walk and a TVM conventions check, then file it as a Linear issue |
| `create-a-plan` | Same interview and module design, but the plan stays local in `plans/` and is queued in `progress.yaml` |
| `prd-to-local-implementation-plan` | Mirror a Linear issue's PRD into `plans/` and queue it in `progress.yaml` for a sandboxed ralphish run |
| `grill-me` | Interview the user one question at a time until every branch of a plan or design is resolved |
| `ralphish` | Run and manage the headless Worker/Reviewer/Orchestrator loop over the `progress.yaml` backlog |
| `clean-architecture` | Layer boundaries and the dependency rule for new Flutter and Go modules (auditing an existing Go repo is `go-clean-arch-qa` in nexus-skills) |
| `ubiquitous-language` | Extract a DDD-style glossary from the conversation into `UBIQUITOUS_LANGUAGE.md` |
| `human-voice` | Draft Slack, email, Linear and Notion text in the user's own voice: short, plain, no assistant tells |
| `eod-summary` | Build the day's EOD from git, GitHub, Linear, Slack and session history, cut it down, post it in `#nexus-eods` |
| `local-secrets` | Map of the API keys and tokens in the macOS keychain (names and purposes only) and how to use one inline without printing it |
| `git-crypt-add-gpg-collaborator` | Add a GitHub user's GPG key to the repo's git-crypt keyring on a branch, with the identity surfaced for confirmation |
| `find-skills` | Search the open skills ecosystem (skills.sh, `npx skills`) for an installable skill and vet it before recommending |

## Writing and editing skills

Skills follow the [superpowers `writing-skills`](https://github.com/obra/superpowers) discipline: establish baseline behavior, write the skill, verify it changes behavior.

`description` fields say *what the skill does* first, then *when to use it*: the key use case leads, followed by a few natural phrasings a user would actually type. Claude Code truncates the listing at 1,536 characters, and Claude 5-generation models trigger readily on descriptions, so a long trigger-phrase list or capitalised emphasis (MUST, ALWAYS, CRITICAL) makes a skill over-trigger rather than helps it fire. The same goes for the body: explain the reason behind a rule instead of shouting it. Keep `SKILL.md` under ~500 lines and move reference material into sibling files, because a loaded skill stays in context for the rest of the session. Measure with `/skill-doctor` and `claude plugin eval` before adding trigger phrases: `evals/` holds two should-fire and two near-miss prompts per skill, and `claude plugin eval . --ablation none --tag <skill>` scores them (see `evals/README.md`).
