# Jira Daily Focus

A Cursor Agent Skill that surfaces your highest-priority Jira ticket each day, eliminating the need to manually triage your backlog. It connects to Jira via the `mcp-atlassian` MCP server and delivers a structured daily briefing directly in your editor.

## The Problem

Developers typically plan work around EPICs -> large bodies of work broken into individual tickets that span weeks or an entire quarter. When an EPIC has 10, 20, or 50 tickets, deciding *which one to pick up next* becomes a daily friction point. You open Jira, scan the board, weigh priorities, check what's blocked, and try to remember what you were working on yesterday. This context-switching tax adds up.

This skill eliminates that overhead. You point it at your quarterly EPICs, and each day it tells you exactly what to focus on — whether that's finishing an in-progress ticket or picking up the highest-priority unstarted one.

## What It Does

When invoked, the skill:

1. **Identifies your current ticket** — If you already have a ticket in progress, it shows that. The philosophy is simple: finish what you started before picking up something new.

2. **Recommends the next ticket** — If nothing is in progress (e.g. you just closed a ticket, or it's a fresh EPIC where everything is "New"), it finds the highest-priority unstarted ticket and recommends it as your next focus.

3. **Shows EPIC progress** — For each configured EPIC, it counts tickets by status and shows a completion percentage table so you can see how the quarter is tracking.

4. **Lists what's coming next** — After the current ticket, it shows the next 2–3 tickets in priority order so you have visibility into what's queued up.

The output is a formatted briefing with the ticket's summary, description, priority, story points, blockers, parent EPIC, a progress table, and an up-next queue.

## How Priority Is Resolved

When multiple tickets compete for attention, the skill uses these rules
in order:

### Static rules

| Precedence | Rule |
|-----------|------|
| 1st | In-progress tickets always win over unstarted ones |
| 2nd | Jira priority field (Highest → High → Medium → Low → Lowest) |
| 3rd | EPIC order in `config.json` (earlier = higher priority) |
| 4th | Creation date (older tickets first) |

See [PRIORITY-LOGIC.md](jira-daily-focus/PRIORITY-LOGIC.md) for the full
breakdown of the selection algorithm.

## Installation

Copy the skill files into your workspace's `.cursor/skills/` directory:

```bash
mkdir -p .cursor/skills/jira-daily-focus
cp jira-daily-focus/SKILL.md .cursor/skills/jira-daily-focus/
cp jira-daily-focus/PRIORITY-LOGIC.md .cursor/skills/jira-daily-focus/
cp jira-daily-focus/config-example.json .cursor/skills/jira-daily-focus/
```

Or, if you cloned this repo outside your project, copy the `jira-daily-focus/` contents into your project's `.cursor/skills/jira-daily-focus/` directory.

Once the files are in place, Cursor will detect the skill automatically.

## Setup

### 1. Configure the MCP server

The skill requires the `mcp-atlassian` MCP server. In `~/.cursor/mcp.json`, add:

```json
{
  "mcpServers": {
    "mcp-atlassian": {
      "command": "uvx",
      "args": ["mcp-atlassian"],
      "env": {
        "JIRA_URL": "https://your-org.atlassian.net",
        "JIRA_USERNAME": "you@example.com",
        "JIRA_API_TOKEN": "your-api-token"
      }
    }
  }
}
```

Verify the server shows a green dot in **Cursor Settings → MCP** before using the skill.

### 2. Create the config file

Copy the included example and fill in your values:

```bash
cp config-example.json config.json
```

Edit `config.json` with your project details:

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

| Field | Description |
|-------|-------------|
| `project_key` | Your Jira project key (e.g. `PROJ`, `MYTEAM`) |
| `epic_keys` | Array of EPIC issue keys for this quarter. Order matters — earlier EPICs are treated as higher priority when tickets tie. |
| `assignee` | JQL assignee value. Use `currentUser()` to match whoever is authenticated with the MCP server. |
| `quarter` | Label shown in the briefing header (purely cosmetic). |
| `jira_base_url` | Base URL of your Jira instance (e.g. `https://your-org.atlassian.net`). Used to generate clickable ticket links in the briefing. |
| `done_statuses` | Jira statuses that mean "complete". |
| `in_progress_statuses` | Jira statuses that mean "actively being worked on". |

Everything that isn't in `done_statuses` or `in_progress_statuses` is treated as "unstarted" (New, To Do, Open, Backlog, etc.).

### 3. Invoke the skill

Type `/jira-daily-focus` in Cursor chat, or ask naturally:

- "What should I work on today?"
- "Show my daily focus"
- "What's my current ticket?"
- "Quarter progress"

## File Structure

```
.cursor/skills/jira-daily-focus/
├── SKILL.md              # Skill instructions (read by the agent)
├── PRIORITY-LOGIC.md     # Priority rules and selection algorithm
├── config.json           # Your personal configuration (gitignored if needed)
├── config-example.json   # Example config for reference
└── README.md             # This file
```

## Updating for a New Quarter

When the quarter rolls over or your EPICs change:

1. Update `epic_keys` in `config.json` with the new EPIC keys
2. Update the `quarter` label
3. Run the skill again — it picks up the new config immediately

## Limitations

- Requires the `mcp-atlassian` MCP server to be connected and authenticated
- Pagination means very large EPICs (50+ tickets) require multiple API calls
- The skill reads Jira's priority field as-is — if your team doesn't set priorities, all tickets will tie and fall back to creation date ordering
- Story points are shown if available but don't affect priority resolution
