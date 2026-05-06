---
name: pulse-signal
description: Use when the user wants to draft a field signal — a customer win, loss, escalation, competitive observation, product feedback, or reusable IP/initiative — for internal sharing, executive review, or team distribution.
---

# Pulse Signal

Draft structured field signals from customer engagements. Signals are short, factual reports that capture wins, losses, escalations, compete intel, product feedback, and reusable IP — formatted for internal review and team distribution.

## Path

Resolve `<PULSE_HOME>` per `/pulse-projects`. Save signals to `<PULSE_HOME>\pulse-signals\<YYYY-MM-DD>-<slug>.md`. Slug = lowercased-hyphenated descriptive title.

## Workflow

1. **Gather input.** What engagement, what observation, what's the headline?
2. **Enrich context.** Use M365 tools (`m365_search_emails`, `m365_list_emails`, `m365_list_chats`, `m365_get_event`, etc.) to pull recent communications. If a matching project exists at `<PULSE_HOME>\projects\<slug>.yaml`, read it for stakeholders, commitments, and timeline.
3. **Classify.** Pick one of the seven signal types below.
4. **Fill the template.** If a required field is unknown, ask the user. **Never fabricate.**
5. **Write** to `<PULSE_HOME>\pulse-signals\<YYYY-MM-DD>-<slug>.md`. One signal per file.

## Signal templates

### Customer Win
- **Customer**: name(s)
- **Deal value / impact**: $$ or strategic measure (consumption commitment, ARR, deal size)
- **Technology**: product/service used
- **Compete**: competitor name (if applicable)
- **IP used**: reusable assets leveraged
- **Use cases**: what the customer is doing
- **Industry**: vertical
- **Description**: 3–4 sentences — situation, approach, technical solution, compete details
- **Business impact**: quantify (ROI, cost reduction, UX, time-to-value)

### Customer Loss
- **Customer**: name(s)
- **Type**: Loss | Blocked | Challenged
- **Deal value / impact**: $$ lost or at risk
- **Technology**: product/service in scope
- **Use cases**: what they wanted
- **Industry**: vertical
- **Compete**: competitor name (if applicable)
- **Description**: 3–4 sentences — situation, why we lost/blocked, technical or execution gap
- **Learnings**: what peers can apply to future opportunities

### Customer Escalation
> Reserve for executive-level issues with material business impact. **Not** for UAT, tech support tickets, or routine blockers.

- **Customer**: name
- **Impact**: quantify ($$ at risk, deal value, strategic exposure)
- **Exec sponsor**: name of internal exec engaged (or "none yet")
- **Compete**: competitor (if applicable)
- **Description**: situation, context, how we got here
- **Impact if not resolved**: $$ impact, deadline
- **Ask**: ideal resolution or recommended next step

### Compete Signal
- **Competitor**: name
- **Type**: New entry | Pricing change | Feature differentiator | Strategy shift | Customer feedback
- **Description**: what was observed, why it's significant, single or multi-customer
- **Customer/segment**: names or market segment
- **Potential impact**: who should act, what's at stake
- **Recommendation**: details

### Product Signal
- **Product/service**: name
- **Type**: Feature request | Bug | Performance | Integration | Other
- **Source**: Customer feedback | Partner | Internal testing | Market research
- **Description**: context, details of the observation
- **Use cases**: where it surfaced
- **Compete comparison**: how a competitor handles it (if relevant)
- **Customer examples**: names where observed
- **Impact**: how this affects customers or your business
- **Recommendation**: what should change

### IP / Initiative / Best Practice
- **Title**: short title
- **Type**: IP | Initiative | Program | Best Practice
- **Description**: objectives, key activities, learnings, success metrics
- **Customers/segment impacted**: names or segments
- **Status**: Planning | In Progress | Completed | Paused
- **Next steps**: what's next
- **Team/region**: where this is owned
- **Recommendation**: details

### Skills / People Signal
- **Team/area**: name
- **Type**: Hiring need | Skill gap | Training requirement | Resource request
- **Description**: recurring asks, capability gaps, asset needs
- **Impact**: on delivery, customer satisfaction, time-to-market
- **Proposed solution**: recommended approach
- **Timeline**: Immediate | This quarter | Next quarter | Long-term

## Cross-reference projects

If a matching project YAML exists at `<PULSE_HOME>\projects\<slug>.yaml`:
- Add a `Source: projects/<slug>.yaml` line at the bottom of the signal file.
- Ensure facts (commitments, stakeholders, dates) are consistent with the project YAML — flag any contradiction to the user before writing.

## Rules

- **Specifics only.** Customer names, deal sizes, product names, dates. Vague signals waste reader time and dilute trust.
- **Never fabricate.** If a required field is unknown, ask the user; if they don't know, omit that field or skip the signal entirely.
- **One file per signal.** Don't combine multiple wins or losses.
- **Filename slug must be informative** — `2026-05-05-acme-foundry-displacement-win.md`, not `2026-05-05-win.md`.

## Never

- Force a signal when the input is vague — say "not enough specifics; need X, Y, Z" and stop.
- Stack multiple unrelated observations in one file.
- Reference projects/customers by abbreviation only — always include the full name once on first mention.
- Editorialize — report what happened and what's recommended, not what the reader should think.
