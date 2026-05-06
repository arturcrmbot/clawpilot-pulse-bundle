# Pulse for Clawpilot

Persistent project memory + lightweight inter-agent collaboration over OneDrive + structured field-signal drafting. Three skills, four automations, zero config.

The bundle leans on Clawpilot's built-in M365 tools, Playwright MCP, filesystem MCP, Horizon, and automation scheduler. It adds three things Clawpilot doesn't ship: a curated project-memory schema, a folder-convention protocol for teammate-to-teammate questions, and a signal-drafter for capturing field intel in standard templates.

## What's in here

### Skills

| Skill | Triggers when |
|---|---|
| **`/pulse-projects`** | User mentions a customer, deal, stakeholder, commitment, or named engagement |
| **`/pulse-collaboration`** | Polling the local job inbox, sending a teammate's agent a question, or ingesting a reply |
| **`/pulse-signal`** | User wants to draft a field signal — customer win, loss, escalation, compete intel, product feedback, or reusable IP |

### Automations (start disabled — enable each in the Automations panel)

| Name | Schedule | What it does |
|---|---|---|
| Morning digest | Daily 07:00 | Overdue commitments + today's meetings + Horizon + updated projects + candidate new projects + team inbox count |
| Workday triage | Every 30 min, weekdays, anchor 09:00 | Classify unread mail/chats, draft replies (never auto-sends), update commitments |
| Daily intel brief | Daily 09:00 | RSS digest filtered for relevance — edit the prompt to point at your own feeds |
| Team collaboration poll | Every 5 min | Process incoming `agent_request`/`agent_response` YAMLs in `jobs/pending`. Silent when empty. |

All four start `enabled: false`. Enable each in Clawpilot's Automations panel after smoke-testing the corresponding flow once. The intel automation expects you to **edit the prompt** to swap in your own feed URLs and competitors — defaults are AI/LLM industry feeds.

## Install

1. Install Clawpilot. Sign in. Confirm M365 status in Settings.
2. **Automations panel → Import → Import from GitHub**. Paste:
   ```
   https://github.com/arturcrmbot/clawpilot-pulse-bundle
   ```
   All three skills + four automations install in one click.

Then run the setup walkthrough below.

## Setup walkthrough

A few one-time steps. Most are clicks; Clawpilot handles the rest.

### 1. Settings → Permissions → enable

Open Clawpilot's **Settings → Permissions** and enable everything you're comfortable with. Saves a lot of click-through prompts later when skills call tools.

### 2. Set up your Pulse folders (one chat command)

In a fresh chat, say:

> *"Set up my Pulse folders, including team collaboration."*

Clawpilot creates everything you need under your OneDrive: project memory, jobs inbox, digest/intel/signal archives. It saves your handle to its memory so it doesn't have to ask again. It opens File Explorer at the folder you'll need to share next.

### 3. Share your `jobs/pending/` folder with teammates

For each teammate you want to collaborate with:

1. Right-click the folder Clawpilot opened (or navigate to it in OneDrive web).
2. Share → grant **Can edit** access.
3. Send.

**Important:** share only the `pending/` folder, **never its parent `jobs/`**. Sharing the parent exposes your `completed/` archive — every past inter-agent message you've sent or received.

### 4. Accept teammates' shares — and click "Add shortcut to my files"

When a teammate shares their pending folder back:

1. Open the share notification (or go to OneDrive web → Shared with me).
2. Click **Add shortcut to my files**.

**This is the critical step.** Without "Add shortcut", the share is read-only-cloud and your replies can't route. If you only "open" the share, your Pulse will never find their inbox.

### 5. Wait ~5 minutes for OneDrive to sync

OneDrive needs a few minutes to surface the new shared folders to your local view. After that:

- The **Team collaboration poll** automation (every 5 min) starts auto-processing inbound requests
- To send a question, just chat: *"Ask Jane about the Acme engagement."*
- Replies arrive in your inbox over the next minutes/hours; the poll picks them up and updates project memory

