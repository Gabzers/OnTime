# Sessions — Working Memory Across Time

Hub: [[OnTimeCRM]] · Maintenance: [[HOW-TO-USE]].

A **session log** captures what happened in a work session so the next one (you, Claude, or the
local model) resumes with context instead of re-deriving it. Logs live in `03-SESSIONS/`.
They are the project's episodic memory; the rest of the vault is its reference knowledge.

## When to write one
- End of any session that produced decisions, fixes, or non-obvious findings.
- Before a long break, or before `/compact` in Claude Code (context will be cleared).
- After finishing a [[ROADMAP]] item or a chunk of a [[KNOWN-ISSUES]] entry.

Skip it for trivial one-off edits — not every session needs a log.

## How to create one
- **In Claude Code:** run `/compress` (if the CPR skills are installed) — it writes a structured
  log here automatically. Otherwise copy `03-SESSIONS/_TEMPLATE.md`.
- **Filename:** `YYYY-MM-DD-HH_MM-topic.md` (e.g. `2026-06-01-09_30-fix-snooze-field.md`). Sortable, searchable.

## What goes in a log (vs other notes)
| Goes in a session log | Goes elsewhere |
|-----------------------|----------------|
| What changed this session, why, what's next | Durable rules → [[CONVENTIONS]] |
| Findings/gotchas discovered | Entities/enums → [[DOMAIN]] |
| Dead ends and what didn't work | Architecture choices → `04-DECISIONS/` |
| Open questions for next time | Bugs to fix → [[KNOWN-ISSUES]] |

A session log is a snapshot in time — it is NOT updated later. If something in it becomes a
durable fact or rule, **promote it** to the right reference note and link back.

## Resuming
- **In Claude Code:** `/resume` loads `CLAUDE.md` + recent log summaries.
- **Manually / local model:** open the latest `03-SESSIONS/*` plus the [[ROADMAP]] to see where you left off.
- Logs are designed so only the summary header needs reading; the raw transcript (if any) sits below a marker and is for search, not for routine context.

## Hygiene
- One log per session; keep summaries tight.
- Cross-link: a log should link the [[ROADMAP]] item, [[KNOWN-ISSUES]] entry, or decision it relates to.
- Raw transcripts can get large — never load them wholesale into the local model; `@file` only the summary.

## Related
[[HOW-TO-USE]] · [[ROADMAP]] · [[KNOWN-ISSUES]] · [[OnTimeCRM]]
