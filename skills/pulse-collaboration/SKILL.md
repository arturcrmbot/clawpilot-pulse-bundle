---
name: pulse-collaboration
description: Use when the user wants to message a teammate's Pulse, see who they can reach, check incoming questions, or ingest a teammate's reply. Also handles the every-5-min poll automation.
---

# Pulse Collaboration

Inter-agent messaging via OneDrive shared folders. Process inboxes, surface what was found, enrich project memory with what other agents know.

## Critical: never produce empty output

Clawpilot's automation runner treats empty agent output as a failure. **Always emit at least one line of output**, even on a routine no-op poll. Use the minimal one-liner format described in "Output" below.

## Setup — share only `pending/`

Teammates write `agent_request` YAMLs into your inbox. They never need to read your responses or history.

**Share only `<PULSE_HOME>\jobs\pending\` with teammates** — never its parent. Sharing `jobs\` or any wrapping `<handle>\` folder exposes your `completed\` archive, which contains historical messages with PII and confidential business context. That's a leak.

In OneDrive web:
1. Navigate to `<PULSE_HOME>\jobs\pending\`.
2. Right-click → Share → grant **Can edit** to each teammate by name (they need to write).
3. Accept their shares the same way — they should be sharing only their `pending\` folder.

If a teammate has over-shared (you see their `completed\` folder when you accept their share), tell them — they should re-share only `pending\`.

## Behavior — on every invocation

Do these in order. Each step is fault-tolerant: a missing folder, a tool error, or 0 results is handled silently and the next step proceeds.

1. **Resolve your handle.** `m_recall` for `pulse_handle`. If unset, derive from `m_get_my_profile` (lowercase first-name from email local-part), persist via `m_recall set pulse_handle <value>`. Never block waiting for user input.

2. **Process both inboxes.** Each independent; missing or empty is fine:
   - `<PULSE_HOME>\jobs\pending\` — your local inbox
   - `<OneDriveRoot>\Documents\Pulse-Team\<your-handle>\jobs\pending\` — your team-shared inbox (legacy Pulse Agent convention)

   For each `*.yaml`:
   - `type: agent_request` → answer (see Answering), move to sibling `completed/`
   - `type: agent_response` → ingest (see Ingesting), move to sibling `completed/`
   - anything else → skip

3. **Discover teammates.** Scan OneDrive for shared folders that contain inbound message YAMLs. Build `[{handle, display_name?, inbox_path, over_shared?}]` from these shapes:

   | Pattern | Path shape | Handle source | Notes |
   |---|---|---|---|
   | **A — recommended (pending-only share)** | `<OneDriveRoot>\<DisplayName>'s files - pending\` | lowercased first-word of `<DisplayName>` | Privacy-clean. Only `pending/` is shared. |
   | B | `<OneDriveRoot>\<DisplayName>'s files - <suffix>\jobs\pending\` | `<suffix>` (the handle) | Sender exposed `completed/`. Mark `over_shared: true`. |
   | C | `<OneDriveRoot>\<DisplayName>'s files - jobs\pending\` | lowercased first-word of `<DisplayName>` | Sender exposed `completed/`. Mark `over_shared: true`. |
   | D | `<OneDriveRoot>\Documents\Pulse-Team\<handle>\jobs\pending\` | `<handle>` | Manual placement. May or may not expose `completed/` depending on what they shared. Mark `over_shared: true` if a sibling `completed\` is visible. |

   `inbox_path` is the resolved `pending/` directory. The skill writes to that path regardless of pattern.

   Exclude `alpha`, `beta`, `agent-alpha`, `agent-beta` (demo personas) and your own self-mirror.

   When emitting the routine status output (see Output), if any teammate has `over_shared: true`, append a one-line note: `⚠ <handle> has over-shared (their completed/ is visible) — ask them to re-share only pending/.`

4. **Emit output** per the format below.

## Schemas

```yaml
# agent_request
type: agent_request
request_id: "<uuid>"
project_id: "<slug-or-empty>"   # empty = general/non-project query
from_handle: "jane"
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

For each `agent_response`:

| `project_id` | Action |
|---|---|
| Set + project YAML exists | Update `team_context` (see below). Append timeline entry: `[TEAM] <from_handle> enriched team_context: <one-line summary>` |
| Set but project YAML missing | Preserve in `<PULSE_HOME>\.broadcast-orphans\<request_id>.yaml` |
| Empty | Surface full `result` in chat output, attributed to `<from_handle>`. No project update. |

### Updating `team_context`

Find or create the entry where `from == from_handle` (replace existing — only the latest context per teammate per project matters). Set:

- **`relevance`**:
  - `high` — status is `answered` and `result` has substantive overlap (specific projects, names, commitments, or assets relevant to this engagement)
  - `low` — status is `answered` but tangential
  - `none` — status is `no_context`
- **`summary`** — 1–3 line distillation of `result` (or `"No active work on this"` for `none`)
- **`last_updated`** — today

Cap `team_context` at 5 entries — drop the oldest by `last_updated` if exceeded.

## Outbound — sending a question

User: *"Ask Jane about the X engagement."* (Or auto-triggered by `/pulse-projects` on new project creation.)

1. Discovery — match by handle exact, or fuzzy substring on `display_name` (lowercased).
2. Compose `agent_request` with your handle, fresh UUID, the question.
3. Write to teammate's `inbox_path`. Filename: `YYYY-MM-DD-request-<your-handle>-<first8charsOfUUID>.yaml`.
4. If no match, emit: `<name> hasn't shared a Pulse folder yet.` Skip the write.

## Output

**Always emit ≥1 line. Never empty.**

### Activity present (anything was processed, or the user asked a specific question)

```
📥 Inbox: 2 processed
  ✓ Answered Jane (project: acme-cloud-migration) — 4 sources cited
  ✓ Enriched team_context on contoso-migration (from Bob: high — has shared assets)

[If any agent_response had empty project_id, dump its result here under a "From <handle>:" header]
[If any teammate is over_shared, note: ⚠ <handle> has over-shared their jobs/ folder — ask them to re-share only pending/.]
```

### No activity (routine 5-min poll, nothing new)

A single brief line, e.g.:
```
Inbox quiet. 2 teammates reachable.
```

This is non-empty → automation marks success → no notification spam.

### Outbound action (user asked to send)

```
✉ Sent → Jane: "<one-line task summary>"
```

## Never

- Fabricate answers — `no_context` is legitimate.
- Delete jobs — always move to sibling `completed/`.
- Cross-move files between the two inbox paths.
- mkdir into a folder that doesn't exist — that's a sync problem.
- Block waiting for user input during automation runs.
- **Produce empty output.** Clawpilot's automation runner treats empty as failure.
- List demo personas (`alpha`, `beta`, `agent-alpha`, `agent-beta`) or your own self-mirror as reachable.
- Track outbound request_ids or pending-inquiry metadata in user-visible output. The project YAML's `team_context` is the source of truth for team enrichment; the messaging plumbing is invisible.
- **Tell users to share `jobs\` or any wrapping `<handle>\` folder.** Only `pending\` should be shared; the rest is private.
