---
name: go-upgrade
description: >
  Upgrade Go dependencies and Docker base images safely, with special handling for
  OpenTelemetry (otel) ecosystem packages, semconv versions, cross-package version
  alignment (e.g. go-kit with otel), and Docker image version consistency. Trigger
  when the user asks to upgrade Go packages, bump Go version, update dependencies,
  fix version mismatches, update Docker images, or mentions otel/semconv version
  issues. Also trigger on phrases like "upgrade deps", "update go.mod", "bump otel",
  "fix version mismatch", "upgrade go-kit", "update Dockerfile", or "bump Docker image".
argument-hint: "[package-or-scope]"
allowed-tools: Read, Grep, Glob, Bash(go:*), Bash(git diff:*), Bash(git log:*)
---

# Go Dependency Upgrade Skill

Safely upgrade Go dependencies with special attention to OpenTelemetry ecosystem
coherence. OTel packages can compile fine with mismatched versions but cause
**runtime panics or silent data loss** — treat version alignment as a correctness
issue, not just a hygiene one.

## Step 1 — Assess current state

Before changing anything, gather the full picture:

```bash
# Current Go version
go version

# List all direct and indirect dependencies
go list -m all

# Show current otel-related dependencies
go list -m all | grep -E 'opentelemetry|otel|semconv'

# Show go-kit and other instrumentation bridge packages
go list -m all | grep -E 'go-kit|contrib'

# Check for outdated packages
go list -m -u all 2>/dev/null | grep '\[' | head -40
```

Read `go.mod` and `go.sum` to understand the dependency graph.

## Step 2 — Identify the upgrade scope

Based on the user's request, determine what to upgrade:

| Request | Scope |
|---------|-------|
| "upgrade otel" / "bump otel" | All `go.opentelemetry.io/*` packages |
| "upgrade go-kit" | `github.com/go-kit/*` + check otel bridge compatibility |
| "upgrade deps" / "update all" | All dependencies, otel ecosystem first |
| "fix version mismatch" | Diagnose and align mismatched packages |
| "upgrade Go" / "bump Go version" | Go toolchain version + check compatibility |
| Specific package name | That package + its otel/semconv implications |

## Step 3 — OpenTelemetry version alignment (CRITICAL)

This is the most dangerous part. OTel packages **must** be version-aligned.

### 3a — Map the OTel dependency groups

OTel packages are released together and must stay in sync within each group:

- **Core**: `go.opentelemetry.io/otel`, `go.opentelemetry.io/otel/trace`, `go.opentelemetry.io/otel/metric`, `go.opentelemetry.io/otel/sdk`, `go.opentelemetry.io/otel/sdk/metric`
- **Bridge/Contrib**: `go.opentelemetry.io/contrib/*` — these have their **own** version scheme but pin to specific core versions
- **Exporters**: `go.opentelemetry.io/otel/exporters/*` — must match core
- **Instrumentation**: `go.opentelemetry.io/contrib/instrumentation/*` — each targets a specific core version range

```bash
# Check the otel core versions in use
go list -m all | grep 'go.opentelemetry.io/otel' | grep -v contrib

# Check contrib versions
go list -m all | grep 'go.opentelemetry.io/contrib'

# Check what core version contrib expects
# (read contrib's go.mod to verify compatibility)
```

### 3b — Semconv version audit (HIGH RISK)

Semconv (`go.opentelemetry.io/otel/semconv`) version mismatches are the #1 source
of silent runtime errors. Different semconv versions define different attribute keys
and schema URLs.

**Failure modes:**
- Repo code uses `semconv/v1.26.0` but an imported library uses `semconv/v1.21.0` → duplicate/conflicting attributes at runtime
- Upgrading semconv without updating attribute references → attributes silently disappear from telemetry
- Mixed semconv versions across packages → schema URL conflicts in exporters

```bash
# Find ALL semconv version imports across the codebase
grep -r 'semconv/v1\.' --include='*.go' . | sort -u

# Find semconv versions pulled in by dependencies
go list -m all | grep semconv

# Check for multiple semconv versions in the build
go mod graph | grep semconv | sort -u
```

