---
name: eod-summary
description: Use when the user asks for an end-of-day summary, daily standup prep, status update, or wants to know what they worked on today. Also use when user says "eod", "standup", "what did I do today", or "daily summary".
---

# EOD Summary

Generate a structured end-of-day summary by pulling from git commits, GitHub PRs/activity, Linear issues, Slack activity, Claude Code session history, and Codex session history.

## Execution

Run all data-gathering steps in parallel using Bash, then compile the results.

### Step 1: Git Commits

```bash
for repo in ~/Dev/Projects/*/; do
  if [ -d "$repo/.git" ]; then
    email=$(git -C "$repo" config user.email 2>/dev/null)
    commits=$(git -C "$repo" log --oneline --after="midnight" --author="$email" 2>/dev/null)
    if [ -n "$commits" ]; then
      echo "### $(basename $repo)"
      echo "$commits"
      echo ""
    fi
  fi
done
```

### Step 2: GitHub Activity

Use `gh` CLI to fetch PRs and commits pushed/merged today. Run in parallel:

```bash
# PRs opened or merged today by the authenticated user
TODAY=$(date +%Y-%m-%d)
GH_USER=$(gh api user --jq '.login' 2>/dev/null)

echo "=== PRs opened today ==="
gh search prs --author="$GH_USER" --created="$TODAY" --limit 30 --json url,title,state,repository 2>/dev/null \
  | python3 -c "import json,sys; [print(f\"- [{o['repository']['nameWithOwner']}] {o['title']} ({o['state']}) {o['url']}\") for o in json.load(sys.stdin)]" 2>/dev/null

echo "=== PRs merged today ==="
gh search prs --author="$GH_USER" --merged="$TODAY" --limit 30 --json url,title,state,repository 2>/dev/null \
  | python3 -c "import json,sys; [print(f\"- [{o['repository']['nameWithOwner']}] {o['title']} (merged) {o['url']}\") for o in json.load(sys.stdin)]" 2>/dev/null

echo "=== Commits pushed today (across local repos) ==="
for repo in ~/Dev/Projects/*/; do
  if [ -d "$repo/.git" ]; then
    email=$(git -C "$repo" config user.email 2>/dev/null)
    remote=$(git -C "$repo" remote get-url origin 2>/dev/null | sed 's/.*github.com[:/]//' | sed 's/\.git$//')
    # Commits authored today that exist on remote branches (i.e. pushed)
    pushed=$(git -C "$repo" log --oneline --after="midnight" --before="tomorrow" \
      --author="$email" --remotes 2>/dev/null | head -20)
    if [ -n "$pushed" ] && [ -n "$remote" ]; then
      echo "### $remote"
      echo "$pushed"
    fi
  fi
done
```

Collect: PR titles, repos, states (opened/merged), and pushed commit summaries. If `gh` is not authenticated or fails, skip and note it.

### Step 3: Linear Issues

Run these commands to capture both active and recently completed issues:

```bash
# Active issues (In Progress, In Review, Todo) — most likely to have today's updates
linear issue list --sort priority --state started --no-pager --team NEX --limit 20 2>&1
linear issue list --sort priority --state unstarted --no-pager --team NEX --limit 10 2>&1
# Recently completed — catch issues closed today
linear issue list --sort priority --state completed --no-pager --team NEX --limit 10 2>&1
```

Filter output for issues updated "today", "just now", "X seconds ago", "X minutes ago", or "X hours ago" (exclude "X days ago" and older). Include ID, title, and state. If `linear` fails, skip and note it.

**Enrich with details:** For each ticket updated today, fetch full context in parallel:

```bash
linear issue view <ID> --no-pager --json 2>&1
```

This returns JSON with `state`, `description`, `comments`, `attachments`, `children`, and `project`. Use this to write richer "Idag" bullets — e.g. mention PR links from attachments, note state transitions (moved to "In Review"), or summarize what was done based on description + comments.

### Step 4: Claude Code Sessions

```bash
python3 << 'PYEOF'
import json, os
from datetime import datetime

today_start = datetime.now().replace(hour=0, minute=0, second=0).timestamp()
claude_projects = os.path.expanduser("~/.claude/projects")

for dirpath, _, filenames in os.walk(claude_projects):
    for f in filenames:
        if not f.endswith(".jsonl") or "subagent" in dirpath or "compact" in f:
            continue
        path = os.path.join(dirpath, f)
        if os.path.getmtime(path) < today_start:
            continue
        project = os.path.relpath(dirpath, claude_projects).split("/")[0]
        for prefix in ["-Users-ZHO223-Dev-Projects-", "-Users-ZHO223-"]:
            project = project.replace(prefix, "")
        with open(path) as fh:
            for line in fh:
                try:
                    obj = json.loads(line)
                    if obj.get("type") != "user":
                        continue
                    content = obj.get("message", {}).get("content", "")
                    if isinstance(content, list):
                        content = " ".join(
                            c.get("text", "") for c in content
                            if isinstance(c, dict) and c.get("type") == "text"
                        )
                    text = content.strip()
                    # Skip IDE context, system messages, and commands
                    if not text or text.startswith(("<ide_", "<command", "<system")):
                        continue
                    # Strip leading tags from mixed content
                    import re
                    text = re.sub(r'^<[^>]+>[^<]*</[^>]+>\s*', '', text).strip()
                    if not text:
                        continue
                    print(f"- {project}: {text[:150]}")
                    break
                except:
                    pass
PYEOF
```

