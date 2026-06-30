---
name: jira-daily-focus
description: >-
  Show the developer's current priority Jira ticket from their quarterly EPICs.
  Determines what to work on today based on ticket status and priority order.
  Use when the user asks what to work on, daily focus, next ticket, current
  ticket, quarterly progress, standup, or priority.
disable-model-invocation: true
---

# Jira Daily Focus

Surface the developer's highest-priority ticket from their quarterly EPICs.
If a ticket is already in-progress, show that. Otherwise, find the
highest-priority unstarted ticket (e.g. "New") and recommend it as
the next one to pick up. This covers fresh EPICs where all tickets
start as "New" — the skill picks the most important one to begin with.

## Prerequisites

The `mcp-atlassian` MCP server must be configured and connected (green dot
in Cursor Settings > MCP). If it is not connected, tell the user:

> "The Jira MCP server is not connected. Open Settings > MCP and verify
> that mcp-atlassian shows a green dot. If not, check your JIRA_URL,
> JIRA_USERNAME, and JIRA_API_TOKEN in ~/.cursor/mcp.json."

Then stop.

## Step 0 — Load configuration and learned context

Read **both** of these files from `.cursor/skills/jira-daily-focus/`
(in the workspace root):

1. `config.json`
2. `PRIORITY-LOGIC.md`

`config.json` should contain:

```json
{
  "project_key": "PROJ",
  "epic_keys": ["PROJ-100", "PROJ-200", "PROJ-300"],
  "assignee": "currentUser()",
  "quarter": "Q3-2026",
  "jira_base_url": "https://your-org.atlassian.net",
  "done_statuses": ["Done", "Closed", "Resolved"],
  "in_progress_statuses": ["In Progress", "Review"]
}
```

**If config.json does not exist**, ask the user:

1. What is your Jira project key? (e.g. "PROJ")
2. What are your EPIC keys for this quarter? (e.g. "PROJ-100, PROJ-200")
3. What quarter is this? (e.g. "Q3-2026")

Then create `config.json` with their answers (use the default status lists
above). Confirm the file was created and proceed.

From `PRIORITY-LOGIC.md`, pay attention to:
- **Learned Preferences** (Section 2) — these override or supplement the
  static rules when choosing which ticket to surface. Apply them.
- **Decision Log** (Section 3) — scan recent entries for context on what
  the user has been working on, any overrides, and feedback patterns.

This context must inform your ticket selection in Steps 1-2.

## Step 1 — Find in-progress tickets

Use the MCP tool `jira_search` with this JQL:

```
project = {project_key} AND assignee = {assignee} AND status in ("{in_progress_statuses joined by '", "'}") AND parentEpic in ({epic_keys joined by ', '}) ORDER BY priority ASC, created ASC
```

Parameters:
- `jql`: the query above with values substituted from config
- `limit`: 5

**If results are returned:** The first result is the **current ticket**.
Skip to Step 3.

**If multiple in-progress tickets exist:** List all of them, highlight the
highest-priority one, and note:

> "You have multiple tickets in progress. Consider focusing on one at a
> time. Showing the highest-priority one below."

## Step 2 — Find next unstarted ticket

No in-progress tickets found. This is normal when starting a fresh EPIC
(all tickets are "New") or when the developer just finished a ticket.

Use `jira_search` with:

```
project = {project_key} AND assignee = {assignee} AND status NOT IN ("{done_statuses + in_progress_statuses joined by '", "'}") AND parentEpic in ({epic_keys joined by ', '}) ORDER BY priority ASC, created ASC
```

Parameters:
- `jql`: the query above
- `limit`: 5

This finds tickets in statuses like "New", "To Do", "Open", "Backlog" —
anything that is neither done nor in-progress. These are the candidates
for the developer's next piece of work.

**If results are returned:** The first result is the **recommended next
ticket** — the highest-priority unstarted ticket. Proceed to Step 3.

**If no results:** All EPIC tickets are either done or unassigned. Report:

> "All your assigned tickets in these EPICs are complete! Update
> config.json with new EPIC keys or check with your lead for new work."

Then stop.

