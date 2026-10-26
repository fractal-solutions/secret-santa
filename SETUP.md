# Jani × Eldama Secret Santa — Setup

This project is a single-file frontend plus one importable n8n workflow.

| File | Purpose |
| --- | --- |
| `index.html` | The whole participant and organizer app. Host it directly on GitHub Pages. No build step. |
| `n8n/secret-santa-workflow.json` | Import into n8n. Includes the API, draw logic, Resend email branch, and Data Table backup. |
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
- A **Resend** account with an API key and a verified sending domain, if you want invite emails sent from n8n.
- A native n8n **Data Table** for the backup snapshot.

Generate an admin token locally:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

> If n8n is on plain HTTP and the app is on GitHub Pages, browsers will block the API calls as mixed content. Use an HTTPS n8n URL.

---

## 2. Upgrading from the first workflow version

The updated workflow has extra nodes for Resend, email branching, and Data Table backup.

- **If you have not added real participants yet:** deactivate the old workflow, import the new JSON, finish the setup below, and activate the new workflow. n8n only allows one active webhook per path/method, so do not activate both.
- **If you already added real participants:** do not delete the old workflow until you have copied the participant list somewhere. The new workflow starts with empty static data. Re-create participants in the new workflow (manual entry or CSV import).
- The Production webhook URL normally stays the same if the n8n instance and path are unchanged.

---

## 3. Set up the n8n backend

### 3.1 Install the Resend node

The workflow uses the verified Resend community node.

1. In n8n, open the nodes panel or **Settings → Community nodes**.
2. Search for **Resend** and install the verified node published by Resend.
3. Create a **Resend API** credential using your Resend API key.
4. You will attach that credential to the **Send invite email** node after importing the workflow.

> If you do not want email sending, you can still use the app manually with copied invite links. The email branch only runs when you use an email action.

### 3.2 Import the workflow

1. In n8n, open **Workflows**.
2. Choose **Import from File** and select `n8n/secret-santa-workflow.json`.
3. Open the **Secret Santa API** Code node.
4. Find these lines near the top:

   ```js
   const ADMIN_TOKEN_FALLBACK = '';
   const APP_BASE_URL_FALLBACK = 'https://fractal-solutions.github.io/secret-santa/';
   const EMAIL_FROM_FALLBACK = '';
   ```

5. Configure the values. The app base URL is intentionally hardcoded in the Code node so no n8n environment permissions are needed:

   | Setting | Required | Purpose |
   | --- | --- | --- |
   | `SECRET_SANTA_ADMIN_TOKEN` env or `ADMIN_TOKEN_FALLBACK` | Yes | Organizer login token. |
   | `SECRET_SANTA_EMAIL_FROM` env or `EMAIL_FROM_FALLBACK` | For email | Verified Resend sender, e.g. `Jani Secret Santa <santa@yourdomain.com>`. |
   | `APP_BASE_URL_FALLBACK` | Yes | Public GitHub Pages URL used in emailed invite links. It is set in the Code node and does not require an environment variable. |

   If your Pages domain or repository path changes, update `APP_BASE_URL_FALLBACK` in the Code node.

   You can either use environment variables for the admin token and email sender, or edit the fallback constants directly in the Code node:

   ```js
   const ADMIN_TOKEN_FALLBACK = 'your-long-random-admin-token';
   const APP_BASE_URL_FALLBACK = 'https://fractal-solutions.github.io/secret-santa/';
   const EMAIL_FROM_FALLBACK = 'Jani Secret Santa <santa@yourdomain.com>';
   ```

   The admin token must be at least 16 characters. The email sender must contain an `@` and must belong to a domain verified in Resend.

   > Do not commit or share the workflow after filling in secrets.

6. Attach the **Resend API** credential to the **Send invite email** node.
7. Save the workflow.

### 3.3 Create the Data Table backup

The workflow mirrors the complete state into a native n8n **Data Table** after every API request. This gives you a second copy of the data and a CSV export path from the n8n UI.

