# Agent Skills

A collection of agent skills that extend capabilities across planning, development, and knowledge.

## Planning & Design

- **write-a-prd** — Create a PRD through user interview, codebase exploration, and module design, then submit as a new or updated Linear issue.

  ```
  npx skills@latest add SebastianGerS/skills/write-a-prd
  ```

- **prd-to-local-implementation-plan** — Mirror information from Linear for local tasks in order to enable sandboxed local agentic development.

  ```
  npx skills@latest add SebastianGerS/skills/prd-to-local-implementation-plan
  ```

- **grill-me** — Interview the user relentlessly about a plan or design until reaching shared understanding, resolving each branch of the decision tree.

  ```
  npx skills@latest add SebastianGerS/skills/grill-me
  ```

- **linear-ticket** — Fetch full context from a Linear ticket before acting on it. Triggers whenever a Linear issue is referenced by ID, URL, or phrases like "this ticket".

  ```
  npx skills@latest add SebastianGerS/skills/linear-ticket
  ```

## Development

- **go-upgrade** — Safely upgrade Go dependencies and Docker base images with special handling for OpenTelemetry ecosystem coherence, semconv alignment, and bridge package compatibility.

  ```
  npx skills@latest add SebastianGerS/skills/go-upgrade
  ```

## Knowledge

- **ubiquitous-language** — Extract a DDD-style ubiquitous language glossary from the current conversation, flagging ambiguities and proposing canonical terms.

  ```
  npx skills@latest add SebastianGerS/skills/ubiquitous-language
  ```

- **find-skills** — Helps users discover and install agent skills.

  ```
  npx skills@latest add SebastianGerS/skills/find-skills
  ```

## Productivity

- **eod-summary** — Generate a structured end-of-day summary from git, GitHub, Linear, Slack, and session history. Outputs Slack-formatted Swedish standup updates.

  ```
  npx skills@latest add SebastianGerS/skills/eod-summary
  ```
