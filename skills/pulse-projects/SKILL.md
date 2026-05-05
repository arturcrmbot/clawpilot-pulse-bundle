---
name: pulse-projects
description: Use when the user mentions a customer, deal, engagement, initiative, or stakeholder by name; describes a meeting/email/conversation that affects a known engagement; or asks about commitments, status, or who-said-what for named work.
---

# Pulse Projects

Persistent project memory: one YAML file per customer engagement, stored in OneDrive. The schema, the file location, and the rules for what's worth tracking.

## Path

Resolve `<PULSE_HOME>` once via shell:

```powershell
if ($env:OneDriveCommercial) { "$env:OneDriveCommercial\Documents\Pulse" } elseif ($env:OneDrive) { "$env:OneDrive\Documents\Pulse" } else { "$env:USERPROFILE\Documents\Pulse" }
```

Files live at `<PULSE_HOME>\projects\<slug>.yaml`. Slug is lowercase-hyphenated. **One file per customer** — workshops, reviews, sub-meetings are timeline entries, never separate files.

## Schema

```yaml
project: "Human-readable name"          # required
involvement: lead                        # lead | contributor | observer (default: observer)
status: active                           # active | blocked | on-hold | completed
risk_level: medium                       # low | medium | high | critical
summary: "1–2 sentences current state"
last_verified: "2026-05-05"
stakeholders:                            # max 6
  - {name, role, org?, last_interaction?}
commitments:
  - what: "Send pricing proposal"
    who: "You" | "<name>"
    to: "<recipient>"
    due: "2026-05-12"                    # YYYY-MM-DD, ONLY if explicitly stated
    due_confidence: explicit             # explicit | inferred
    status: open                         # open | done | overdue | cancelled
    source: "<artifact reference>"
timeline:                                # max 10 entries
  - {date, event, source}
related_artifacts:                       # max 6
  - {type, path, description}
watch_queries: ["customer", "person"]    # max 3
tags: ["industry", "tech"]               # max 4
notes: |
  Free-form context.
updated_at: "<ISO timestamp>"            # set on every write
```

## Discovery — when to create a NEW project

ALL THREE conditions must be met:
1. **3+ mentions** of the customer/initiative across user data
2. **2+ different source types** (transcript + email — not two emails from the same thread)
3. **At least one actionable element** — commitment, deliverable, meeting series, deadline, or explicit ask directed at the user

Passive mentions and CC'd emails do NOT meet the threshold.

Before creating, search existing slugs. If a file already exists, **update it**. Default `involvement: observer` unless the user clearly owns it.

## Read-before-write

Always: list directory → read existing file → merge new info → apply curation limits → set `updated_at` → write the full file. Never overwrite blindly.

## Curation limits (HARD)

Stakeholders ≤6 · Timeline ≤10 · Tags ≤4 · Watch queries ≤3 · Artifacts ≤6 · Summary ≤2 sentences. When exceeded, **prune oldest low-value entries first** (digest mentions, repeated flags) — never the newest.

## Commitment lifecycle

`open` + `due` past today + `due_confidence: explicit` → `overdue`. Inferred dates **never** trigger overdue (false alarms erode trust). `overdue` >5 days unresolved → `cancelled` with `cancelled_reason`. Mark `done` with completion source when fulfilled.

## Auto-archive observer projects

`involvement: observer` + `status: active` + no activity in 14+ days → set `status: completed` with timeline entry `[AUTO-ARCHIVED] No activity in 14+ days and involvement is observer`. Observer projects must prove they're worth tracking.

## Never

- Invent dates, names, amounts, or commitments — only persist what's explicit in source
- Create a second file for the same customer
- Mark `lead` from a single email or meeting attendance
- Treat `inferred` due dates as overdue
- Delete history when uncertain — add a `[STALE]` or `[INFO]` timeline entry instead