1. Open the **Data tables** tab in your n8n project.
2. Create a table named exactly **Secret Santa Backup**.
3. Add three **string** columns:
   - `state_key`
   - `snapshot`
   - `updated_at`
4. Return to the workflow. The **Backup state to Data Table** node is already configured to upsert the row where `state_key = main`.

If the table is missing, the backup node is set to continue on error so the main app still works. Create the table to actually keep the backup.

> The backup is a JSON snapshot row, not a separate relational table. Use the app's **Export CSV** buttons for clean participant and pairing files.

### 3.4 Activate and copy the Production URL

1. Activate / publish the workflow.
2. Open the **Secret Santa Webhook** node and copy the **Production URL**:

   ```text
   https://your-n8n.example.com/webhook/secret-santa
   ```

Use the production URL, not the test URL.

- n8n static data — the primary store — only persists for published workflow executions. Editor test executions do not persist it.
- Email links are built with the webhook URL that received the request. If you send emails during a test execution, the links can point at the test URL. Use Production.
- Data Table backup writes also run through the production webhook.

### 3.5 CORS

The workflow sets **Allowed Origins (CORS)** to `*` so GitHub Pages can call it.

To lock it down:

1. Open **Secret Santa Webhook**.
2. Go to **Options → Allowed Origins (CORS)**.
3. Replace `*` with your Pages origin:

   ```text
   https://your-github-username.github.io
   ```

The frontend sends simple form-encoded `POST` requests, so no special preflight setup is required.

### 3.6 Verify the backend

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri "https://your-n8n.example.com/webhook/secret-santa" `
  -Body @{ action = "health" }
```

Expected:

```json
{
  "ok": true,
  "service": "jani-eldama-secret-santa",
  "adminConfigured": true,
  "emailConfigured": true,
  "drawStatus": "NOT_STARTED"
}
```

- `adminConfigured: false` — set the admin token and re-save/activate.
- `emailConfigured: false` — set `SECRET_SANTA_EMAIL_FROM` or `EMAIL_FROM_FALLBACK`.
- After adding a participant, open **Data tables → Secret Santa Backup** and confirm the `main` row's `updated_at` changed. If it did not, check the table name and columns.

---

## 4. Publish the frontend on GitHub Pages

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
5. Choose **Deploy from a branch**, select **main** and **/(root)**. Save.
6. Wait for the published Pages URL.

Open the Pages URL and choose **Organizer sign in**. Enter the n8n Production webhook URL and your admin token.

The frontend stores the backend URL in the browser. Invite links carry it in the URL fragment, so participants normally never need to configure anything.

---

## 5. Run the exchange

### Add participants manually

1. Choose **Add participant**.
2. Enter name, email, optional phone, company, and participation.
3. Save.
4. Copy the private invite link and share it, or send it by email.

### Import participants from CSV

Use **Import CSV** from the participant area. The import is all-or-nothing: if any row fails validation, no participants are created.

Supported headers:

```csv
name,email,phone,company
Alice Example,alice@example.com,+254 700 000 000,Jani
Bob Example,bob@example.com,+254 700 000 001,Eldama
```

- `name` and `email` are required.
- `phone` is optional.
- `company` is optional if you select a fallback company in the import dialog.
- You can upload a `.csv` file or paste CSV text.
- You can choose to email invite links immediately after import.
- There is a **CSV template** button in the import dialog.
- Maximum 500 participants per import.

### Send invite emails with Resend

There are three ways to send:

- **Email all:** sends to all active participants.
- **Invite → Send email:** sends one participant's link.
- **Email invite links after import:** sends to the newly imported participants only.

The email includes:

- The participant's private invite link.
- The gift budget.
- A reminder to keep the link private.
- A draw-status-aware message.

The link is built from:

- The configured `APP_BASE_URL_FALLBACK` in the Code node.
- The n8n webhook URL configured in the frontend (normally the Production URL).
- The participant's private invite token.

If email is not configured, the app shows a notice and disables the email buttons so the organizer knows what is missing.

### Export data

