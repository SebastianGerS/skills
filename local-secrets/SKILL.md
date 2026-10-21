---
name: local-secrets
description: >
  Maps the API keys and tokens stored in Sebastian's macOS keychain (names and purposes,
  never values) and shows how to use one inline without it reaching the transcript. Use
  when a command needs a credential Sebastian keeps locally, such as the Hive, Vimond,
  Linear, Grafana, Aikido, metadata-ingest or metadata-api key, or when he asks "which
  key is that", "where is the X token" or wants to store a new secret.
---

# Local secrets in the keychain

Every credential Sebastian uses from his laptop lives in the macOS login keychain as a
generic password. `~/.zshrc` exports the env-var ones with a keychain lookup, so no file
holds a literal value. This skill is the map: what each item is for, how to use one
without printing it, and how to add a new one.

It holds names only. A value never belongs in this file, a memory note, a commit, a
Linear ticket or the transcript.

## Using a secret

Read it inline, in the same command that needs it:

```bash
H="$(security find-generic-password -s hive_api_key -w)"; \
  curl -sS -H "X-Api-Key: $H" https://...
```

- Never `echo`, `printf` or `cat` a value, and never pipe it to a file in the repo.
  Command output lands in the transcript on disk.
- Pass it through env without a value on the command line where the tool allows it,
  e.g. `docker run -e GH_TOKEN ...`, not `-e GH_TOKEN=$(...)` spelled out.
- Don't build URLs with a credential in them (`https://x-access-token:$T@github.com/`):
  tools such as `mix deps.get` write the rewritten URL into lockfiles.
- To check a secret exists or matches, compare without printing:
  `security find-generic-password -s <name> >/dev/null && echo exists`.
- Prefer the keychain read over the env var. Claude Code's shell keeps the env it
  started with, so after a rotation `$VAR` in this session is stale while the keychain
  is current.

If the auto-mode classifier blocks a read as credential materialisation, don't route
around it. Hand Sebastian the command to run himself with `!` or in his own terminal.

## Naming

- An item behind an env var is named after the variable, lowercased:
  `HIVE_API_KEY_PROD` → `hive_api_key_prod`.
- A service API key with no env var is `<service>-<brand>[-<env>]`:
  `metadata-ingest-tv4`, `metadata-api-mtv-prod`.

## The map

| Keychain item | Env var | Used for |
|---|---|---|
| `github_auth_token` | `GITHUB_AUTH_TOKEN`, `NODE_AUTH_TOKEN` | The `gh` login token (scopes `repo`, `read:org`, `read:packages`, `gist`): npm GitHub Packages for `@tv4/*`, GitHub API. Goes stale on every `gh auth login`; re-store it then. |
| `linear_api_key` | `LINEAR_API_KEY` | Linear API: `linear-ticket`, `aikido-weekly` |
| `grafana_api_key` | `GRAFANA_API_KEY` | Grafana API: `deploy-watch`, `incident-triage` |
| `aikido_api_key` | `AIKIDO_API_KEY` | Aikido API: `aikido-weekly` |
| `cc_token` | `CC_TOKEN` | Claude Code token for ralphish's `claude` agent |
| `vimond_secret` | `VIMOND_SECRET` | Vimond API: `tv4-metadata-ingest/tools/vimond-fetcher`, mms-title-sync tools |
| `hive_api_key` | `HIVE_API_KEY` | Hive API, stage: `hive-adapter/tools/hive-adapter-resend` |
| `hive_api_key_prod` | `HIVE_API_KEY_PROD` | Hive API, prod twin of `hive_api_key` |
| `docker_hub_token` | `DOCKER_HUB_TOKEN` | Nothing found using it (checked 2026-10-21); candidate to remove |
| `metadata-ingest-tv4`, `metadata-ingest-mtv` | — | Prod metadata-ingest API (`/raw/<source>/<id>`). Ingest also needs a `Service-Name` header and rate-limits bursts with 429. |
| `metadata-api-tv4-prod`, `metadata-api-mtv-prod` | — | Prod metadata-api API keys |

Ralphish keeps its own secrets in the keychain via `ralphish-secure`; see the `ralphish`
skill.

## When a key isn't here

For an internal `*.a2d.tv` service, `nexus-skills:a2d-tokens` finds where the key lives.
Store what it finds straight into the keychain under the naming rule above, then add a
row to the map. Only the name and purpose go in the row.

## Adding or replacing a secret

Sebastian runs this himself, outside Claude or with `!`. It prompts for the value, so the
value never appears on a command line:

```bash
security add-generic-password -U -a "$USER" -s <name> -w
```

From an env var that already holds it (zsh), without the value appearing in the command:

```bash
security add-generic-password -U -a "$USER" -s <name> -w "${(P)VAR}"
```

If it backs an env var, export it in `~/.zshrc` with a lookup, not a literal:

```bash
export VAR="$(security find-generic-password -s <name> -w 2>/dev/null)"
```

Then add a row to the map above.