## Step 3 — Gather full ticket details

Use the MCP tool `jira_get_issue` with:
- `issue_key`: the key of the ticket identified in Step 1 or 2

Extract:
- Summary, description (first 3-4 lines or acceptance criteria)
- Priority, status, story points (if available)
- Linked issues where link type is "is blocked by" (these are blockers)
- Parent epic key and summary

## Step 4 — Gather EPIC progress

For each EPIC in `epic_keys`, use `jira_search` with:

```
parentEpic = {epic_key}
```

Parameters:
- `jql`: as above
- `limit`: 50

From the results, count:
- **Done**: tickets with status in `done_statuses`
- **In Progress**: tickets with status in `in_progress_statuses`
- **Remaining**: everything else

Calculate percentage: `done / total * 100`

## Step 5 — Also fetch "Up Next" list

From the Step 2 query results (or run it now if skipped), take the next
2-3 tickets after the current one. These form the "Up Next" list.

## Step 6 — Present the briefing

Format the output as shown below.

**If the ticket came from Step 1** (already in-progress), use the heading
"Currently Working On". **If it came from Step 2** (unstarted), use the
heading "Recommended Next Ticket" to make it clear this is a suggestion.

Construct ticket links using `{jira_base_url}/browse/{ticket_key}` from
the config. Every ticket key in the briefing should be a clickable
markdown link.

```
## Daily Focus — {today's date}

### Currently Working On / Recommended Next Ticket
**[{ticket_key}]({jira_base_url}/browse/{ticket_key}) {summary}**
Status: {status} | Priority: {priority} | Points: {points or "—"}
Epic: [{parent_epic_key}]({jira_base_url}/browse/{parent_epic_key}) — {parent_epic_summary}

**Description:**
{first 3-4 lines of description or acceptance criteria}

**Blockers:**
{list of blocking issues as "[KEY](link) summary", or "None"}

---

### Quarter Progress ({quarter})
| Epic | Done | In Progress | Remaining | % Complete |
|------|------|-------------|-----------|------------|
| [{epic_key}]({jira_base_url}/browse/{epic_key}): {epic_summary} | {done} | {in_progress} | {remaining} | {pct}% |

### Up Next (after current ticket)
1. [{key}]({jira_base_url}/browse/{key}) {summary} ({priority})
2. [{key}]({jira_base_url}/browse/{key}) {summary} ({priority})
3. [{key}]({jira_base_url}/browse/{key}) {summary} ({priority})
```

## Step 7 — Record the decision (learning)

After presenting the briefing, append a new entry to the **Decision Log**
section (Section 3) of `PRIORITY-LOGIC.md`. Write it directly after the
`<!-- DECISION-LOG-START -->` marker (before any older entries), so the
most recent entry is always on top.

Use this format:

```
### {today's date}
- **Surfaced:** [{ticket_key}] {summary} ({priority}, {status})
- **Reason:** {1-2 sentences on why this ticket won — cite the static
  rule or learned preference that applied}
- **Alternatives:** {list 2-3 other candidates with their priorities}
- **User feedback:** (pending)
- **Notes:** {anything notable — blockers cleared, EPIC nearing
  completion, multiple in-progress tickets, etc.}
```

Set "User feedback" to `(pending)` initially.

**If the user responds** with feedback after seeing the briefing (e.g.
"no, I'll work on X instead", or "yes that's right", or asks a question
about priority), update the latest entry's "User feedback" field:
- `accepted` — user agreed with the recommendation
- `overrode → [TICKET-KEY] reason: "{user's reason}"` — user chose
  a different ticket; record which one and why
- `no response` — user moved on without commenting on the choice

**Synthesize preferences** when the log has 5+ entries: review all
entries and update the "Learned Preferences" section (between the
`<!-- LEARNED-PREFERENCES-START -->` and `<!-- LEARNED-PREFERENCES-END -->`
markers) with any patterns you observe. Keep doing this every 5 new
entries.

## Priority resolution

For details on how priority is determined when multiple tickets compete,
see [PRIORITY-LOGIC.md](PRIORITY-LOGIC.md).
