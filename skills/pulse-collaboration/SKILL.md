---
name: pulse-collaboration
description: Use when the user wants to message a teammate's Pulse, see who they can reach, check incoming questions, or ingest a teammate's reply. Also fires for the every-5-min poll automation.
---

# Pulse Collaboration

Inter-agent messaging via OneDrive shared folders. Each teammate runs their own Pulse; questions and answers flow as YAML files between shared folders.

## Behavior

**On every invocation, do all three:**

1. **Scan inboxes**, process new YAML files (see "Inbox processing"), and update what was found.
2. **Discover teammates** by scanning OneDrive for any folder containing `jobs\pending\` (see "Discovery").
3. **Output** a compact status report (see "Output").

If the user asks something specific (e.g., "ask Jane about X"), do that as the outbound action and skip the routine status report.

## Inbox processing

Two inbox locations to scan. Both belong to the user; teammates may write to either depending on which Pulse they run.

- **Your local inbox**: `<PULSE_HOME>\jobs\pending\`
- **Your team inbox** (legacy convention used by Pulse Agent Python daemon): `<OneDriveRoot>\Documents\Pulse-Team\<your-handle>\jobs\pending\`

For every `*.yaml` in either folder, dispatch on `type:`:

| `type:` | Action |
|---|---|
| `agent_request` | Answer it (see "Answering"), write the response to the sender's inbox, then move the request to its sibling `completed/` |
| `agent_response` | Ingest it (see "Ingesting"), then move it to its sibling `completed/` |
| anything else | Skip (other Pulse Agent job types) |

Files always move to the `completed/` folder that's a sibling of the `pending/` they were found in. Never cross-move between the two inboxes. Never delete.

## Discovery

A teammate is anyone who has shared a OneDrive folder where you can read a `jobs\pending\` directory. OneDrive lays shared folders down in three shapes; check all three:

```
<OneDriveRoot>\<DisplayName>'s files - <suffix>\jobs\pending\    → handle = <suffix>
<OneDriveRoot>\<DisplayName>'s files - jobs\jobs\pending\         → handle = lowercased first word of <DisplayName>
<OneDriveRoot>\<DisplayName>'s files - jobs\pending\              → handle = lowercased first word of <DisplayName> (Sasa-style direct share)
<OneDriveRoot>\Documents\Pulse-Team\<handle>\jobs\pending\        → handle = <handle> (legacy/manual)
```

Return `[{handle, display_name?, inbox_path}]`. Exclude `alpha`, `beta`, `agent-alpha`, `agent-beta` (demo personas) and your own self-mirror under `Documents\Pulse-Team\<your-handle>\`.

## Your handle

Resolve once via `m_recall` for `pulse_handle`. If absent, derive from `m_get_my_profile` (lowercase first name from email local-part), confirm with the user once, persist via `m_recall` set.

## Schemas

```yaml
# agent_request
type: agent_request
request_id: "<uuid>"
project_id: "<slug-or-empty>"   # empty = general/non-project
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

No `reply_to` field — sender knows where you'll reply (they wrote into your inbox; you write into theirs at their discovered `inbox_path`).

## Answering

1. **Find context.** If `project_id` is set, load `<PULSE_HOME>\projects\<project_id>.yaml`. Else fuzzy-match `task` against project slugs.
2. **Pick `status`:** `answered` (direct evidence), `no_context` (no relevant data — DO NOT fabricate), `declined` (rare; explicit reason).
3. **Compose the response YAML.**
4. **Resolve recipient:** discovery gives you each teammate's `inbox_path`. Find the one where `handle == from_handle`. Filename: `YYYY-MM-DD-response-<your-handle>-<first8charsOfRequestId>.yaml`. UTF-8, block YAML.
5. **If no match,** surface to user: `"Cannot reply to <from_handle> — accept their OneDrive share first."` Do NOT mkdir.

## Ingesting

For each `agent_response`:

| `project_id` | Action |
|---|---|
| Set + matching YAML exists | Append timeline entry to that YAML: `[TEAM] <from_handle> answered: <first line of result>` with `source` pointing to the response file |
| Set but YAML missing | Preserve in `<PULSE_HOME>\.broadcast-orphans\<request_id>.yaml` |
| Empty | Surface the full `result` to the user in chat, attributed to `<from_handle>`. No project update. |

## Outbound — sending a question

User: "Ask Riccardo about the X engagement."

1. Run discovery. Match by handle exact, OR fuzzy on `display_name` (lowercased substring). For "Riccardo", `Riccardo Chiodaroli's files - ricchi` → handle `ricchi`.
2. Compose `agent_request` with your handle, fresh UUID, the user's question.
3. Write to that teammate's `inbox_path`. Filename: `YYYY-MM-DD-request-<your-handle>-<first8charsOfUUID>.yaml`.
4. If no match: `"<name> hasn't shared a Pulse folder yet."`

## Output (status report)

When invoked without a specific outbound request, format like:

```
📥 Inbox: 2 new requests, 1 response
✓ Answered Jane (project: acme-corp) — 4 sources cited
✓ Answered Bob (general) — status: no_context
✓ Ingested response from Jane → updated acme-corp timeline

📤 Outstanding (sent, no reply yet):
  • request-artur-da6b62c4 → Sasa (sent 2026-05-05 14:34)

👥 Reachable teammates:
  ricchi (Riccardo Chiodaroli)
  sasa (Sasa Juratovic)
```

When running on automation, **stay completely silent** if no requests/responses processed and no outstanding items have aged past 24h.

## Never

- Fabricate answers. `no_context` is legitimate.
- Delete jobs. Always move to `completed/`.
- Cross-move files between the two inbox paths.
- mkdir into a teammate's folder that doesn't exist — that's a sync problem, not a routing problem.
- List demo personas or your own self-mirror as reachable teammates.
- Drop a reply when the project YAML is missing — preserve in `.broadcast-orphans/`.
