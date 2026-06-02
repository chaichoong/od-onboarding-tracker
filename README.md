# Operations Director · Client Onboarding

**Version:** v2.0
**Audience:** Operations Director internal team
**Purpose:** track every Operations OS client through the 90-day implementation, from payment to all-10-modules-live, in one dashboard.

Single-page vanilla JavaScript app hosted on GitHub Pages. Airtable is the database. Each team member uses their own Airtable Personal Access Token, stored only in their browser. No secrets in the repo.

Same pattern as your Content Machine app (`github.com/chaichoong/Runpreneur`).

---

## What it tracks

Every client goes through 7 stages during the 90-day Operations OS build:

```
Day 0          Day 0-2          Day 2-14            Day 14            Day 14-90       Day 90
PAYMENT  →     WELCOME      →   BUILD            →  FIRST WIN     →   EXPANSION   →   ACTIVATED
               + Onboarding     Priority module     Priority module   Other 9         All 10
               form             being built         live + working    modules         modules
                                + 9 in parallel     for the client    online weekly   live
```

The dashboard surfaces:

- **Cohort view:** every active client, their current stage, days in stage, days since signup, health score, open MAR (Member-at-Risk) flag count, target First Win date, Priority Module.
- **Client detail:** stage transition timeline, open and resolved MAR flags, Priority Module + pain statement, manual stage override (audited), Resolve flag actions.
- **MAR triage:** open risk flags sorted by severity then age, SLA breach highlighted, resolve in one click.
- **Settings:** reconnect Airtable, force-reload data from the API.

What it deliberately does **not** do in the browser:

- Send emails. Postmark handles this from Airtable Automations.
- Post to Slack. Airtable Automations do.
- Compute health scores or evaluate MAR rules. Airtable formulas and Automations do.

---

## Architecture

```
Browser  ──(Bearer PAT)──►  Airtable REST API  ◄──(Automations)──►  Postmark / Slack / Make
   │                              ▲
   │                              │
   │                          GHL Workflow  ◄── customer pays £1,997
   │
   └── localStorage: od_at_key, od_at_base, od_current_page, od_current_client
```

