---
name: ralphish
description: Use when the user wants to run ralphish, kick off headless/automated coding on the task backlog, add/cancel/inspect tasks in progress.yaml, or check what ralphish is working on. Triggers: "run ralphish", "start the loop", "add a task to the backlog", "what is ralphish doing", "resume ralphish".
---

# Ralphish

Headless agentic task runner (Claude or Codex) in a persistent Docker sandbox. Each iteration runs three phases — Worker (implements one task), Reviewer (reviews the diff), Orchestrator (updates state, picks next work) — over tasks in `progress.yaml`.

- Binary: `~/.local/bin/ralphish` — `ralphish --help` is authoritative (README may lag the binary)
- Source + full docs: `~/Dev/Scripts/ralphish` (README.md)
- Standard workspace: `~/Dev/Projects` (backlog `progress.yaml`, secrets `.ralphish.env`)

## Invoking

Always pass `--env-file .ralphish.env` — it carries `GH_TOKEN`, `LINEAR_API_KEY`, etc.; without it the worker cannot push branches or open PRs. The path is resolved relative to the current directory, so either `cd ~/Dev/Projects` first or use absolute paths for both the env file and the workspace.

```bash
cd ~/Dev/Projects
ralphish --env-file .ralphish.env .                     # one iteration (default)
ralphish --env-file .ralphish.env --max-iterations 5 .  # up to 5 iterations
ralphish --env-file .ralphish.env --new .               # recreate sandbox first
ralphish --rm .                                         # remove sandbox
```

Runs are long (minutes per iteration) — from Claude Code, launch with `run_in_background` and check output periodically.

Headless caveats (running from Claude Code / CI rather than a terminal):

- Requires the `[ -t 0 ]` TTY guard in `~/Dev/Infra/docker-sandbox/sandbox` (prompt mode); without it every phase dies with `the input device is not a TTY`. Fixed locally 2026-07-14; check it's still there after pulling that repo — upstream may not have it.
- With `--include-local-config`, a `model` pinned in `~/.claude/settings.json` (e.g. `fable`) is synced into the sandbox, and the worker fails with "issue with the selected model" when the sandbox token can't use that alias. Pass `--worker-model claude-opus-5` as the fallback — reviewer/orchestrator already get explicit models.

Other flags: `--codex`/`--agent codex` (Codex instead of Claude), `--include-local-config` (copy local plugins/skills into sandbox), `--env KEY=VALUE`, `--reviewer-model` / `--orchestrator-model` (default sonnet-tier), `--worker-model` (fallback only — normally the orchestrator sets `worker_tier: simple|medium|complex|frontier` in `context.yaml` and ralphish resolves the model from that).

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

- Forgetting `--env-file .ralphish.env` → worker fails at push/PR time, task gets rejected for the wrong reason
- Editing `progress.yaml` while a loop is running — the orchestrator rewrites it each iteration
- Reaching for `--new` routinely — it wipes the sandbox; use only when the sandbox is broken or stale
- Adding a task without bumping `next_id`
