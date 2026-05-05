---
name: pulse-collaboration
description: Use when the user wants to message a teammate's Pulse, see who they can reach, check incoming questions, or ingest a teammate's reply. Also handles the every-5-min poll automation.
---

# Pulse Collaboration

Inter-agent messaging via OneDrive shared folders. Process inboxes, surface what was found.

## Critical: never produce empty output

Clawpilot's automation runner treats empty agent output as a failure. **Always emit at least one line of output**, even on a routine no-op poll. Use the minimal one-liner format described in "Output" below.

## Behavior — on every invocation

Do these in order. Each step is fault-tolerant: a missing folder, a tool error, or 0 results is handled silently and the next step proceeds.

1. **Resolve your handle.** `m_recall` for `pulse_handle`. If unset, derive from `m_get_my_profile` (lowercase first-name from email local-part), persist via `m_recall set pulse_handle <value>`. Never block waiting for user input — automation can't answer.

2. **Process both inboxes.** Each independent; missing or empty is fine:
   - `<PULSE_HOME>\jobs\pending\` — your local inbox
   - `<OneDriveRoot>\Documents\Pulse-Team\<your-handle>\jobs\pending\` — your team-shared inbox (legacy Pulse Agent convention)

   For each `*.yaml` file in either:
   - `type: agent_request` → answer (see Answering), move to sibling `completed/`
   - `type: agent_response` → ingest (see Ingesting), move to sibling `completed/`
   - anything else → skip

3. **Discover teammates.** Scan OneDrive for folders containing `jobs\pending\`. Build `[{handle, display_name?, inbox_path}]` from these patterns:
   - `<OneDriveRoot>\<DisplayName>'s files - <suffix>\jobs\pending\` → handle = `<suffix>`, display = `<DisplayName>`
   - `<OneDriveRoot>\<DisplayName>'s files - jobs\pending\` → handle = lowercased first-word of display, display = `<DisplayName>`
   - `<OneDriveRoot>\Documents\Pulse-Team\<handle>\jobs\pending\` → handle = `<handle>`

   Exclude `alpha`, `beta`, `agent-alpha`, `agent-beta` (demo personas) and your own self-mirror.

4. **Emit output** per the format below.

## Schemas

```yaml
# agent_request
type: agent_request
request_id: "<uuid>"
project_id: "<slug-or-empty>"   # empty = general/non-project query
from_handle: "ricchi"
task: "<free-form question>"
created_at: "<ISO>"

# agent_response
type: agent_response
request_id: "<same uuid>"
project_id: "<same project_id>"
from_handle: "<your handle>"
status: answered                # answered | no_context | declined
result: |
  3–8 lines, specific. Names, dates, status.
sources: ["projects/<slug>.yaml"]
created_at: "<ISO>"
```

No `reply_to` field — sender knows where you'll write back (their discovered `inbox_path`).

## Answering

1. **Find context.** If `project_id` set: load `<PULSE_HOME>\projects\<project_id>.yaml`. Else fuzzy-match `task` against project slugs.
2. **Pick status:** `answered` (direct evidence), `no_context` (no relevant data — DO NOT fabricate), `declined` (explicit reason; rare).
3. **Compose** the response YAML.
4. **Write to recipient.** Find teammate from discovery where `handle == from_handle`. Write to their `inbox_path` with filename `YYYY-MM-DD-response-<your-handle>-<first8charsOfRequestId>.yaml`. UTF-8, block YAML.
5. **If no discovery match,** flag in output: `⚠ Cannot reply to <from_handle> — accept their OneDrive share.` Do NOT mkdir.
6. **Move** the original request to sibling `completed/`.

## Ingesting

| `project_id` | Action |
|---|---|
| Set + YAML exists | Append timeline entry to project YAML: `[TEAM] <from_handle> answered: <first line of result>` with `source` pointing to the response file |
| Set but YAML missing | Preserve in `<PULSE_HOME>\.broadcast-orphans\<request_id>.yaml` |
| Empty | Surface full `result` in chat output, attributed to `<from_handle>`. No project update. |

## Outbound — sending a question

User: *"Ask Riccardo about the X engagement."*

1. Discovery — match by handle exact, or fuzzy substring on `display_name` (lowercased).
2. Compose `agent_request` with your handle, fresh UUID, the question.
3. Write to teammate's `inbox_path`. Filename: `YYYY-MM-DD-request-<your-handle>-<first8charsOfUUID>.yaml`.
4. If no match, emit: `<name> hasn't shared a Pulse folder yet.` Skip the write.

## Output

**Always emit ≥1 line. Never empty.**

### Activity present (anything was processed, or the user asked a specific question)

```
📥 Inbox: 2 processed
  ✓ Answered Jane (project: acme-corp) — 4 sources cited
  ✓ Ingested response from Bob → general (surfaced below)

📤 Outstanding (sent, no reply yet):
  • Sasa — sent 2026-05-05 14:34

👥 Reachable: ricchi, sasa

[If any agent_response had empty project_id, dump its result here under a "From <handle>:" header]
```

### No activity (routine 5-min poll, nothing new)

A single line, e.g.:
```
Inbox quiet. 2 teammates reachable (ricchi, sasa). 1 outstanding sent.
```

This is non-empty → automation marks success → user gets no notification spam.

### Outbound action (user asked to send)

```
✉ Sent → Riccardo: "<one-line task summary>" (request_id 7f3a...)
```

## Never

- Fabricate answers — `no_context` is legitimate.
- Delete jobs — always move to sibling `completed/`.
- Cross-move files between the two inbox paths.
- mkdir into a folder that doesn't exist — that's a sync problem.
- Block waiting for user input during automation runs.
- **Produce empty output.** Clawpilot's automation runner treats empty as failure.
- List demo personas (`alpha`, `beta`, `agent-alpha`, `agent-beta`) or your own self-mirror as reachable.
