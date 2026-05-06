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
   Both skills + all four automations install in one click.
3. New chat → `/pulse-projects` and describe a customer engagement. The skill creates the YAML in your OneDrive.

That's the entire setup. No config files. No onboarding walkthrough.

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
<OneDriveRoot>\Documents\
├── Pulse\                            ← YOUR projects, jobs, etc.
└── Pulse-Team\
    ├── <your-handle>\jobs\pending\   ← what teammates write to ask YOU
    ├── jane\jobs\pending\            ← what you write to ask Jane
    └── bob\jobs\pending\             ← what you write to ask Bob
```

**Setup (one-time, in OneDrive web):** share your `Pulse\jobs\pending\` folder with teammates as `Pulse-Team\<your-handle>\jobs\pending\`. Accept their shares. The folder name IS the handle.

`/pulse-collaboration` does the rest: poll your inbox, route requests/responses by folder name, archive processed jobs. No team list to maintain — folders appear/disappear as people share/unshare.

Enable the **Team collaboration poll** automation only after you've shared your folder with at least one teammate. Until then it'll just no-op every 5 minutes (silent).

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
