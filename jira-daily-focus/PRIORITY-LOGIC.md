# Priority Resolution Logic

How the skill determines which ticket to surface when multiple candidates
exist. This file has three parts:

1. **Static rules** — the baseline algorithm (rarely changes)
2. **Learned preferences** — patterns synthesized from the decision log
   (updated periodically by the skill)
3. **Decision log** — raw record of every invocation (appended each run)

The skill MUST read this entire file before making a priority decision.
Learned preferences override static rules when they conflict.

---

## 1. Static Rules

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

---

## 2. Learned Preferences

These preferences are synthesized from the decision log below. The skill
should apply them as overrides or tie-breakers on top of the static rules.

When the decision log accumulates 5+ entries, the skill should review the
log and update this section to reflect observed patterns. Replace the
placeholder below with actual learned rules.

> **No preferences learned yet.** This section will be populated after
> several skill invocations build up the decision log below.

<!-- LEARNED-PREFERENCES-START -->
<!-- LEARNED-PREFERENCES-END -->

### What counts as a learned preference

- **EPIC weighting:** "User consistently picks EPIC-A tickets over EPIC-B
  even when EPIC-B has higher Jira priority" → add a preference to
  weight EPIC-A higher.
- **Status preferences:** "User prefers finishing Review tickets before
  picking up new In Progress work" → add a preference to rank Review
  above In Progress.
- **Ticket-type affinity:** "User tends to pick bug fixes over stories
  when both are available" → note the pattern.
- **Time-of-week patterns:** "User prefers lighter tickets on Fridays"
  → note if observed.
- **Override patterns:** "User frequently overrides the recommendation
  to work on a different ticket" → record what they actually chose and
  why.

### Format for learned preferences

Each preference should be a short rule with a confidence note:

```
- [HIGH confidence, 6/8 entries] Prefer "Review" tickets over new
  "In Progress" tickets — user consistently finishes reviews first.
- [MEDIUM confidence, 3/5 entries] EPIC PROJ-100 work is preferred on
  Mondays/Tuesdays (nightly test triage cycle).
```

---

## 3. Decision Log

Each skill invocation appends one entry here. The format is:

```
### YYYY-MM-DD
- **Surfaced:** [TICKET-KEY] summary (priority, status)
- **Reason:** Why this ticket was chosen over alternatives
- **Alternatives:** Other candidates that were considered
- **User feedback:** What the user said or did after seeing the briefing
  (accepted / overrode with TICKET-KEY / no response)
- **Notes:** Any context that might help future decisions
```

The skill MUST append a new entry after presenting the briefing. If the
user provides feedback (e.g. "no, I want to work on X instead"), update
the latest entry's "User feedback" field.

When this log reaches 10+ entries, the skill should synthesize patterns
into the "Learned Preferences" section above and may archive older
entries by moving them to a `<!-- LOG-ARCHIVE ... -->` comment block to
keep the file readable.

<!-- DECISION-LOG-START -->
