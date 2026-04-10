---
name: linear-ticket
description: >
  Fetch full context from a Linear ticket before acting on it. Trigger this skill
  whenever the user references a Linear issue — by ID (e.g. NEX-123, CEP-456), by URL
  (linear.app/…/issue/NEX-123), or phrases like "this ticket", "the issue", "make
  a plan for NEX-123", "fix this Linear ticket", "how complex is STD-42", or any
  request to plan, implement, estimate, review, or discuss a Linear issue. Always
  fetch the ticket first — never proceed without the full context.
---

# Linear Ticket Skill

Always fetch ticket context before planning, coding, estimating, or discussing a Linear issue.

## Step 1 — Extract the issue ID

Parse the issue ID from the user's message. Issue IDs follow the pattern `<TEAM_KEY>-<NUMBER>`
where the team key is a short uppercase prefix (2–5 letters). Known team prefixes in the org
include (non-exhaustive):

| Prefix | Team |
|--------|------|
| `NEX` | Nexus |
| `PAJ` | Köket |
| `STD` | Streaming Delivery |
| `STV` | Smart TV React |
| `CRM` | CRM B2B |
| `ADO` | Ad Orchestration |
| `CON` | Consumer Growth & Retention |
| `CEP` | Core Engineering Platform |
| `AID` | AI Development Hub |
| `INF` | Infrastructure |

Other teams exist — do not reject an ID just because the prefix isn't listed above.
Any `[A-Z]{2,5}-\d+` pattern is a valid Linear issue ID.

| Input form | Extracted ID |
|---|---|
| `NEX-123` | `NEX-123` |
| `https://linear.app/tv4/issue/NEX-123/title-slug` | `NEX-123` |
| Current git branch (e.g. `nex-123-fix-thing`) | `NEX-123` |

If no ID is found anywhere, ask: *"Could you share the Linear ticket ID or URL?"*

## Step 2 — Choose your data source

Try the **`linear` CLI** first. Run `which linear` to check availability. If the CLI is not installed or any CLI command fails with a "command not found" or connection error, fall back to the **Linear MCP tools** instead.

### Option A — Linear CLI (preferred)

```bash
# Full issue details (title, description, status, priority, assignee, project, labels, linked issues)
linear issue view NEX-123

# Comments on the issue
linear issue comment list NEX-123

# If the issue belongs to a project, fetch project context
# (project ID will be shown in the issue view output)
linear project view <project-id>
```

If the issue view output includes linked/blocking issues, fetch those too:
```bash
linear issue view <linked-issue-id>
```

#### Auth troubleshooting (CLI)
If any command fails with an auth error, tell the user to run `linear auth login` or set `LINEAR_API_KEY` in their environment.

### Option B — Linear MCP (fallback)

If the CLI is unavailable, use the Linear MCP tools (provided by the `linear` MCP server plugin at `https://mcp.linear.app/mcp`). Use ToolSearch to find available Linear MCP tools — they typically follow naming like `mcp__linear__*` or `mcp__plugin*linear*`.

Use the MCP tools to:
1. **Search for the issue** by its ID (e.g. `NEX-123`) to get full details — title, description, status, priority, assignee, project, labels, linked issues.
2. **Fetch comments** on the issue.
3. **Fetch project context** if the issue belongs to a project.
4. **Fetch linked/blocking issues** if any are referenced.

#### Auth troubleshooting (MCP)
If MCP tools return auth errors, tell the user to check that the Linear MCP plugin is enabled and authenticated (it uses OAuth via `https://mcp.linear.app/mcp`).

## Step 3 — Fetch linked GitHub PRs

If the issue output contains a GitHub PR link (e.g. in Attachments), fetch it using the `web_fetch` tool to get the full PR description — this often contains the real implementation details, especially for tickets with sparse descriptions.

## Step 4 — Present a summary

Produce a structured summary from the fetched data:

**Summary card**
| Field | Value |
|---|---|
| **ID** | NEX-123 |
| **Title** | … |
| **Status** | e.g. Todo / In Progress / Done |
| **Priority** | Urgent / High / Medium / Low / No priority |
| **Assignee** | … |
| **Project / Milestone** | … |
| **Labels** | … |
| **Cycle** | … (if present) |
| **Blocking / Blocked by** | … (if present) |

Then write a short prose summary (3–5 sentences) covering:
- What the ticket is asking for
- Acceptance criteria or definition of done
- Relevant technical context from the description
- Key decisions or open questions from comments
- Dependencies or blockers

## Step 5 — Proceed with the user's request

With full ticket context in hand, fulfil the original ask:

- **"Make a plan" / "How do I implement this?"** → Write a concrete step-by-step implementation plan grounded in the ticket details, including specific files/areas of the codebase to touch if inferable.
- **"How hard / complex is this?"** → Estimate effort with reasoning drawn from the ticket scope and any linked issues.
- **"Fix this ticket"** → Use the ticket context to write or modify code.
- **"Summarise"** → The summary from Step 4 is the full response.

Always ground your response in what the ticket actually says — don't invent requirements.
