# Priority Resolution Logic

How the skill determines which ticket to surface when multiple candidates
exist.

The skill MUST read this file before making a priority decision.

---

## Static Rules

### Selection hierarchy

1. **In-progress tickets always win.** If the developer already has work
   in flight, that work takes precedence. Context-switching is expensive;
   finish what you started.

2. **Among in-progress tickets**, tie-break by:
   - Jira priority field (Highest > High > Medium > Low > Lowest)
   - Creation date (older first — older work has been waiting longer)

3. **Among to-do tickets** (when nothing is in-progress), tie-break by:
   - Jira priority field
   - EPIC order as listed in `config.json` (first EPIC = highest priority)
   - Creation date within the same EPIC and priority bucket

### How the JQL achieves this

The `ORDER BY priority ASC, created ASC` clause in JQL handles the first
two levels. Jira's priority field maps to numeric values where lower =
higher priority:

| Priority | Jira internal value |
|----------|---------------------|
| Highest  | 1                   |
| High     | 2                   |
| Medium   | 3                   |
| Low      | 4                   |
| Lowest   | 5                   |

So `ORDER BY priority ASC` puts Highest-priority tickets first. The
`created ASC` secondary sort ensures older tickets surface before newer
ones at the same priority level.

EPIC order is handled by the skill: if the JQL returns tickets from
multiple EPICs at the same priority, the skill prefers the ticket whose
parent EPIC appears earlier in the `epic_keys` array in config.json.

### Status mapping

The config defines three buckets via two lists:

| Bucket      | Statuses (from config)                           |
|-------------|--------------------------------------------------|
| Done        | `done_statuses` — e.g. Done, Closed, Resolved   |
| In Progress | `in_progress_statuses` — e.g. In Progress, Review |
| Unstarted   | Everything else (New, To Do, Open, Backlog, etc.) |

"New" is the typical starting status for fresh EPIC tickets. When all
tickets in an EPIC are "New", the skill picks the highest-priority one
and recommends it as the first ticket to start working on.

### Edge cases

- **Multiple in-progress tickets:** list all, highlight highest-priority,
  nudge to focus on one.
- **No tickets assigned:** suggest checking config and Jira assignments.
- **All tickets complete:** congratulate, suggest updating config.json.
- **No story points:** show "—"; points don't affect priority.
- **EPIC has zero tickets:** show "0 total" in progress table.
