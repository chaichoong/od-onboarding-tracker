# Operations Director · Client Onboarding

Internal client onboarding tracker for the Operations Director team.

Single-page vanilla JavaScript app hosted on GitHub Pages. Airtable is the database. Each team member uses their own Airtable Personal Access Token, stored in their browser's `localStorage`. No secrets in the repo.

This pattern matches the Content Machine app (`github.com/chaichoong/Runpreneur`).

---

## Architecture

```
Browser  ──(Bearer PAT)──►  Airtable REST API  ◄──(automations)──►  Slack / Postmark
   │
   └── localStorage: od_at_key, od_at_base, od_current_page, od_current_customer
```

- **Frontend:** one `index.html`, no build step, no framework.
- **Hosting:** GitHub Pages.
- **Auth:** each team member pastes their own Airtable PAT into the connect screen on first load. Token sits only in their browser.
- **Database:** Airtable base `Operations Director Client Onboarding` (schema in `05_AIRTABLE_SCHEMA.md`).
- **Cron / side effects:** Airtable Automations handle scheduled MAR sweeps, Slack alerts, and Postmark sends. The browser app reads and writes records; it does not orchestrate.

---

## What it does

- **Dashboard:** cohort table of active onboarding journeys with stage, days in stage, days since signup, health score, open MAR count, target First Win.
- **Customer detail:** timeline of stage transitions, open and resolved MAR flags, manual stage override (audited), Resolve flag actions.
- **MAR Triage:** open flags sorted by severity then age. Resolve in one click. SLA breach highlighted.
- **Settings:** reconnect Airtable, force-reload from API.

What it deliberately does **not** do in v1:
- Send emails directly. Airtable Automation sends emails on record changes.
- Post to Slack directly. Airtable Automation does.
- Compute health scores or evaluate MAR rules in the browser. Airtable formulas and Automations do.

---

## Set up

### 1. Build the Airtable base
Follow `05_AIRTABLE_SCHEMA.md`. Get a Pro plan for Personal Access Tokens with base-scoped permissions.

Required tables (names must match exactly):
- `Customers`
- `Journeys`
- `Stage Transitions`
- `Telemetry Events`
- `Touchpoint Deliveries`
- `MAR Rules`
- `MAR Flags`
- `Hero Workflows`
- `Activation Event Definitions`
- `Tutorial Progress`
- `Team Members`

### 2. Create the GitHub repo
1. Create a new private repo: `od-client-onboarding` (or any name).
2. Drop `index.html` and `README.md` into the repo root.
3. Commit.

### 3. Enable GitHub Pages
1. Repo → Settings → Pages.
2. Source: Deploy from branch.
3. Branch: `main` / `(root)`.
4. Save.
5. Wait ~30 seconds for first deploy. URL will be `https://<your-user>.github.io/od-client-onboarding/`.

Custom domain optional. Add a `CNAME` file with `onboarding.opsdirector.co.uk` (or chosen subdomain) and point DNS at GitHub.

### 4. Each team member connects on first load
1. Each user creates their own Airtable PAT at <https://airtable.com/create/tokens>.
   - Scopes: `data.records:read`, `data.records:write`.
   - Access: only the `Operations Director Client Onboarding` base.
2. Open the GitHub Pages URL.
3. Paste the PAT and the Base ID (starts with `app...`, visible in any Airtable URL for the base).
4. Click Connect.

The token is verified against the `Customers` table and saved to `localStorage`. Reconnect any time via Settings.

---

## Updating the app

Same pattern as Content Machine:

1. Edit `index.html` directly in GitHub (pencil icon → Edit).
2. Commit.
3. GitHub Pages rebuilds in roughly 30 seconds.
4. Team hard-refreshes their browser (Cmd+Shift+R or Ctrl+Shift+R).

For larger changes, clone the repo locally and push as normal.

---

## Airtable Automations to add (post-deploy)

Build these inside the Airtable base. The browser app does not need to know about them.

1. **MAR sweep (every 4 hours):** scheduled trigger → script step that evaluates time-based MAR rules and creates `MAR Flags` rows.
2. **Slack alert on new MAR Flag:** trigger when `MAR Flags` record created → conditional by `Severity` → send to one of three Slack incoming webhooks.
3. **Email on stage transition:** trigger when `Stage Transitions` record created where `Trigger Kind = system` → send Postmark email from template.
4. **Failure protocol (daily at 09:00 UK):** scheduled trigger → script step that flags Day 21, 28, 35 journeys per `01_DESIGN_DOC.md` Section 10.
5. **Health Score recalc (hourly):** scheduled trigger → script step that recomputes the composite score per `01_DESIGN_DOC.md` Section 8 and writes to `Journeys.Health Score`.

Each script is plain JavaScript inside an Airtable Automation. The `01_DESIGN_DOC.md` spec is the reference for the logic.

---

## Security notes

- The Airtable PAT lives only in the team member's browser (`localStorage`).
- Make the GitHub Pages repo **private** (GitHub Pro or Enterprise) so only authenticated team members can load the page.
- Revoke a token at <https://airtable.com/create/tokens> any time if a device is lost.
- Do not put any token in the repo. The app prompts for it on first load.
- Optional hardening: add a Cloudflare Worker proxy if you later need any operation that cannot be done from the browser (e.g. a vendor API with no CORS).

---

## File map

```
od-client-onboarding/
├── index.html        ← the app
├── README.md         ← this file
└── (optional later)
    ├── CNAME         ← custom domain
    └── automations/  ← Airtable Automation scripts as JS files for version control
```

## Related docs (in the same Drive/repo)

- `01_DESIGN_DOC.md` — full target architecture (used as reference for Airtable Automation logic).
- `05_AIRTABLE_SCHEMA.md` — exact Airtable schema to build first.
- `Customer_Onboarding_Experience_Process_v1.1.docx` — philosophy and journey map.

---

## Roadmap (after v1 ships)

- Manual "send template email" action in the customer detail page (writes to a `Touchpoint Queue` table; Airtable Automation sends and updates).
- "Escalate to Kevin" wired to a Slack DM via an Airtable Automation.
- Telemetry events display on customer detail timeline.
- Health Score component breakdown tooltip.
- Saved filter sets per user.