- **Participants → Export CSV** — name, email, phone, company, participation, assignment status, revealed status, invite link, created date.
- **Pairings → Export CSV** — giver and recipient details, including recipient email and phone, reveal status, and assignment date.

These files come from the authenticated admin state, not from the public participant view.

### Run the draw

1. Confirm the participant list and active counts.
2. Choose **Run draw**.
3. All assignments are generated and validated before anything is saved.
4. The admin screen switches to **Complete** and opens the admin-only pairings view.
5. **Draw integrity verified** confirms the draw constraints.

### Participant experience

Participants:

1. Open their private invite link.
2. Review their participation status and gift budget.
3. Choose **Continue**.
4. Choose **Reveal My Person**.
5. See the recipient's:
   - name
   - company
   - email
   - phone, if provided
   - gift budget
6. Can return later and review the same assignment.

The first reveal updates the participant's status in the organizer dashboard after a refresh.

Only that participant (and the organizer/admin views) can retrieve that recipient's contact information.

### Reset or rerun

**Reset draw** deletes all pairings, resets all reveal statuses, and unlocks the participant list. It asks for confirmation and cannot be undone.

After a draw, participant add/edit/remove/toggle is locked until the draw is reset.

---

## 6. How the draw works

- Only participants with `participating: true` are included.
- The backend uses a cryptographically secure Fisher–Yates shuffle and rejection sampling until it finds a valid derangement.
- A valid draw has:
  - no self-assignments
  - exactly one giver per active participant
  - exactly one recipient per active participant
  - no duplicate givers or recipients
  - no inactive participants
- The complete assignment set is built and validated in memory before it is saved.
- Existing assignments are never silently overwritten.
- The API derives the participant from their invite token. Participant requests never accept an arbitrary participant ID.

---

## 7. Where the data lives

There are now three layers:

1. **Primary storage:** global static data inside the published n8n workflow.
   - Persists across normal webhook requests and n8n restarts.
   - Not available in editor test executions.
   - Tied to that specific workflow. Re-importing creates a new workflow with empty data.
2. **Backup mirror:** the `Secret Santa Backup` Data Table.
   - Updated after every API request.
   - Contains a JSON snapshot including participant contact data and invite tokens.
   - Downloadable from the n8n Data tables UI.
3. **On-demand exports:** participant and pairing CSV files from the admin dashboard.

Do not delete or replace the workflow after real participants are added unless you have a backup and are prepared to recover manually.

For very large events or strict database-grade transactions/concurrency, replace the static-data store with Postgres or another transactional store.

---

## 8. API reference

The frontend uses one n8n webhook. Every request is a `POST` with `Content-Type: application/x-www-form-urlencoded`:

| Field | Required | Description |
| --- | --- | --- |
| `action` | Yes | Action name. |
| `token` | Per action | Participant invite token or organizer admin token. |
| `data` | No | JSON string with action-specific fields. |

### Action map

| Operation | `action` | Auth |
| --- | --- | --- |
| `GET /api/me` | `me` | Participant token |
| `GET /api/me/assignment` | `my_assignment` | Participant token |
| Admin state / draw status | `admin_state` / `admin_login` | Admin token |
| `GET /api/admin/participants` | `admin_participants` | Admin token |
| Create one participant | `admin_create_participant` | Admin token |
| Create many participants | `admin_bulk_create_participants` | Admin token |
| Update participant | `admin_update_participant` | Admin token |
| Remove participant | `admin_delete_participant` | Admin token |
| Rotate invite link | `admin_regenerate_token` | Admin token |
| Send invite emails | `admin_email_invites` | Admin token |
| Run draw | `admin_run_draw` | Admin token |
| Get complete pairings | `admin_assignments` | Admin token |
| Reset draw | `admin_reset_draw` | Admin token |
| Public setup check | `health` | None |

### Email action

Send to all active participants:

```json
{
  "sendAll": true,
  "appBaseUrl": "https://YOUR-USERNAME.github.io/YOUR-REPO/"
}
```

Send to selected participants:

