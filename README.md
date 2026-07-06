# The CEO Visibility Reset — Live Diagnosis Tool

Internal tool for the Kocrea podcast: runs a solo founder through a 4-phase AI visibility diagnosis, live on screen.

- **`index.html`** — the entire front end (static, no build step, GitHub Pages). Fill in `SHARED_SECRET` in the CONFIG block before deploying.
- **`n8n/kocrea-ceo-visibility-workflow.json`** — the complete n8n workflow (webhook → secret check → phase router → 4 TokenMix prompt branches → Google Sheets audit log). Paste into the n8n canvas to import.
- **`DEPLOY.md`** — step-by-step setup: sheet prep, n8n import, credentials, curl tests, GitHub Pages deploy, CORS hardening.

Phases: 1 Visibility Friction → 2 Positioning Gap → 3 Content Architecture (3 pillars) → 4 Weekly Reflex (A.A.A. formula).