**Resolution strategy:**
1. Identify the highest semconv version required by any dependency
2. Update ALL repo code to use that same semconv version
3. After updating, grep for any remaining references to old semconv versions
4. Verify attribute names still exist in the new semconv version (they can be renamed or removed)

### 3c — Third-party bridge packages (go-kit, etc.)

Packages like `go-kit` that integrate with otel pin to specific otel versions.

```bash
# Check what otel version go-kit (or similar) expects
go mod graph | grep -E 'go-kit.*otel|contrib.*go-kit'

# Read the go.mod of the bridge package to find its otel requirements
go list -m -json github.com/go-kit/kit 2>/dev/null
```

**Key rule:** The otel version in your repo must be **compatible** with what your
bridge packages expect. If go-kit pins `otel v1.24.0` and you upgrade to `v1.28.0`,
the bridge may compile but produce incorrect spans or metrics at runtime.

**Resolution:**
1. Check the bridge package's changelog/go.mod for otel compatibility
2. Upgrade the bridge package first if a newer version supports your target otel version
3. If no compatible bridge version exists, hold the otel upgrade until one does — or fork/patch

## Step 4 — Perform the upgrade

Execute upgrades in this order to minimize breakage:

### Order of operations:
1. **OTel core packages** — upgrade as a group
2. **OTel contrib/exporters** — match to the new core version
3. **Bridge packages** (go-kit, etc.) — upgrade to versions compatible with new otel
4. **Semconv** — align to single version, update all references
5. **Other dependencies** — standard upgrades

### Patch version pinning — when dependencies already specify patch versions

When any dependency in the module graph already pins a **patch version** (e.g., `v1.28.1`),
you must also specify the full patch version in your `go get` commands.

**Why this matters:** If a transitive dependency requires e.g. `v1.28.1`, and you run
`go get package@v1.28` (minor only), Go resolves this to `v1.28.0`. Then `go mod tidy`
bumps it to the minimum patch required by the dependency graph (`v1.28.1`) — which may
NOT be the latest patch. This leaves you on an older patch with potentially missing bug
fixes or security patches.

**This only applies when patch versions are already in play.** If all dependencies use
`.0` patch versions, specifying just the minor version is fine.

**How to check if patch versions matter:**
```bash
# Check if any dependency already pins a non-zero patch version
go mod graph | grep 'go.opentelemetry.io/otel@' | awk '{print $2}' | sort -uV
# If you see versions like v1.28.1, v1.28.2 etc → you MUST specify patch versions

# Find the latest available patch version
go list -m -versions go.opentelemetry.io/otel

# Or check the latest version directly
go list -m -json go.opentelemetry.io/otel@latest 2>/dev/null | grep '"Version"'
```

**Rule:** Before running `go get`, check the dependency graph for existing patch versions.
If any are present, look up the latest patch and specify it explicitly. This applies to
any package, not just otel.

```bash
# Upgrade otel core (example — always use full MAJOR.MINOR.PATCH)
go get go.opentelemetry.io/otel@v1.XX.Y
go get go.opentelemetry.io/otel/trace@v1.XX.Y
go get go.opentelemetry.io/otel/metric@v1.XX.Y
go get go.opentelemetry.io/otel/sdk@v1.XX.Y
go get go.opentelemetry.io/otel/sdk/metric@v1.XX.Y

# Upgrade contrib to matching version (also full patch)
go get go.opentelemetry.io/contrib/instrumentation/...@v0.XX.Y

# Tidy and verify — then check that tidy didn't downgrade any patch versions
go mod tidy
git diff go.mod  # ← review: if any version went DOWN after tidy, investigate
```

### Post-tidy patch version check

After `go mod tidy`, verify that no patch versions were reverted:

```bash
# Compare go.mod before and after tidy — look for version decreases
# If a version decreased, a dependency is pulling in a lower version
# Fix by explicitly pinning with go get package@vX.Y.Z (latest patch)
```

## Step 5 — Verify correctness

Compilation is **necessary but not sufficient**. Perform these checks:

```bash
# Must compile
go build ./...

# Run tests
go test ./...

# Verify no mixed semconv versions remain
grep -r 'semconv/v1\.' --include='*.go' . | awk -F'semconv/' '{print $2}' | sort -u
# ↑ This MUST return exactly ONE version

# Verify no mixed otel core versions
go list -m all | grep 'go.opentelemetry.io/otel '
go mod graph | grep 'go.opentelemetry.io/otel ' | awk '{print $2}' | sort -u
# ↑ Check for version conflicts

# Check for replace directives that might mask issues
grep 'replace' go.mod
```

## Step 6 — Report results

Present a summary to the user:

| Category | Before | After | Notes |
|----------|--------|-------|-------|
| Go version | ... | ... | |
| OTel core | ... | ... | |
| OTel contrib | ... | ... | |
| Semconv | ... | ... | ⚠️ if changed |
| go-kit / bridges | ... | ... | ⚠️ if otel compat changed |
| Other packages | ... | ... | |

Flag any of these **runtime risk** warnings:
- ⚠️ Semconv version changed — verify attribute names in your code still exist
- ⚠️ Multiple semconv versions in dependency graph — may cause schema conflicts
- ⚠️ Bridge package otel version doesn't match repo otel version
- ⚠️ `replace` directives in go.mod that could mask version issues
- ⚠️ Indirect dependency pulling in a different otel version

## Step 7 — Docker image upgrades

When upgrading Go or dependencies, Dockerfiles must stay in sync.

### 7a — Find all Dockerfiles

```bash
# Find Dockerfiles in the project
find . -name 'Dockerfile*' -o -name '*.dockerfile' | head -20

# Also check for docker-compose files
find . -name 'docker-compose*' -o -name 'compose*' | head -20
```

### 7b — Go version alignment in Docker

The Go version in `go.mod` must match the Docker base image:

```bash
# Check go.mod version
grep '^go ' go.mod

# Check Dockerfile Go versions
grep -n 'FROM.*golang' Dockerfile*
```

**If the Go version was bumped**, update ALL Dockerfiles to match:
- `FROM golang:1.XX-alpine` → `FROM golang:1.YY-alpine`
- Multi-stage builds: update both builder and any intermediate stages

### 7c — Base image upgrades

When upgrading base images:

```bash
# Check current base images
grep -n '^FROM' Dockerfile*

# Common base images to keep aligned:
# - golang:X.YY-alpine → match go.mod version
# - alpine:X.YY → check for security updates
# - gcr.io/distroless/* → check for latest stable
# - scratch → no version to update
```

**Key checks:**
- If switching Alpine minor versions, verify that C dependencies (CGO) still compile
- If using distroless, check that the target platform is still supported
- If using a custom/private base image, verify it exists in the registry

### 7d — Docker build verification

After updating Dockerfiles:

```bash
# Verify the Dockerfile builds successfully
docker build -t test-build .

# If multi-stage, verify the final image runs
docker run --rm test-build --version 2>/dev/null || true

# Check image size hasn't exploded (sanity check)
docker images test-build --format '{{.Size}}'
```

### 7e — Docker Compose version alignment

If docker-compose files reference specific image versions or build args:

```bash
# Check for hardcoded versions in compose files
grep -n 'image:' docker-compose*.yml 2>/dev/null
grep -n 'GO_VERSION' docker-compose*.yml 2>/dev/null
```

Update any build args or image tags to match the new versions.

## Common pitfalls reference

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Mixed semconv versions | Missing attributes in traces/metrics, schema URL warnings | Align all to one version |
| OTel core/contrib mismatch | Panic on span creation, nil pointer in metric callbacks | Upgrade contrib to match core |
| go-kit otel bridge mismatch | Spans created but missing attributes, or duplicate spans | Match go-kit version to otel version per go-kit's compatibility matrix |
| Stale `go.sum` | Build works locally, fails in CI | Run `go mod tidy` after all changes |
| `replace` directive hiding conflict | Compiles fine, runtime uses wrong version | Remove replace, fix the real conflict |
| Go version mismatch in Dockerfile | CI builds fail or produce binaries with wrong version | Align `FROM golang:X.YY` with `go.mod` |
| Alpine version bump breaks CGO | Build fails with missing C headers | Pin Alpine version or install missing deps |
| Stale Docker cache after dep upgrade | Old dependencies baked into image layer | Use `--no-cache` or restructure layer order |
