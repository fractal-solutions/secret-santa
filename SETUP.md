# Jani × Eldama Secret Santa — Setup

This project is a single-file frontend plus one importable n8n workflow.

| File | Purpose |
| --- | --- |
| `index.html` | The whole participant and organizer app. Host it directly on GitHub Pages. No build step. |
| `n8n/secret-santa-workflow.json` | Import this into n8n. It contains the webhook, API logic, draw algorithm, and response node. |
| `SETUP.md` | This guide. |

The browser talks directly to n8n. GitHub Pages only serves the static HTML file; participant data and pairings never live in GitHub.

The frontend is deliberately a single `index.html` with inline CSS and JavaScript. There is no build step, package manager, or framework to maintain.

---

## 1. What you need

- An n8n instance reachable over **HTTPS** from the devices that will use the app.
  - n8n Cloud works.
  - A self-hosted n8n instance works if it has a public HTTPS URL.
- A GitHub account for GitHub Pages.
- One person to act as the organizer.
- A long random **admin token**. Use at least 32 random characters. Do not reuse a password.

  You can generate one locally:

  ```bash
  node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
  ```

> If n8n is on plain HTTP and the app is on GitHub Pages, browsers will block the API calls as mixed content. Use an HTTPS n8n URL.

---

## 2. Set up the n8n backend

### 2.1 Import the workflow

1. In n8n, open **Workflows**.
2. Choose **Import from File** and select `n8n/secret-santa-workflow.json`.
3. Open the **Secret Santa API** Code node.
4. Find this line near the top:

   ```js
   const ADMIN_TOKEN_FALLBACK = '';
   ```

5. Set your admin token using **one** of these options:
   - **Self-hosted n8n:** add `SECRET_SANTA_ADMIN_TOKEN=<your-long-random-token>` to the n8n environment and restart n8n.
   - **Any n8n instance (including n8n Cloud):** replace the empty string in the Code node with your token:

     ```js
     const ADMIN_TOKEN_FALLBACK = 'paste-a-long-random-token-here';
     ```

   The token must be at least 16 characters. The backend treats anything shorter as not configured.

   > If you edit the Code node token directly, do not commit or share the populated workflow JSON. The copy in this repository is intentionally blank.

6. Save the workflow.
7. **Activate / publish** the workflow.

### 2.2 Copy the Production webhook URL

Open the **Secret Santa Webhook** node and copy the **Production URL**. It normally looks like:

```text
https://your-n8n.example.com/webhook/secret-santa
```

Use the production URL, not the test URL.

Important: n8n's static data — where this workflow stores participants and pairings — is only saved for published workflow executions. Testing from the n8n editor does not persist data.

### 2.3 CORS

The workflow already sets **Allowed Origins (CORS)** to `*` so GitHub Pages can call it.

If you prefer to lock it down:

1. Open **Secret Santa Webhook**.
2. Go to **Options** → **Allowed Origins (CORS)**.
3. Replace `*` with your Pages origin, for example:

   ```text
   https://your-github-username.github.io
   ```

4. Save and keep the workflow active.

The frontend sends simple form-encoded `POST` requests, so no special preflight configuration is needed.

### 2.4 Verify the backend

Run this from PowerShell (replace the URL):

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri "https://your-n8n.example.com/webhook/secret-santa" `
  -Body @{ action = "health" }
```

Or with curl:

```bash
curl -X POST "https://your-n8n.example.com/webhook/secret-santa" \
  -d "action=health"
```

Expected response:

```json
{
  "ok": true,
  "service": "jani-eldama-secret-santa",
  "adminConfigured": true,
  "drawStatus": "NOT_STARTED"
}
```

If `adminConfigured` is `false`, the admin token was not set correctly.

---

## 3. Publish the frontend on GitHub Pages

1. Create a GitHub repository.
2. Add `index.html` at the repository root.
3. Commit and push:

   ```bash
   git init
   git add index.html SETUP.md n8n/secret-santa-workflow.json
   git commit -m "Add Jani × Eldama Secret Santa"
   git branch -M main
   git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPO.git
   git push -u origin main
   ```

4. In the repository, open **Settings → Pages**.
5. Under **Build and deployment**, choose **Deploy from a branch**.
6. Select branch **main** and folder **/(root)**. Save.
7. Wait for GitHub to show the published Pages URL.

Open the Pages URL. Because there is no build process, future frontend updates are just changes to `index.html`.

### Connect and sign in

1. On the landing page, choose **Organizer sign in**.
2. Enter the n8n Production webhook URL and your admin token.
3. You should land on the **Draw control room**.

The backend URL is saved in the browser. Participants do not need to enter it separately: each invite link carries it in the URL fragment.

---

## 4. Run the exchange

### Add participants

1. Choose **Add participant**.
2. Enter name, email, company, and whether they are participating.
3. Save.
4. Copy the participant's **private invite link** from the dialog and send it to them directly.

Invite links are bearer credentials. Anyone with a link can open that participant's view. Do not post them in a group chat.

### Before the draw

