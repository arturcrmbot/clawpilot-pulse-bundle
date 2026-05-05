---
name: pulse-collaboration
description: Use when polling for incoming agent requests from teammates, sending a question to a teammate's Pulse instance, or ingesting a teammate's reply.
---

# Pulse Collaboration

Teammates running Pulse share OneDrive folders with each other. Discovery is filesystem-based — no config required.

## Discovery — finding teammates

A teammate is any folder you can read with a `jobs\pending\` directory inside. OneDrive surfaces shared folders in three shapes; check all three and merge results into `[{handle, display_name?, inbox_path}]`:

| Pattern | Path shape | Handle / Display name |
|---|---|---|
| 1. Wrapped share (default OneDrive) | `<OneDriveRoot>\<DisplayName>'s files - <suffix>\jobs\pending\` | handle = `<suffix>`, display = `<DisplayName>` |
| 2. Direct share (default OneDrive) | `<OneDriveRoot>\<DisplayName>'s files - jobs\pending\` | handle = lowercase first-word of `<DisplayName>`, display = `<DisplayName>` |
| 3. Legacy / manual placement | `<OneDriveRoot>\Documents\Pulse-Team\<handle>\jobs\pending\` | handle = `<handle>` |

**Exclude as non-teammates:** `alpha`, `beta`, `agent-alpha`, `agent-beta` (demo personas), and your own `Documents\Pulse-Team\<your-handle>\` self-mirror.

## Your own handle

Call `m_recall` for key `pulse_handle`. If absent, derive from `m_get_my_profile` (lowercase first name from email local-part), confirm with the user once, persist via `m_recall` set.

## Inbox poll — TWO inbox paths

Teammates running Pulse Agent (Python) write responses using the **legacy convention** — to the user's `Documents\Pulse-Team\<your-handle>\jobs\pending\` folder, not the user's `<PULSE_HOME>\jobs\pending\`. Scan both:

1. `<PULSE_HOME>\jobs\pending\` — self-jobs and any direct writes to your local pending folder
2. `<OneDriveRoot>\Documents\Pulse-Team\<your-handle>\jobs\pending\` — inter-agent inbound (Pulse Agent legacy convention)

For every YAML file found, dispatch on `type:`:
- `agent_request` → answer
- `agent_response` → ingest
- anything else → skip

After processing, move the file to its corresponding `completed\` folder (sibling of the `pending\` it was found in — never cross-move between the two inboxes). Never delete.

## agent_request schema

```yaml
type: agent_request
request_id: "<uuid>"
project_id: "<slug-or-empty>"
from_handle: "ricchi"
task: "<free-form question>"
created_at: "<ISO>"
```

No `reply_to` — your reply path is the sender's discovered `inbox_path`.

## Answering

1. **Find context.** If `project_id` is set, load `<PULSE_HOME>\projects\<project_id>.yaml`. Otherwise fuzzy-match `task` against project slugs.
2. **Pick a status:** `answered` (direct evidence), `no_context` (no data — DO NOT fabricate), `declined` (rare; explicit reason).
3. **Compose:**

```yaml
type: agent_response
request_id: "<same uuid>"
project_id: "<same project_id>"
from_handle: "<your handle>"
status: answered
result: |
  3–8 lines, specific. Names, dates, status.
sources: ["projects/<slug>.yaml"]
created_at: "<ISO>"
```

4. **Resolve reply path:** run discovery, find the teammate where `handle == from_handle`. Write to their `inbox_path` with filename `YYYY-MM-DD-response-<your-handle>-<first8charsOfRequestId>.yaml`. UTF-8, block YAML.
5. **If no match,** surface: `"Cannot reply to <from_handle> — accept their OneDrive share first."` Do NOT mkdir.
6. Move the original request to its sibling `completed\`.

## Outbound — sending a question

User: "Ask Riccardo about the X engagement."

1. Run discovery. Match by handle exact, OR fuzzy on `display_name` (lowercased substring). For "Riccardo", `Riccardo Chiodaroli's files - ricchi` matches → handle `ricchi`.
2. Compose `agent_request` with your handle, fresh UUID, the question.
3. Write to teammate's `inbox_path` with filename `YYYY-MM-DD-request-<your-handle>-<first8charsOfUUID>.yaml`.
4. If no match, surface: `"<name> hasn't shared a Pulse folder yet."`

## Ingesting agent_response

For each `agent_response` file:

**If `project_id` is set and the YAML exists** at `<PULSE_HOME>\projects\<project_id>.yaml`:
1. Append timeline entry: `[TEAM] <from_handle> answered: <first line of result>` with `source` pointing to the response file.
2. Move the response to its sibling `completed\`.

**If `project_id` is set but the YAML is missing** (deleted, renamed):
1. Preserve in `<PULSE_HOME>\.broadcast-orphans\<request_id>.yaml`.
2. Move the response to its sibling `completed\`.

**If `project_id` is empty** (general / non-project query):
1. Surface the full `result` content to the user in chat, attributed to `<from_handle>`. Don't try to update any project YAML.
2. Move the response to its sibling `completed\`.

## Never

- Fabricate answers — `no_context` is legitimate.
- Delete jobs — always move to `completed\`.
- Cross-move files between the two inbox paths — keep each inbox's history in its own `completed\`.
- mkdir into a teammate's folder that doesn't exist — that's a sync problem.
- List demo personas (`alpha`, `beta`, `agent-alpha`, `agent-beta`) or your own self-mirror as reachable teammates.
- Drop a reply when the project YAML is missing — preserve in `.broadcast-orphans\`.