- **Frontend:** one `index.html`, no build step, no framework.
- **Hosting:** GitHub Pages (private repo if you're on Pro plan).
- **Auth:** each team member pastes their own Airtable PAT into the connect screen on first load. Token sits only in their browser.
- **Database:** Airtable base `⚙️ Operations Director` (Base ID: `appnqjDpqDniH3IRl`).
- **Customer service:** AI chatbot (Claude via Cloudflare Worker) + email to `info@operationsdirector.co.uk`. **No live calls.**

---

## The data model (7 tables in the Operations Director base)

All onboarding tables are prefixed `🎯 Onboarding` so they group together in your existing base.

| Table | Purpose |
|---|---|
| `🎯 Onboarding Clients` | One row per paying client. Identity, journey state, Priority Module link, access permissions. |
| `🎯 Onboarding Activity Log` | One row per event. Polymorphic — stage transitions, telemetry, touchpoints, tutorials, notes. |
| `🎯 Onboarding MAR Flags` | Member-at-Risk flags raised by the Automations. Each flag links to a Client and a Rule. |
| `🎯 Onboarding MAR Rules` | 13 editable rule definitions (no_login_24h, day21_no_activation_event, etc.). |
| `🎯 Onboarding Modules` | The 10 Operations OS modules. Source of truth: operationsdirector.co.uk. |
| `🎯 Onboarding Module Data` | Just-in-time mini-form responses, one row per (Client, Module). 10 form views, one per module. |
| `🎯 Onboarding Form Responses` | Main onboarding form submissions (post-payment). One row per client. |

Plus the existing `Team Members` table (reused, not duplicated).

---

## The 10 Operations OS modules

All 10 are built during every implementation. The client picks one on the onboarding form as their **Priority Module** — the one built first by Day 14. The other 9 come online weekly over Days 14-90.

| Module | Category |
|---|---|
| Leadership Dashboard OS | Leadership |
| Objective & Strategy OS | Leadership |
| Profit & Loss OS | Finance |
| Cash Flow Voids OS | Finance |
| Invoices OS | Finance |
| Task & Project Management OS | Operations |
| Contractor Job List OS | Operations |
| Inbound Comms OS | Communications |
| Property Compliance OS | Compliance |
| OD Launch Plan OS | Launch |

---

## Set up

### 1. Deploy this app to GitHub Pages

1. Create a private repo: `od-client-onboarding`.
2. Drop `index.html` and this `README.md` into the repo root.
3. Repo → Settings → Pages → Source: deploy from branch → `main` / `(root)` → Save.
4. Wait ~30 seconds. URL: `https://<your-user>.github.io/od-client-onboarding/`.
5. Optional: add a `CNAME` file with `onboarding.opsdirector.co.uk` and a DNS CNAME to GitHub for a custom domain.

### 2. Build the Airtable schema

All 7 tables are already in the `⚙️ Operations Director` base. If starting from scratch, follow `05_AIRTABLE_SCHEMA.md` for the universal table structure and `17_MODULE_MINI_FORMS.md` for the just-in-time module data table.

There are a few manual fields the API cannot create (lookups, formulas, counts, rollups). See `10_AIRTABLE_MANUAL_FIELDS.md`.

### 3. Wire up payment → form → tracker

Three external pieces, all documented:

- **GHL:** create the `Operations OS Client` tag, apply it on checkout, fire a webhook on tag added. See `13_PAYMENT_FORM_INTEGRATION.md` Part 1.
- **Make.com:** receive the GHL webhook, create the row in `🎯 Onboarding Clients`. See `13_PAYMENT_FORM_INTEGRATION.md` Part 1.3.
- **Airtable form views:** main onboarding form (`14_AIRTABLE_FORM_SETUP.md`) + 10 mini-forms one per module (`17_MODULE_MINI_FORMS.md`).

### 4. Set up the 10 Airtable Automations

Pre-written Airtable Script actions, one per file in `automations/`. Setup steps per Automation in `automations/README.md`.

| Automation | Trigger |
|---|---|
| 01_on_signup_send_welcome | Client created with Stage = `welcome` |
| 02_on_brief_submitted | (Legacy/fallback) Pain Statement set on Client |
| 03_on_config_complete | Configuration Complete checkbox ticked |
| 04_mar_sweep | Scheduled every 4 hours |
| 05_failure_protocol | Scheduled daily 09:00 UK |
| 06_health_score | Scheduled hourly |
| 07_slack_on_mar_flag | MAR Flag record created |
| 08_on_activation_event | Activation Event Fired At populated on Client |
| 09_on_form_submitted | Main onboarding form submitted |
| 10_on_module_data_submitted | Module Data mini-form submitted |

### 5. Postmark sender

Verify `info@operationsdirector.co.uk` as a Postmark sender. DKIM + SPF on operationsdirector.co.uk DNS. Note the server token.

### 6. Slack incoming webhooks

Create 4 channels and 4 webhooks: `#cs-mar-critical`, `#cs-mar-high`, `#cs-mar-digest`, `#cs-onboarding`.

### 7. AI chatbot (Claude)

Customer service tier 1 is a Claude-powered chatbot embedded on operationsdirector.co.uk and linked from every email. Architecture and build plan in `16_NEXT_STEPS_v2.md` Section 4. Roughly 1-2 engineering days.

### 8. Each team member connects on first load

1. Each user creates their own Airtable PAT at <https://airtable.com/create/tokens>:
   - Scopes: `data.records:read`, `data.records:write`.
   - Access: only the `⚙️ Operations Director` base.
2. Open the GitHub Pages URL.
3. Paste the PAT and the Base ID `appnqjDpqDniH3IRl`.
4. Click Connect.

The token is verified against the Clients table and saved to `localStorage`. Reconnect any time via Settings.

---

## Updating the app

Same flow as Content Machine:

1. Edit `index.html` directly in GitHub (pencil icon → Edit).
2. Commit to `main`.
3. GitHub Pages rebuilds in ~30 seconds.
4. Team hard-refreshes their browser (Cmd+Shift+R / Ctrl+Shift+R).

For larger changes, clone the repo locally and push as normal.

---

## Customer service model

No live calls during onboarding. Ever. The brand promise on operationsdirector.co.uk is "the business runs without you watching it" — we model that internally too.

**Tier 1: Claude AI chatbot.** Hosted on the website. Powered by Claude via a Cloudflare Worker proxy. System prompt contains the full playbook + module SOPs.

**Tier 2: email to info@operationsdirector.co.uk.** Shared inbox. Team triages. SLA: 1 working day non-urgent, 4 hours urgent.

No phone numbers, no Calendly links, no Zoom rooms anywhere in the onboarding stack.

---

## Security notes

- The Airtable PAT lives only in the team member's browser (`localStorage`).
- Make the GitHub Pages repo **private** (GitHub Pro or Enterprise) so only authenticated team members can load the page.
- Revoke a token at <https://airtable.com/create/tokens> any time if a device is lost.
- Do not put any token in the repo. The app prompts for it on first load.
- Optional: a Cloudflare Worker proxy for the AI chatbot endpoint (same pattern as Content Machine's `content-machine-proxy`).

---

## File map (in repo)

```
od-client-onboarding/
├── index.html                              ← the dashboard
├── README.md                               ← this file
├── form_cover_1800.jpg                     ← Airtable form cover image
└── automations/                            ← Airtable Automation scripts
    ├── README.md
    ├── 01_on_signup_send_welcome.js
    ├── 02_on_brief_submitted.js
    ├── 03_on_config_complete.js
    ├── 04_mar_sweep.js
    ├── 05_failure_protocol.js
    ├── 06_health_score.js
    ├── 07_slack_on_mar_flag.js
    ├── 08_on_activation_event.js
    ├── 09_on_form_submitted.js
    └── 10_on_module_data_submitted.js      ← coming next session
```

---

## Related docs

The canonical references, in the order you'll typically need them:

| Doc | What's in it |
|---|---|
| `11_STAGE_PLAYBOOK.md` | The operating manual. Every stage. Every email. Every automation. **Read this first.** |
| `15_REVISION_v2.md` | The shift from v1.1 (3 hero workflows) to v2.0 (10 modules). |
| `16_NEXT_STEPS_v2.md` | Answers to 5 design decisions. Includes Claude chatbot setup guide. |
| `13_PAYMENT_FORM_INTEGRATION.md` | GHL → Make → Airtable pipeline + onboarding form setup. |
| `14_AIRTABLE_FORM_SETUP.md` | Click-by-click for the main onboarding form view. |
| `17_MODULE_MINI_FORMS.md` | Setup for the 10 per-module mini-forms. |
| `10_AIRTABLE_MANUAL_FIELDS.md` | The lookup/formula/count fields the API can't create. |
| `09_GITHUB_DEPLOY_STEPS.md` | Click-by-click deploy of this app to GitHub Pages. |
| `automations/README.md` | Per-Automation setup steps inside Airtable. |
| `Customer_Onboarding_Experience_Process_v1.1.docx` | Original visual journey map. Still useful as philosophy doc. |

---

## v3 backlog (after first 10 paying clients)

- AI suggests Priority Module from the Pain Statement using a small Claude call on form submit.
- Slack alerts route to the assigned Owner directly, not just channels, for high-priority flags.
- Health Score breakdown tooltip in the client detail page.
- Weekly digest email to Kevin summarising the cohort.
- Auto-advance from First Win to Expansion after 72h if no testimonial submitted.
- Auto-detect Definition of Activated nightly and auto-advance to `activated`.
- Per-Industry Postmark template variants.