### Step 5: Codex Sessions

Codex stores a session index at `~/.codex/session_index.jsonl` and session files at `~/.codex/sessions/YYYY/MM/DD/*.jsonl`. The index has `thread_name` and `updated_at`; session files have `cwd` in the first line's `payload`.

```bash
python3 << 'PYEOF'
import json, os
from datetime import datetime

today_str = datetime.now().strftime("%Y-%m-%d")
today_dir = datetime.now().strftime("%Y/%m/%d")

# Build session ID → project mapping from today's session files
sessions_dir = os.path.expanduser(f"~/.codex/sessions/{today_dir}")
id_to_project = {}
if os.path.isdir(sessions_dir):
    for f in os.listdir(sessions_dir):
        if not f.endswith(".jsonl"):
            continue
        path = os.path.join(sessions_dir, f)
        with open(path) as fh:
            try:
                first = json.loads(fh.readline())
                cwd = first.get("payload", {}).get("cwd", "")
                sid = first.get("payload", {}).get("id", "")
                project = cwd.split("/")[-1] if "/" in cwd else cwd
                if sid:
                    id_to_project[sid] = project
            except:
                pass

# Read session index for today's entries
index_path = os.path.expanduser("~/.codex/session_index.jsonl")
seen = set()
if os.path.isfile(index_path):
    with open(index_path) as f:
        for line in f:
            try:
                obj = json.loads(line)
                updated = obj.get("updated_at", "")
                if not updated.startswith(today_str):
                    continue
                name = obj.get("thread_name", "")
                sid = obj.get("id", "")
                key = (sid, name)
                if key in seen or not name:
                    continue
                seen.add(key)
                project = id_to_project.get(sid, "unknown")
                print(f"- {project}: {name}")
            except:
                pass
PYEOF
```

### Step 6: Slack Activity

Use the Slack MCP tools to gather today's relevant Slack activity. The current user's Slack user ID is `U0314GC3YRY`. Run these three searches in parallel using `mcp__plugin_slack_slack__slack_search_public_and_private`:

**Search 1 — Messages I wrote today:**
```
query: "from:<@U0314GC3YRY> on:YYYY-MM-DD"
sort: "timestamp"
response_format: "concise"
include_context: false
limit: 20
```

**Search 2 — Messages mentioning me today:**
```
query: "to:<@U0314GC3YRY> on:YYYY-MM-DD"
sort: "timestamp"
response_format: "concise"
include_context: false
limit: 20
```

**Search 3 — Threads I participated in (updated today by others):**
```
query: "from:<@U0314GC3YRY> is:thread on:YYYY-MM-DD"
sort: "timestamp"
response_format: "concise"
include_context: false
limit: 20
```

Replace `YYYY-MM-DD` with today's date.

For any threads found in the results, use `mcp__plugin_slack_slack__slack_read_thread` to check if others have replied today. This catches conversations where the user is involved but may not have been explicitly mentioned.

**Processing the results:**
- Group by channel — note which channels had activity
- Identify discussion topics and decisions made
- Flag any open questions or action items directed at the user
- Note any threads where the user is waiting for a response or where someone is waiting for theirs
- Skip bot messages, join/leave notifications, and automated messages
- Use the Slack activity to enrich "Idag" bullets (e.g. "Diskuterade X med Y i #channel") and to surface blockers/open questions for "Top of mind"

If the Slack MCP tools are not authenticated or fail, skip gracefully and note the gap.

### Step 7: Infer "Imorgon/Måndag"

Before compiling, scan the gathered data for signals of unfinished work that is likely to continue the next day. Use these heuristics:

**Git signals:**
- Commits on a branch that hasn't been merged to main → likely needs a PR or deploy tomorrow
- A merge to stage but not to main → probably needs prod deploy
- A hotfix or fix commit late in the day → might need monitoring or follow-up

**GitHub signals:**
- PRs opened today but not yet merged → likely needs review follow-up tomorrow
- PRs merged today → check if deployment or monitoring follow-up is needed

**Linear signals:**
- Issues still "In progress" or "In Review" after today's activity → likely continued tomorrow
- Issues with incomplete task checklists (unchecked `- [ ]` items in description) → remaining tasks
- Issues where a PR exists but is not merged → follow up on review

**Session signals:**
- Claude/Codex session that ended mid-task (e.g. "watch metrics", "investigate", "plan") → may need follow-up
- Any session where the user mentioned "tomorrow", "later", or "next" in their messages

**Slack signals:**
- Open questions or requests directed at the user that haven't been answered yet → follow up tomorrow
- Discussions the user started that are still ongoing → may need to check back
- Action items assigned to the user in Slack threads

Based on these signals, draft 1–3 bullets for "Imorgon" (or "Måndag" on Friday) as a suggested starting point. Mark it clearly as a suggestion — the user will confirm or adjust. Phrase as intentions, e.g. "Fortsätta med X", "Merga och deploya Y till prod", "Följa upp Z".