- Toggle an active participant to **Inactive** if they are not taking part.
- Use **Invite** on any participant row to copy their link again.
- Use **Create new link** in the invite dialog if a link was leaked. The old link stops working immediately.
- At least two active participants are required.

### Run the draw

1. Confirm the active counts and participant list.
2. Choose **Run draw**.
3. The draw validates everything first, then saves all pairings together.
4. The admin screen switches to **Complete** and opens the admin-only pairings view.
5. The **Draw integrity verified** panel confirms the key checks.

### Participant reveal

Participants:

1. Open their private invite link.
2. Review their participation status and the gift budget.
3. Choose **Continue**.
4. Choose **Reveal My Person**.
5. See the recipient's name, company, and gift budget.
6. Can return later and review the same assignment.

The first reveal marks the participant as revealed in the organizer dashboard. Refreshing the organizer dashboard shows that status.

### Reset or rerun

Choose **Reset draw** to delete all pairings and unlock the participant list. The app asks for confirmation. This cannot be undone.

Participants cannot see who selected them, cannot see other assignments, and cannot trigger a draw.

### Draw lock behavior

After the draw, participant add/edit/remove/toggle actions are locked. This prevents the participant list from silently changing while pairings exist. If changes are needed:

1. **Reset draw**
2. Edit participants
3. **Run draw** again

---

## 5. How the draw works

- Only participants with `participating: true` are included.
- The backend uses a cryptographically secure Fisher–Yates shuffle and rejection sampling until it has a valid derangement.
- A valid draw has:
  - no self-assignments
  - exactly one giver per active participant
  - exactly one recipient per active participant
  - no duplicate givers or recipients
  - no inactive participants
- The entire assignment set is built and validated in memory before it is saved.
- Existing assignments are never overwritten. A second draw returns `ALREADY_DRAWN` until the organizer resets.
- The API derives the participant from their invite token. Participant requests never accept an arbitrary participant ID.

---

## 6. API reference

The frontend uses a single n8n webhook to keep setup minimal. Every request is a `POST` with `Content-Type: application/x-www-form-urlencoded`.

Fields:

| Field | Required | Description |
| --- | --- | --- |
| `action` | Yes | One of the actions below. |
| `token` | Per action | Participant invite token or organizer admin token. |
| `data` | No | JSON string containing action-specific fields. |

### Action map

The requested application behavior maps to these actions:

| Desired operation | `action` | Auth |
| --- | --- | --- |
| `GET /api/me` | `me` | Participant token |
| `GET /api/me/assignment` | `my_assignment` | Participant token |
| Admin login / draw status | `admin_state` or `admin_login` | Admin token |
| `GET /api/admin/participants` | `admin_participants` | Admin token |
| `POST /api/admin/participants` | `admin_create_participant` | Admin token |
| `PATCH /api/admin/participants/:id` | `admin_update_participant` | Admin token |
| `DELETE /api/admin/participants/:id` | `admin_delete_participant` | Admin token |
| `POST /api/admin/draw` | `admin_run_draw` | Admin token |
| `GET /api/admin/assignments` | `admin_assignments` | Admin token |
| `POST /api/admin/draw/reset` | `admin_reset_draw` | Admin token |
| Rotate a participant invite link | `admin_regenerate_token` | Admin token |
| Public setup check | `health` | None |

### Examples

Health check:

```bash
curl -X POST "https://your-n8n.example.com/webhook/secret-santa" \
  -d "action=health"
```

Admin state:

```bash
curl -X POST "https://your-n8n.example.com/webhook/secret-santa" \
  -d "action=admin_state" \
  -d "token=YOUR_ADMIN_TOKEN"
```

Create a participant:

```bash
curl -X POST "https://your-n8n.example.com/webhook/secret-santa" \
  -d "action=admin_create_participant" \
  -d "token=YOUR_ADMIN_TOKEN" \
  --data-urlencode 'data={"name":"Alice","email":"alice@example.com","company":"Jani","participating":true}'
```

Read the authenticated participant's own assignment:

```bash
curl -X POST "https://your-n8n.example.com/webhook/secret-santa" \
  -d "action=my_assignment" \
  -d "token=PARTICIPANT_INVITE_TOKEN"
```

Run the draw:

```bash
curl -X POST "https://your-n8n.example.com/webhook/secret-santa" \
  -d "action=admin_run_draw" \
  -d "token=YOUR_ADMIN_TOKEN"
```

Errors use this shape:

```json
{
  "ok": false,
  "error": {
    "code": "ALREADY_DRAWN",
    "message": "The Secret Santa draw has already been completed."
  }
}
```

Defined error codes include `PARTICIPANT_NOT_FOUND`, `DRAW_NOT_COMPLETE`, `ALREADY_DRAWN`, `INSUFFICIENT_PARTICIPANTS`, `UNAUTHORIZED`, `VALIDATION_ERROR`, `EMAIL_IN_USE`, and `ADMIN_NOT_CONFIGURED`.

---

## 7. Data storage and limits

This is intentionally a small-event, low-ops implementation.

