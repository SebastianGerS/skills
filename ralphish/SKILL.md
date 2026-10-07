---
name: ralphish
description: >
  Runs and manages ralphish, the headless Worker/Reviewer/Orchestrator loop that works
  the progress.yaml backlog in a Docker sandbox: starting it with the right flags,
  adding or cancelling tasks, and reading its state files. Use for "run ralphish",
  "resume ralphish", "what is ralphish doing" or "add a task to progress.yaml".
---

# Ralphish

Headless agentic task runner (Claude or Codex) in a persistent Docker sandbox. Each iteration runs three phases — Worker (implements one task), Reviewer (reviews the diff), Orchestrator (updates state, picks next work) — over tasks in `progress.yaml`.

- Binary: `~/.local/bin/ralphish` — `ralphish --help` is authoritative (README may lag the binary)
- **Invoke through `~/.local/bin/ralphish-secure`, not `ralphish` directly** — same flags, but it supplies the secrets from the macOS keychain
- Source + full docs: `~/Dev/Scripts/ralphish` (README.md)
- Standard workspace: `~/Dev/Projects` (backlog `progress.yaml`)
- Secrets: macOS keychain, service `ralphish`, one item per variable. There is no `.ralphish.env` any more — it was deleted once the keychain held its contents

## Invoking

**Run `ralphish-secure`, never bare `ralphish`.** It reads `NODE_AUTH_TOKEN`, `AIKIDO_API_KEY` and `LINEAR_API_KEY` from the keychain and `GH_TOKEN` from `gh auth token`, writes them to a 0600 temp env file, passes it as `--env-file`, and deletes it when the run ends (Ctrl-C included). Every other flag passes straight through. Without those secrets the worker cannot push branches or open PRs.

`ralphish-secure --check` lists which keys resolve, without printing values — run it first if a worker fails at push or PR time. `SLACK_WEBHOOK_URL` showing `not set (optional)` is expected; it only feeds the PAUSE notification, which fails open.

**Always pass `--include-local-config` too**, so the sandboxed worker gets the same skills, plugins, and agent config as the local session — without it the worker has none of the team's conventions or tooling. This syncs the `model` pinned in `~/.claude/settings.json` into the sandbox, so **always also pass `--worker-model claude-opus-5`** as insurance for iteration 1 (when `context.yaml` has no `worker_tier` yet) in case the sandbox token can't use whatever alias is pinned locally.

**Before starting an unrelated task**, blank `.ralphish/context.yaml` in the workspace (`rm -f .ralphish/context.yaml` or `> .ralphish/context.yaml`). Nothing in the harness wipes it between runs, and a stale `prior_attempt_branch` from a previous task makes the worker try to check out that task's branch instead of starting fresh.

```bash
cd ~/Dev/Projects
[ -n "$(cat .ralphish/context.yaml 2>/dev/null)" ] && echo "context.yaml is non-empty — blank it first if this is a new/unrelated task"

ralphish-secure --include-local-config --worker-model claude-opus-5 .                     # one iteration (default)
ralphish-secure --include-local-config --worker-model claude-opus-5 --max-iterations 5 .  # up to 5 iterations
ralphish-secure --include-local-config --worker-model claude-opus-5 --new .               # recreate sandbox first
ralphish --rm .                                                                                              # remove sandbox
```

Runs are long (minutes per iteration) — from Claude Code, launch with `run_in_background` and check output periodically.

Headless caveats (running from Claude Code / CI rather than a terminal):

- Requires the `[ -t 0 ]` TTY guard in `~/Dev/Infra/docker-sandbox/sandbox` (prompt mode); without it every phase dies with `the input device is not a TTY`. Fixed locally 2026-07-14; check it's still there after pulling that repo — upstream may not have it.
- `--include-local-config` syncs the `model` pinned in `~/.claude/settings.json` into the sandbox, and the worker fails with "issue with the selected model" when the sandbox token can't use that alias — this is exactly what the default `--worker-model claude-opus-5` above guards against. Reviewer/orchestrator already get explicit models, so they're unaffected.

Other flags: `--codex`/`--agent codex` (Codex instead of Claude), `--env KEY=VALUE`, `--reviewer-model` / `--orchestrator-model` (default sonnet-tier), `--worker-model` (fallback only — normally the orchestrator sets `worker_tier: simple|medium|complex|frontier` in `context.yaml` and ralphish resolves the model from that).

## Managing tasks (progress.yaml)

- Statuses the user sets: `todo`, `in_progress`, `done`, `cancelled`; the orchestrator may also set `blocked`/`skipped`
- Adding a task: append under `tasks:` using the current `next_id` as `id`, then increment `next_id`:

```yaml
- id: 25
  title: "NEX-2001: Bump Go in nordic-progress"
  description: >
    Patch-pin Go X.Y.Z in go.mod, workflows, and Dockerfile.
  status: todo
  blocked_by: []
  repo: nordic-progress
```

- If the task has a Linear issue, put the ID in the title: `"NEX-1540: …"`
- Cancelling: mark `cancelled` in progress.yaml AND cancel the Linear issue together
- `repo:` scopes the sandbox to a single repo (path relative to progress.yaml, e.g. `partners`) — set it whenever the task belongs to one repo
- `blocked_by: [ids]` gates task selection; ralphish picks the first `todo` task whose blockers are all `done`

Task authoring pipeline for bigger work: `/grill-me` → `/write-a-prd` → `/prd-to-local-implementation-plan` produces progress.yaml tasks.

## Monitoring a run

- `~/Dev/Projects/.ralphish/context.yaml` — orchestrator's current state: assigned task, feedback, `worker_tier` (transient; wiped each invocation)
- `.ralphish/summary.yaml` / `.ralphish/review.yaml` — what the worker did / reviewer verdict this iteration
- `progress.yaml` `orchestration.history` — durable append-only log of iteration outcomes
- `.ralphish-metrics.jsonl` — harness-written, one JSON line per iteration: per-phase wall-clock and token counts (agents no longer self-report tokens)
- Exit code 0 with tasks remaining usually means the orchestrator signalled PAUSE (external action needed — e.g. merge a PR); do the action, then re-run the same command to resume

## Common mistakes

- Running bare `ralphish` instead of `ralphish-secure` → no secrets, worker fails at push/PR time and the task gets rejected for the wrong reason
- Forgetting `--include-local-config` → worker runs without local skills/plugins/conventions
- Starting a new task with a stale `.ralphish/context.yaml` left over from a previous one → worker tries to resume the wrong branch; blank the file first
- Editing `progress.yaml` while a loop is running — the orchestrator rewrites it each iteration
- Reaching for `--new` routinely — it wipes the sandbox; use only when the sandbox is broken or stale
- Adding a task without bumping `next_id`
