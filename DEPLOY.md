# CEO Visibility Reset — Setup & Deploy Guide

Follow these sections in order. Sections A–C get the n8n backend live; D–F get the front end onto GitHub Pages and lock down CORS.

---

## A. Prepare the Google Sheet (2 minutes)

1. Open the log sheet: https://docs.google.com/spreadsheets/d/1j1ZgyGwu-iMIMa5TIfRKQGsCZsmDkNC2UonXBiTN4r8/edit
2. Rename the first tab (bottom-left) to exactly: `log` (lowercase).
3. In row 1, add these 7 headers, one per column, A through G:

   | A | B | C | D | E | F | G |
   |---|---|---|---|---|---|---|
   | session_id | guest_name | phase | attempt_number | input | output | timestamp |

That's it. Every call to the webhook appends one new row — nothing ever overwrites.

---

## B. Import and configure the n8n workflow

1. Open your workflow at https://n8n.seichodigital.com/workflow/rO8eAu106JzSlpN9
2. Open the file `n8n/kocrea-ceo-visibility-workflow.json` from this repo in a text editor, select all, copy.
3. Click on an empty spot of the n8n canvas and paste (Ctrl+V). All 29 nodes and their connections appear.
   - Alternative: n8n menu (⋯ top-right) → **Import from File** → select the JSON.
4. **Set the shared secret:** open the **Validate Request** node. At the top of its code:
   ```js
   const SHARED_SECRET = 'REPLACE_WITH_YOUR_SHARED_SECRET';
   ```
   Replace with a long random string (e.g. run `openssl rand -hex 24`, or just mash 40+ random characters). Keep it — the front end needs the same value.
   The AI model is set on the next line (`claude-sonnet-4.6`) — change it there if you ever want a different TokenMix model.
5. **TokenMix credential** — the 4 `Call TokenMix` nodes are native **OpenAI** nodes (TokenMix is OpenAI-compatible). If you already have a TokenMix credential of type *OpenAI* on this instance, skip to step 6. Otherwise create it once:
   - n8n → Credentials → Add credential → **OpenAI**
   - Name it `TokenMix`
   - **API Key:** your TokenMix key (`tm-…`)
   - **Base URL:** `https://api.tokenmix.ai/v1`
   - Save. (A red "could not connect" warning on save is usually just the credential test hitting an OpenAI-only endpoint — if the model call works in step C, ignore it.)
6. **Attach the credential to the 4 OpenAI nodes:** open each of `Call TokenMix P1`…`P4` and select the TokenMix credential. The *Model* field is set by expression (`{{ $json.ai_model }}`, fed from the Validate Request config) — don't replace it with a dropdown pick.
7. **Attach your Google credential to the 7 Sheets nodes:** open each of `Log P1`…`P4` and `Read Log P2`…`P4`, select your existing Google Sheets credential.
8. **Save** the workflow, then toggle it **Active** (top-right).

Your production webhook URL is now:
```
https://n8n.seichodigital.com/webhook/ceo-visibility
```
(While testing with "Execute workflow" in the canvas, use `webhook-test/ceo-visibility` instead — it only accepts one call per click.)

---

## C. Test the backend (before touching the front end)

Run these from any terminal. Replace `YOUR_SECRET` with the value from step B4.

**1. Secret check (expect HTTP 401):**
```bash
curl -i -X POST https://n8n.seichodigital.com/webhook/ceo-visibility \
  -H "Content-Type: application/json" -H "X-Shared-Secret: wrong" \
  -d '{"phase":1,"inputs":{}}'
```

**2. Missing session on phase 2 (expect HTTP 400, code NO_SESSION):**
```bash
curl -i -X POST https://n8n.seichodigital.com/webhook/ceo-visibility \
  -H "Content-Type: application/json" -H "X-Shared-Secret: YOUR_SECRET" \
  -d '{"phase":2,"inputs":{}}'
```