```json
{
  "participantIds": ["participant-uuid"],
  "appBaseUrl": "https://YOUR-USERNAME.github.io/YOUR-REPO/"
}
```

The backend does not send email directly from the Code node. It emits email items, and the n8n **Send invite email** Resend node sends them.

### CSV bulk action

```json
{
  "participants": [
    {
      "name": "Alice Example",
      "email": "alice@example.com",
      "phone": "+254 700 000 000",
      "company": "Jani",
      "participating": true
    }
  ]
}
```

### Examples

Health check:

```bash
curl -X POST "https://your-n8n.example.com/webhook/secret-santa" \
  -d "action=health"
```

Send all invite emails:

```bash
curl -X POST "https://your-n8n.example.com/webhook/secret-santa" \
  -d "action=admin_email_invites" \
  -d "token=YOUR_ADMIN_TOKEN" \
  --data-urlencode 'data={"sendAll":true,"appBaseUrl":"https://YOUR-USERNAME.github.io/YOUR-REPO/"}'
```

Create a participant:

```bash
curl -X POST "https://your-n8n.example.com/webhook/secret-santa" \
  -d "action=admin_create_participant" \
  -d "token=YOUR_ADMIN_TOKEN" \
  --data-urlencode 'data={"name":"Alice","email":"alice@example.com","phone":"+254 700 000 000","company":"Jani","participating":true}'
```

Read the authenticated participant's own assignment:

```bash
curl -X POST "https://your-n8n.example.com/webhook/secret-santa" \
  -d "action=my_assignment" \
  -d "token=PARTICIPANT_INVITE_TOKEN"
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

Relevant error codes include `PARTICIPANT_NOT_FOUND`, `DRAW_NOT_COMPLETE`, `ALREADY_DRAWN`, `INSUFFICIENT_PARTICIPANTS`, `UNAUTHORIZED`, `VALIDATION_ERROR`, `EMAIL_IN_USE`, `ADMIN_NOT_CONFIGURED`, `EMAIL_NOT_CONFIGURED`, `EMAIL_SEND_FAILED`, `APP_URL_REQUIRED`, and `NO_RECIPIENTS`.

---

## 9. Security notes

- Participant invite links are bearer secrets. They live in the URL fragment (`#...`) so the token is not sent to GitHub Pages in the request URL.
- The invitation email contains that private link. Treat email delivery and inbox access as part of the security boundary.
- The organizer admin token is separate, sent only to n8n over HTTPS, and stored in the browser's `sessionStorage`.
- Admin authorization is enforced server-side.
- A participant's assignment is looked up from their invite token; the API does not accept a participant ID for `my_assignment`.
- The complete pairings list is returned only by admin actions.
- **Recipient contact information** (email and phone) is returned to the assigned giver after reveal and is visible in admin views. It is not exposed to other participants.
- The Data Table backup contains PII and invite tokens. Protect access to your n8n instance and project.
- CSV exports can contain PII and invite tokens. Store and share them carefully.
- Invite tokens can be rotated from the admin participant table.
- The default CORS policy is open (`*`). Lock it to your Pages origin if you prefer.
- There is no rate limiting or email verification system. Keep the n8n webhook URL and admin token private.
- The Resend sender domain must be verified; otherwise Resend will reject sends.

---

## 10. Troubleshooting

