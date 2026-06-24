---
name: write-a-prd
description: Create a PRD through user interview, codebase exploration, and module design, then submit as a new or updated Linear issue. Use when user wants to write a PRD, create a product requirements document, or plan a new feature using Linear.
---

This skill will be invoked when the user wants to create a PRD and track it in Linear. You may skip steps if you don't consider them necessary.

1. Ask the user for a long, detailed description of the problem they want to solve and any potential ideas for solutions.

2. Explore the repo to verify their assertions and understand the current state of the codebase.

3. **Apply TVM conventions.** The following TVM Engineering Platform standards must be considered for every PRD. Surface any conflict between the user's proposal and these standards before proceeding.

   **Language & framework (choose one):**
   - **Go + Huma** — required when the service will handle high traffic, needs low latency, performs CPU-intensive work, or where long-term operational cost matters. Default when in doubt about scale.
   - **TypeScript/Node.js + Hono** — acceptable when traffic is clearly moderate, the team has strong TS/JS expertise and limited Go experience, and iteration speed is the priority.
   - Use the `tvm-docs:tvm-docs` skill to read `guidance/guidelines/language-choice` for the latest decision framework if the choice is unclear.

   **Naming (enforce these rules):**
   - Service/repo names: lowercase-with-hyphens, no redundant suffixes (`transcoder` not `transcoding-service`, `profiles` not `user-profile-management-api`)
   - Use agent nouns or clear nouns/verbs: `ingester`, `search`, `auth` — not gerunds like `ingesting`
   - Namespace prefixes for related components: `metadata-catalog`, `metadata-ingest`, `metadata-search`
   - Go packages: lowercase no separators (`videostream`, `contentcache`)
   - npm packages: `@tv4/<name>` with lowercase-hyphens
   - API path segments: hyphen-separated (`/similar-titles`, `/playable-from`)
   - JSON fields: camelCase (or snake_case if surrounding systems require it)
   - Enum values: `SCREAMING_SNAKE_CASE`

   **API design (for new or changed REST APIs):**
   - List endpoints: wrap in `{ "items": [...] }` to allow future top-level properties
   - Single-item endpoints: return the object directly, 404 for missing
   - Batchable endpoints: accept `?ids=0001,0002` style query params

   **Data storage (choose the right tool):**
   - Relational data → Aurora RDS PostgreSQL; never use t-series instances in production (use m/r-series)
   - NoSQL / unpredictable workloads → DynamoDB on-demand mode
   - Caching → Valkey (ElastiCache)
   - Use `tvm-docs:tvm-docs` to read `guidance/guidelines/data-storage` for edge cases.

   **Observability:**
   - All metrics must include labels: `brand` (e.g. `tv4`, `mtv`, `shared`), `job` (service name), `account` (`prodcommon1`, `stagecommon1`, `dev1`)
   - Use OpenTelemetry semantic conventions for HTTP metrics and traces
   - Use `tvm-docs:tvm-docs` for `operations/observability/observability` if adding custom metrics

   **Feature flags:** Consider Unleash for any gradual rollout, A/B test, or kill-switch. Use `tvm-docs:tvm-docs` to read `development/feature-flags` if applicable.

   **Git workflow:**
   - Trunk-based development: short-lived branches (1–2 days), squash merge to main
   - Branch naming: `<username>/<ticket-id>-<short-description>` (e.g. `john-doe/PLAT-1234-add-spell-correction`)
   - PRs require ≥1 approval + all CI checks passing (tests, linters, type checks, security scans)

   **Standard libraries (prefer over custom implementations):**
   - Go: go-kit (`github.com/go-kit/kit`, `github.com/go-kit/log`) for middleware, transport, logging, metrics, tracing, rate limiting, circuit-breaking
   - Node.js: node-kit (`@tv4/node-kit-*`) for the same concerns

4. Interview the user relentlessly about every aspect of this plan until you reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. Make sure to cover:
   - **Language choice** — Go (Huma) or Node.js (Hono), with justification from the framework above
   - **Brand scope** — TV4 only, MTV only, or shared codebase with both brand targets
   - **Service/repo naming** — verify it follows TVM naming rules before committing
   - **Observability plan** — what metrics (with correct labels), log fields, and trace spans will be added
   - **Feature flags** — whether Unleash will be used, which flags, and the rollout strategy
   - **Infrastructure** — new AWS resources, Scalr workspace changes, or changes to existing infra. For ECS-based services, infrastructure must use `@tv4/cdktf-extras` prefabs (`InfraStack`, `EcsFargateApp`, `EcsAlb`, `setupGithubActions`) rather than hand-rolled raw CDKTF providers. Flag any proposal that reaches for raw `@cdktf/provider-aws` for resources already abstracted by cdktf-extras.
   - **Secrets** — how secrets will be managed (TVM secrets management)
   - **Data storage** — which technology and why, per TVM guidelines above