**3. Real Phase 1 run (expect JSON with a fresh session_id + 5 scored dimensions):**
```bash
curl -s -X POST https://n8n.seichodigital.com/webhook/ceo-visibility \
  -H "Content-Type: application/json" -H "X-Shared-Secret: YOUR_SECRET" \
  -d '{"phase":1,"guest_name":"Test HR advisor","inputs":{
    "visibility_priority":"I do not have time for marketing, client work always comes first",
    "communication_habits":"I post on LinkedIn when I have time, maybe once a month",
    "target_clarity":"HR directors mostly, but honestly anyone who pays",
    "revenue_connection":"Referrals and word of mouth bring most of my business",
    "authority_signals":"I do not know what I do better than my competitors"}}'
```
Copy the `session_id` from the response, then chain phases 2 → 3 → 4 passing the previous outputs in `inputs` (the front end does this automatically — this is just for sanity).

**4. Check the sheet:** each call above (including the failed ones' successful siblings) should have appended a row with input/output JSON and a timestamp. Attempt numbers should increment if you repeat the same phase for the same session.

---

## D. Configure and deploy the front end (GitHub Pages)

The repo already exists: https://github.com/maxclaeys/kocrea_ceo_visibility (public — required for free GitHub Pages; note the shared secret ends up readable in the page source either way, so treat it as soft protection against drive-by calls, not as a real credential. Rotate it whenever you like: change it in one n8n node + one line of index.html.)

1. Edit `index.html` — near the bottom, in the CONFIG block:
   ```js
   const WEBHOOK_URL   = 'https://n8n.seichodigital.com/webhook/ceo-visibility'; // already correct
   const SHARED_SECRET = 'REPLACE_WITH_YOUR_SHARED_SECRET';                      // paste your secret
   ```
2. Commit and push (see F for the exact commands).
3. On GitHub: repo → **Settings → Pages**
   - *Source:* **Deploy from a branch**
   - *Branch:* `main`, folder `/ (root)` → **Save**
   - `index.html` lives at the repo root (not `/docs`) because this repo contains nothing but this tool — no reason to add a folder level.
4. Wait ~1 minute, refresh the Pages settings page. The live URL will be:
   ```
   https://maxclaeys.github.io/kocrea_ceo_visibility/
   ```
5. Open it, run a full Phase 1→4 session, then test "Redo this phase" on Phase 2 and confirm Phases 3–4 rebuild from the edited output.

---

## E. Lock down CORS (after first successful deploy)

The imported workflow ships with CORS open (`*`) so you can test from anywhere. Once the page is live:

1. Open the **Webhook** node in n8n.
2. Under *Options → Allowed Origins (CORS)*, change `*` to:
   ```
   https://maxclaeys.github.io
   ```
   (Origin = scheme + domain only, no path.)
3. Save and re-activate. The browser preflight and the POST responses are both handled by this one setting — don't add a duplicate `Access-Control-Allow-Origin` header on the Respond nodes, doubling the header breaks CORS.

---

## F. Updating the deployed page (minimum reliable git workflow)

From the repo folder:

```bash
git add index.html          # or: git add -A   for everything
git commit -m "Update front end"
git push
```

GitHub Pages redeploys automatically ~30–60s after each push to `main`. Hard-refresh the live page (Ctrl+Shift+R) to bypass cache.

---

## Response envelope reference (what the front end expects)

```jsonc
// success
{ "session_id": "…uuid…", "phase": 2, "attempt_number": 1, "error": false, "output": { /* AI JSON */ } }
// AI soft-failure (HTTP 200)
{ "session_id": "…", "phase": 2, "attempt_number": 1, "error": true, "message": "Diagnosis incomplete, please retry" }
// missing session (HTTP 400)
{ "error": true, "code": "NO_SESSION", "message": "Missing session_id — start from Phase 1" }
// bad secret (HTTP 401)
{ "error": true, "code": "BAD_SECRET", "message": "Unauthorized" }
```

## Session rules (as built)

- Only Phase 1 mints a `session_id` (fresh UUID every time, even on retry — a Phase 1 retry is a new session).
- Phases 2–4 reject with 400/NO_SESSION if no `session_id` is sent.
- Every call appends a new row; `attempt_number` = count of prior rows for that session+phase, +1.
- "Final" state is derived (highest phase, latest timestamp) — there is no stored flag.