### Step 8: Infer "Top of mind"

Scan all gathered data for things that don't fit cleanly into "what I did" or "what I'll do next" — things that are on your mind but not yet actions. Look for signals like:

**Blockers / waiting on others:**
- PRs awaiting review from someone else
- Linear issues blocked or waiting on external input
- Deploys that need sign-off

**Open questions / uncertainty:**
- A session that explored something without a clear resolution (e.g. "investigate X", "figure out why Y")
- A hotfix or incident that may not be fully resolved
- Architectural or design decisions that came up but weren't settled

**Things to watch:**
- A recent deploy that should be monitored
- A flaky test or intermittent failure that surfaced today
- Metrics or alerts that were mentioned in sessions

**Slack signals:**
- Unanswered questions or requests directed at the user (mentions without a reply)
- Threads the user is part of that are still actively being discussed
- Decisions or announcements in channels the user participated in that may need follow-up

**Anything explicit:**
- Messages in sessions or Slack where the user wrote something like "need to check", "ask about", "remember to", "don't forget"

Draft 1–3 "Top of mind" bullets from these signals. If nothing meaningful is found, leave it as `(fyll i)` — don't invent things. Phrase naturally, e.g. "Väntar på review av PR X", "Vet inte varför Y börjat fela — bör undersökas", "Hålla koll på deploy av Z".

### Step 9: Compile and Present

Synthesize all gathered data into a **draft** "Idag" section. Write in Swedish, using natural narrative bullet points (not technical commit-style).

**Hard limit: 3–4 bullets.** Aggressively merge related work into single bullets. The goal is a concise high-level summary, not an exhaustive changelog. Merging rules:
- All work on the same initiative/theme = one bullet (e.g. "Security v13" across 15 repos is ONE bullet, not per-repo)
- CI/infrastructure changes related to the same effort fold into that effort's bullet
- Multiple repos touched for the same reason = mention the count, not the list (e.g. "across 12 repon")
- Only split into a separate bullet if the work is genuinely unrelated to other bullets
- If you have more than 4 candidate bullets, keep merging until you're at 3–4

Examples:
- 20 commits across 15 repos for security upgrades + docker CI + lint fixes → ONE bullet: "Security v13 — pushade uppgraderingar (Go 1.26, grpc, DHI-images) och lade till docker-build-test CI i 12 repon"
- A commit "feat: add search-sfn-executions tool" + Linear NEX-796 in review → "Färdigställt PR för NEX-796 (search-sfn-executions verktyget)"
- Multiple Claude sessions about the same topic → one bullet summarizing the activity

Check the current day of week. If it's Friday, use *Måndag* instead of *Imorgon* for the next-day section.

**Slack mrkdwn formatting rules — strictly follow these:**
- Bold headings: `*text*` (single asterisk), NOT `**text**`
- Every item under a heading must be a bullet: use `•` character (not `-` or `*`)
- Indent sub-bullets with two spaces + `◦`
- Links: `<url|text>`
- No markdown headers (`#`), no code fences, no blank lines between a heading and its first bullet
- Each section heading is on its own line, followed immediately by bullets on the next line

Present the draft in this format (exactly — no extra blank lines between heading and bullets):

```
*Idag*
• {bullet 1}
• {bullet 2}
• {bullet 3}

*{Måndag if Friday, otherwise Imorgon}*
• {inferred bullet 1, or "(fyll i)" if nothing could be inferred}

*Top of mind*
• {inferred bullet from Step 7, or "(fyll i)" if nothing found}
```

After presenting the draft, ask the user:
1. Whether any bullets in "Idag" should be rephrased or removed
2. Whether there are additional things they worked on that weren't captured (meetings, investigations, discussions, Slack threads, etc.)
3. Whether the inferred "Imorgon/Måndag" bullets look right, and what else to add or change
4. Whether the "Top of mind" bullets are accurate, and if there's anything else on their mind

Then produce the final version incorporating their input.

### Step 10: Copy to Clipboard

After the user confirms the final version, copy it to the clipboard using:

```bash
pbcopy << 'EOF'
{final Slack-formatted message}
EOF
```

Confirm with: "Copied to clipboard — ready to paste into Slack."

## Notes

- Linear CLI needs `--team NEX` when not in a mapped repo directory.
- Git author is auto-detected per repo via `git config user.email`.
- Session JSONL uses `type: "user"` with content as string or list of blocks.
- If any source fails, skip gracefully and note the gap.
- Write "Idag" bullets in Swedish, casual/natural tone — like writing to teammates, not a formal report. Keep technical terms in English (e.g. "deploy", "merge", "PR", "CI", "pipeline", "endpoint", "refactor", service names, tool names, etc.) — only the surrounding narrative should be in Swedish.
- Slack activity is gathered via the Slack MCP tools (`mcp__plugin_slack_slack__slack_search_public_and_private` and `mcp__plugin_slack_slack__slack_read_thread`). User's Slack ID: `U0314GC3YRY`. If Slack is not authenticated, skip and note the gap.
