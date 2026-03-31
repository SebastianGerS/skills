# Agent Skills

A collection of agent skills that extend capabilities across planning, development, and knowledge.

## Planning & Design

- **grill-me** — Interview the user relentlessly about a plan or design until reaching shared understanding, resolving each branch of the decision tree.

  ```
  npx skills@latest add perhells/skills/grill-me
  ```

- **write-a-prd** — Create a PRD through user interview, codebase exploration, and module design, then submit as a new or updated Linear issue.

  ```
  npx skills@latest add perhells/skills/write-a-prd
  ```

- **prd-to-local-implementation-plan** — Mirror information from Linear for local tasks in order to enable sandboxed local agentic development.

  ```
  npx skills@latest add perhells/skills/prd-to-local-implementation-plan
  ```

## Knowledge

- **ubiquitous-language** — Extract a DDD-style ubiquitous language glossary from the current conversation, flagging ambiguities and proposing canonical terms.

  ```
  npx skills@latest add perhells/skills/ubiquitous-language
  ```

- **find-skills** — Helps users discover and install agent skills.

  ```
  npx skills@latest add perhells/skills/find-skills
  ```