| Problem | Likely cause | Fix |
| --- | --- | --- |
| “Could not reach the n8n backend” | Wrong URL, inactive workflow, HTTP/HTTPS mismatch, or CORS | Copy the Production URL, activate the workflow, use HTTPS. |
| `adminConfigured: false` | Token missing or under 16 characters | Set the admin token, save, activate. |
| `emailConfigured: false` | Sender not set | Set `SECRET_SANTA_EMAIL_FROM` or `EMAIL_FROM_FALLBACK`. |
| “The frontend URL is not valid” | `APP_BASE_URL_FALLBACK` is incorrect, or the active workflow is running an older published version | Set `APP_BASE_URL_FALLBACK` to the full `https://...` Pages URL, save, and publish/activate the workflow. |
| Email buttons disabled | Email not configured | Configure the sender and refresh the dashboard. |
| “Could not find node type n8n-nodes-resend.resend” | Resend node not installed | Install the verified Resend community node, then reopen the workflow. |
| `EMAIL_SEND_FAILED` | Missing/invalid Resend credential, unverified sender domain, or provider rejection | Attach the Resend API credential and verify the sender domain. Check n8n executions. Some sends may have succeeded before the failure, so retry individual invitations rather than blasting everyone again. |
| Invite emails arrive with test URLs | Emails sent from the n8n editor/test webhook | Use the Production webhook URL in the app. |
| Email all only reaches some people | Only active participants are included | Activate the missing participants or send individual invites. |
| Data Table backup not updating | Table missing or columns wrong | Create `Secret Santa Backup` with `state_key`, `snapshot`, `updated_at`. |
| CSV import rejected | Missing name/email header, invalid email, duplicate email, or invalid company | Fix the reported row and retry. Imports are all-or-nothing. |
| Phone not shown on reveal | Participant has no phone stored | Edit the participant and add a phone number. |
| Draw button disabled | Fewer than two active participants or draw complete | Add/activate participants, or reset the draw. |
| Add/Edit/Import disabled | Assignments exist | Reset the draw, make changes, then draw again. |
| “The draw hasn’t been completed yet” | Participant opened reveal before draw | Run the draw. |
| Participants/assignments disappeared | Workflow was deleted, re-imported, or static data cleared | Restore from your backup/export. Keep the workflow in place. |
| Invite works on desktop but not phone | Invite carries a localhost/unreachable webhook URL | Use a public HTTPS n8n URL and regenerate links. |
| Admin reveal status stale | Dashboard not auto-refreshing | Use **Refresh**. |

---

## 11. Acceptance checklist

- [ ] Zero self-assignments after the draw.
- [ ] Unique givers equal active participants.
- [ ] Unique recipients equal active participants.
- [ ] Every recipient appears exactly once.
- [ ] Inactive participants are excluded.
- [ ] A participant sees only their own recipient.
- [ ] The recipient's email and phone are shown only to their giver after reveal.
- [ ] The admin can view the complete pairing list.
- [ ] Refreshing or reopening the participant link shows the same assignment.
- [ ] Running the draw again without resetting is rejected.
- [ ] **Draw integrity verified** is green.
- [ ] A participant changes from `Assigned` to `Revealed` after they open their assignment, then dashboard refresh.
- [ ] CSV import creates all valid rows and creates nothing when one row is invalid.
- [ ] Participant and pairing CSV exports download correctly.
- [ ] An invite email reaches a real address through Resend and its link opens the correct participant.
- [ ] The Data Table backup row updates when participants change.

---

## 12. Customizing

- **Gift budget:** edit `BUDGET` in the **Secret Santa API** Code node and the fallback in `formatBudget()` in `index.html`.
- **Email sender:** set `SECRET_SANTA_EMAIL_FROM` or `EMAIL_FROM_FALLBACK`. The sender must be verified in Resend.
- **Email links base URL:** edit `APP_BASE_URL_FALLBACK` in the Code node if the Pages domain or path changes.
- **Participant cap:** adjust `MAX_BULK_PARTICIPANTS` and `MAX_INVITE_EMAILS` in the Code node.
- **Webhook path:** change the Webhook node path, then update the frontend connection settings.
- **CORS:** change the Webhook node's Allowed Origins option.
- **Backup table:** rename the Data Table and update the **Backup state to Data Table** node accordingly.
- **Frontend branding:** edit copy and colors in `index.html`; no build step is required.

---

## 13. Quick n8n Cloud notes

- Install the verified Resend node before activating the workflow.
- If custom environment variables are unavailable, use the fallback constants in the Code node.
- Use the Production URL, not the test URL.
- The workflow must remain active.
- n8n Cloud execution limits apply to the email batch. The app caps a single email send at 100 recipients.
- Keep the workflow published after the event if you want the data and invite links to remain available.
