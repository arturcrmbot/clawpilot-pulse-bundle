---
name: pulse-collaboration
description: Use when polling the local jobs/pending folder for incoming agent requests, sending a question to a teammate's Pulse instance, or ingesting a teammate's reply.
---

# Pulse Collaboration

Teammates running Pulse share OneDrive folders with each other. The folder name IS the handle. No config required — discovery is filesystem-based.

## Folder convention

```
<OneDriveRoot>\Documents\
├── Pulse\                              ← YOUR data
│   ├── projects\
│   └── jobs\
│       ├── pending\                    ← your inbox
│       └── completed\
└── Pulse-Team\                         ← teammates' shared folders
    ├── jane\jobs\pending\              ← Jane's inbox (write here to ask Jane)
    └── bob\jobs\pending\               ← Bob's inbox
```

Each subfolder under `Pulse-Team\` is a teammate. **The folder name is their handle.** Folders appear/disappear as teammates share/unshare.

To make yourself reachable: share your `Pulse\jobs\pending\` folder with teammates as `Pulse-Team\<your-handle>\jobs\pending\` (one-time setup in OneDrive web). Pick any handle — convention is lowercase first name from your work email.

## Your own handle

Resolve via Clawpilot memory: call `m_recall` for `pulse_handle`. If absent, derive from `m_get_my_profile` (lowercase first-name from email local-part). Confirm with the user once and persist via `m_recall` set. Use this handle for all outbound `from_handle` fields.

## Inbox poll

List `<PULSE_HOME>\jobs\pending\*.yaml`. Dispatch on `type:`:
- `agent_request` → answer
- `agent_response` → ingest
- anything else → skip (local-only job types)

Move every processed file to `completed\`. Never delete.

## agent_request schema

```yaml
type: agent_request
request_id: "<uuid>"
project_id: "<slug-or-empty>"
from_handle: "jane"
task: "<free-form question>"
created_at: "<ISO>"
```

No `reply_to` field — sender knows where to reply (your `Pulse-Team\<from_handle>\jobs\pending\`).

## Answering

1. **Find context.** If `project_id` is set, load `<PULSE_HOME>\projects\<project_id>.yaml`. Otherwise fuzzy-match `task` against project slugs.
2. **Pick a status:**
   - `answered` — direct evidence supports a specific answer
   - `no_context` — no relevant data; do NOT fabricate
   - `declined` — out of scope or marked private (rare)
3. **Compose the response:**

```yaml
type: agent_response
request_id: "<same uuid>"
project_id: "<same project_id>"
from_handle: "<your handle>"
status: answered
result: |
  3–8 lines, specific. Names, dates, status.
sources: ["projects/<slug>.yaml", "transcripts/<file>.md"]
created_at: "<ISO>"
```

4. **Write to** `<OneDriveRoot>\Documents\Pulse-Team\<from_handle>\jobs\pending\YYYY-MM-DD-response-<your-handle>-<first8charsOfRequestId>.yaml`. UTF-8, block-style YAML.
5. **If that folder doesn't exist**, the teammate hasn't shared back yet. Surface to user: `"Cannot reply to <from_handle> — accept their OneDrive share first."` Do NOT mkdir.
6. Move the original request to `<PULSE_HOME>\jobs\completed\`.

## Outbound — sending a question

User: "Ask Jane about the X engagement."

1. **Discover Jane:** check `<OneDriveRoot>\Documents\Pulse-Team\jane\jobs\pending\` exists. If missing, surface: `"Jane hasn't shared a Pulse folder yet."`
2. **Compose** an `agent_request` with your handle, fresh UUID, the user's question.
3. **Write to** `<OneDriveRoot>\Documents\Pulse-Team\jane\jobs\pending\YYYY-MM-DD-request-<your-handle>-<first8charsOfUUID>.yaml`.

## Ingesting agent_response

For each `agent_response` in your inbox:
1. Load `<PULSE_HOME>\projects\<project_id>.yaml` if set.
2. Append timeline entry: `[TEAM] <from_handle> answered: <first line of result>` with `source` pointing to the response file.
3. If the project YAML is missing, preserve the response in `<PULSE_HOME>\.broadcast-orphans\<request_id>.yaml`. Don't drop the teammate's work.
4. Move the response to `completed\`.

## Never

- Fabricate answers. `no_context` is legitimate.
- Delete jobs. Always move to `completed/`.
- mkdir into a teammate's folder that doesn't exist — that's a sync problem, not a routing problem.
- Drop a reply when the project YAML is missing — preserve in `.broadcast-orphans/`.