5. Sketch out the major modules you will need to build or modify to complete the implementation. Actively look for opportunities to extract deep modules that can be tested in isolation.

   A deep module (as opposed to a shallow module) is one which encapsulates a lot of functionality in a simple, testable interface which rarely changes.

   Before proposing a custom module for middleware, transport, logging, metrics, tracing, rate limiting, or circuit-breaking, check whether go-kit (Go) or node-kit (Node.js) already covers it. If it does, the PRD should call for using it.

   Check with the user that these modules match their expectations. Check with the user which modules they want tests written for.

6. Ask the user whether to **create a new Linear issue** or **update an existing one**. If updating, ask for the issue identifier (e.g., NEX-123). Also ask for:
   - **Team** (required for new issues)
   - **Project** (optional)
   - **Parent issue** (optional)
   - **Labels** (optional)
   - **Priority** (optional: 0=None, 1=Urgent, 2=High, 3=Normal, 4=Low)

7. Once you have a complete understanding of the problem and solution, use the template below to write the PRD. Submit it as a Linear issue, with the PRD title as the issue `title` and the rendered template as the issue `description` (markdown).

<prd-template>

## Problem Statement

The problem that the user is facing, from the user's perspective.

## Solution

The solution to the problem, from the user's perspective.

## User Stories

A LONG, numbered list of user stories. Each user story should be in the format of:

1. As an <actor>, I want a <feature>, so that <benefit>

<user-story-example>
1. As a mobile bank customer, I want to see balance on my accounts, so that I can make better informed decisions about my spending
</user-story-example>

This list of user stories should be extremely extensive and cover all aspects of the feature.

## TVM Conventions

How this feature aligns with TVM Engineering Platform standards.

- **Language & framework** — Go + Huma / TypeScript + Hono, and the justification (traffic/performance requirements, team competency)
- **Standard libraries** — which go-kit or `@tv4/node-kit-*` packages will be used (logging, metrics, tracing, auth, etc.)
- **Brand scope** — TV4 only / MTV only / shared codebase; how brand-specific behaviour is handled at the deploy/config layer
- **Service & repo naming** — verified against TVM naming rules: lowercase-with-hyphens, no redundant suffixes, agent nouns preferred
- **API design** — path casing (hyphen-separated), field casing (camelCase / SCREAMING_SNAKE_CASE for enums), list response shape (`{ "items": [...] }`)
- **Observability** — metrics with `brand`/`job`/`account` labels (OpenTelemetry semantics), structured log fields, trace spans
- **Feature flags** — whether Unleash is used, which flag keys, and the rollout strategy
- **Data storage** — technology choice and justification (Aurora PostgreSQL / DynamoDB on-demand / Valkey); production instance class if RDS (no t-series)
- **Infrastructure** — new or changed AWS resources, Scalr workspace/variable changes; ECS services must use `@tv4/cdktf-extras` prefabs (`InfraStack`, `EcsFargateApp`, `EcsAlb`, `setupGithubActions`)
- **Secrets management** — how secrets are stored and injected
- **Security** — Aikido scanning coverage, input validation, auth/authz approach
- **Git workflow** — trunk-based development, branch naming (`<user>/<ticket-id>-<description>`), PR requirements (1 approval + CI)

Omit any subsection that is not applicable to this feature.

## Implementation Decisions

A list of implementation decisions that were made. This can include:

- The modules that will be built/modified
- The interfaces of those modules that will be modified
- Technical clarifications from the developer
- Architectural decisions
- Schema changes
- API contracts
- Specific interactions

Do NOT include specific file paths or code snippets. They may end up being outdated very quickly.

## Testing Decisions

A list of testing decisions that were made. Include:

- A description of what makes a good test (only test external behavior, not implementation details)
- Which modules will be tested
- Prior art for the tests (i.e. similar types of tests in the codebase)

## Out of Scope

A description of the things that are out of scope for this PRD.

## Further Notes

Any further notes about the feature.

</prd-template>