- Participants and pairings are stored in the n8n workflow's **global static data**.
- Static data persists across normal production webhook executions and n8n restarts.
- Static data is **not available during test executions**.
- Exporting the workflow JSON does not back up its runtime static data. Do not delete or replace the workflow after participants are added unless you are prepared to lose the data.
- n8n marks static data as experimental and does not recommend it for high-frequency concurrent workloads.
- Use one organizer tab for draw actions. The draw refuses a second sequential draw, but two simultaneous draws from different tabs are not a supported pattern.
- For a very large event or strict database-grade transactions/concurrency, replace the static-data store with Postgres or another transactional store.
- The backend runs inside one Code node. There is no email sending, no external database, and no separate server.

This is appropriate for an internal office exchange; it is not a general-purpose identity or event platform.

---

## 8. Security notes

- Participant invite links are bearer secrets. They are placed in the URL **fragment** (`#...`) so the token is not sent to GitHub Pages in the request URL. Treat the whole link as private.
- The admin token is separate and stored in the browser's `sessionStorage`, not committed to the frontend.
- The admin token is sent only to n8n over HTTPS.
- All admin actions are authorization-checked server-side. Hiding buttons is not relied on for security.
- A participant's assignment is looked up from their invite token; the API does not accept a participant ID for `my_assignment`.
- The complete pairings list is only returned by admin actions.
- Participants can see their own profile, participation status, the draw status, and their own recipient. They cannot see who selected them.
- Invite tokens can be rotated from the admin participant table.
- Use a long random admin token. Anyone with the admin token can view and reset the draw.
- The default CORS policy is open (`*`) so the static frontend works without extra setup. Lock it to your Pages origin if you prefer.
- There is no rate limiting or email verification. Keep the n8n webhook URL and admin token private, and only share invite links with the intended participants.

---

## 9. Troubleshooting

| Problem | Likely cause | Fix |
| --- | --- | --- |
| “Could not reach the n8n backend” | Wrong webhook URL, workflow inactive, HTTP/HTTPS mismatch, or CORS blocked | Copy the Production URL again, activate the workflow, and confirm it is HTTPS. |
| `adminConfigured: false` | Token missing or shorter than 16 characters | Set `SECRET_SANTA_ADMIN_TOKEN` or edit `ADMIN_TOKEN_FALLBACK`, save, and keep the workflow active. |
| Organizer sign-in returns 401 | Wrong admin token or workflow not re-saved after editing it | Re-enter the token; check the Code node and environment variable. |
| “That invite link is no longer valid” | Link was regenerated, participant was removed, or workflow data was reset | Copy a new invite link from the organizer dashboard. |
| “At least two participants are required” | Fewer than two active participants | Add or reactivate participants. |
| Draw button is disabled | Fewer than two active participants, or draw already complete | Add participants, or reset the draw first. |
| Add/Edit/Remove are disabled | Assignments already exist | Reset the draw, make changes, then draw again. |
| “The draw hasn’t been completed yet” | Participant opened the reveal screen before the draw | Run the draw from the organizer dashboard. |
| Participants/assignments disappeared | The workflow was deleted, re-imported, or its static data was cleared | Re-add participants. Keep the published workflow in place for the event. |
| Invite works on the organizer laptop but not a phone | The invite carries a webhook URL that is only reachable on that laptop (`localhost`) | Use a public HTTPS n8n webhook URL and generate fresh invite links. |
| Admin cannot see reveal status change | Dashboard is not auto-refreshing | Use **Refresh** on the dashboard. |
| Data is stale after an update | Browser cached an old local state | Reload the page and use **Refresh** in the dashboard. |

---

## 10. Acceptance checklist

Use the organizer dashboard and a participant link to verify:

- [ ] After a draw, zero assignments have the same giver and recipient.
- [ ] Unique givers equal the number of active participants.
- [ ] Unique recipients equal the number of active participants.
- [ ] Every recipient appears exactly once.
- [ ] Inactive participants appear in neither column.
- [ ] A participant sees only their own recipient.
- [ ] The admin can view the complete pairing list.
- [ ] Refreshing or reopening the participant link shows the same assignment.
- [ ] Running the draw again without resetting is rejected.
- [ ] The **Draw integrity verified** panel is green and all checks pass.
- [ ] A participant who has not revealed can be seen as `Assigned`; after revealing and refreshing the dashboard, they become `Revealed`.

---

## 11. Customizing

- **Gift budget:** edit `BUDGET` inside the **Secret Santa API** Code node. Participant and admin views display the values returned by the backend. Also update the fallback budget in `formatBudget()` in `index.html` so the pre-connection landing screen matches.
- **Webhook path:** change the Webhook node path in n8n, then paste the new Production URL into the frontend connection settings.
- **CORS:** change the Webhook node's Allowed Origins option.
- **Frontend branding:** edit the copy and colors in `index.html`; no build process is required.

---

## 12. Quick troubleshooting notes for n8n Cloud

- Use the **Production URL**, not the test URL.
- The workflow must be active.
- If custom environment variables are not available in your plan, use `ADMIN_TOKEN_FALLBACK` in the Code node.
- If n8n applies a shorter webhook timeout on your plan, the app still fits comfortably: every request is a short read/write of static data.
- Keep the workflow around after the event if you want the assignments to remain available.