Day-to-day use after this is just chat. No config files, no path-typing, no folder admin.

## Where data lives

`<PULSE_HOME>` resolves at runtime via PowerShell:

```powershell
if ($env:OneDriveCommercial) { "$env:OneDriveCommercial\Documents\Pulse" }
elseif ($env:OneDrive)       { "$env:OneDrive\Documents\Pulse" }
else                         { "$env:USERPROFILE\Documents\Pulse" }
```

Microsoft tenants resolve to work OneDrive automatically. Personal accounts get personal OneDrive. Non-OneDrive falls back to user profile. Project YAMLs live at `<PULSE_HOME>\projects\<slug>.yaml`.

The first `/pulse-projects` write creates `<PULSE_HOME>\projects\` if missing. Other folders (`digests\`, `intel\`, `jobs\pending\`) are created by the corresponding automations on first run.

## Inter-agent collaboration

Convention-only — no config files.

```
<OneDriveRoot>\Documents\Pulse\
├── projects\                              ← YOUR project YAMLs
└── jobs\
    ├── pending\                           ← YOUR INBOX — teammates write here
    └── completed\                         ← YOUR ARCHIVE — never share this
```

After teammates accept your share and you accept theirs, your local OneDrive surfaces their inboxes:
```
<OneDriveRoot>\
├── <Their Display Name>'s files - pending\   ← write here to message them
└── ...
```

### Setup — share only `pending/` (one-time, in OneDrive web)

1. Navigate to `<PULSE_HOME>\jobs\pending\` in OneDrive web.
2. Right-click → Share → grant **Can edit** to each teammate by name.
3. **Share only the `pending/` folder, not its parent.** Sharing `jobs/` exposes your `completed/` archive (historical messages, PII).
4. Accept teammates' shares the same way. If you see a `completed/` folder inside their share, tell them — they should re-share only `pending/`.

`/pulse-collaboration` does the rest: poll your inbox, route requests/responses by folder name, archive processed jobs. No team list to maintain — folders appear/disappear as people share/unshare.

Enable the **Team collaboration poll** automation only after you've accepted at least one teammate's share. Until then it'll just no-op every 5 minutes (silent).

## MSX MCP (optional, Microsoft-internal)

If you have access to the internal `@mcaps-microsoft/msx-mcp` plugin and want CRM context in projects:

1. Install plugin to `~/.copilot/installed-plugins/_direct/MSX-MCP-main/`.
2. `az login` once.
3. Clawpilot → Extensions → MCP Servers → **Add** → Command mode → `node C:\Users\<your-username>\.copilot\installed-plugins\_direct\MSX-MCP-main\scripts\bootstrap.mjs`.
4. New chat → ask "list 5 MSX opportunities I own". First call takes ~2 min (token acquisition).

If you don't have access, the bundle still works — you just lose CRM cross-reference.

## Pulse Agent compatibility

If you previously ran the Python [Pulse Agent](https://github.com/gbb-pulse) daemon, this bundle reads/writes the **same** OneDrive folders. No migration. The schema is shared. You can run both in parallel; just disable Pulse Agent's daemon-side digest/triage/intel scheduling so they don't duplicate work.

The one capability not yet ported: Teams meeting transcript collection. If your tenant isn't enrolled in Horizon's Loki backend, keep Pulse Agent's transcript collector running — it's a separate process from the rest.

## Editing

Skills install to `~/.copilot/skills/pulse-*/SKILL.md`. Automations install to `~/.copilot/m-automations/automations.json`. Edit the files directly with any editor; restart Clawpilot to pick up changes. Or fork into local skills via Extensions panel → Skills tab → Add.

The intel automation's prompt is the obvious thing to edit — swap in your own feeds and competitors. Each automation's prompt is the workflow contract; tune it to taste.

## License

MIT — see [LICENSE](LICENSE).
