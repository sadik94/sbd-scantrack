# SBD ScanTrack

Scanning operations and equipment tracking, running **inside Trimble Connect for Browser** as a project extension.

Survey teams use it to log daily 3D-scanning work, track scanning equipment (e.g. SX12, GS-200), watch calibration expiry, and see utilization and productivity per project and per asset. The data is shared by the team: it lives as a file in the Trimble Connect project.

---

## Features

| Tab | What it does |
|---|---|
| **Dashboard** | Calibration warnings (expired or due within 30 days); totals for scans, scanning days, standby days, hours, utilization, scans/hour, m² covered and downtime; per-project table; data-quality checks; recent activity |
| **Scanning Log** | Daily entry form (date, asset, project, area, scans, m², planned/actual/downtime hours, reason, condition, operator, remarks) with validation; edit and delete |
| **Equipment** | Asset register (status, site, project, responsible person, dates, calibration expiry) with per-asset statistics |
| **Setup & Lists** | Manage projects, areas per project, operators, downtime reasons, equipment statuses and conditions |
| **Sync & Connection** | Region, access token, manual sync, Excel import/export, JSON backup/restore, diagnostics log |

**How the numbers are calculated**
- **Utilization** = scanning hours ÷ planned hours
- **Scans / hour** = scans ÷ scanning hours
- **Standby days** = days with entries − days with at least one scan

---

## How it works

```
Trimble Connect project (browser)
   │  reads manifest.json  ──►  "app is at …/index.html"
   ▼
index.html shown inside Trimble (iframe)
   ├─ Workspace API  → current project + user access token (after "Allow")
   └─ Core API 2.0   → reads/writes  "SBD ScanTrack Data/sbd_scantrack_data.json"
                        in the Trimble project
```

- **Hosting (GitHub Pages)** serves only the app's code. **Your data is never stored on GitHub.**
- **Shared data**: one JSON file in the project folder *SBD ScanTrack Data*. The app downloads it every 30 seconds, and uploads about 1.5 seconds after any change. Each upload is a new version of the same file.
- **Local copy**: a copy is kept in the browser (`localStorage`), so the app keeps working if a sync fails. Changes are merged by record, and the newest edit wins.
- **Deletes** are flagged rather than removed, so they reach other users.
- **Permissions**: users without write access to the project get **read-only** mode.

---

## Files

| File | Purpose |
|---|---|
| `index.html` | The whole app (HTML, CSS and JavaScript in one file) |
| `manifest.json` | Extension manifest that Trimble Connect reads |
| `icon.svg` | Menu icon |
| `README.md` | This file |

External libraries, loaded from CDNs:
- Trimble Connect Workspace API: `components.connect.trimble.com/trimble-connect-workspace-api`
- SheetJS 0.18.5 for Excel: `cdnjs.cloudflare.com`

---

## Setup

### 1. Host the files (GitHub Pages)
1. Create a **public** repository named `sbd-scantrack` and upload `index.html`, `manifest.json` and `icon.svg`.
2. Edit `manifest.json` to use your own address:
   ```json
   {
     "url": "https://<your-username>.github.io/sbd-scantrack/index.html",
     "title": "SBD ScanTrack",
     "icon": "https://<your-username>.github.io/sbd-scantrack/icon.svg",
     "description": "Scanning operations and equipment tracking inside Trimble Connect",
     "extensionType": ["project"]
   }
   ```
3. Go to **Settings › Pages** and set **Deploy from a branch**, **main**, **/(root)**, then click **Save**.
4. After 1–2 minutes, check that `https://<your-username>.github.io/sbd-scantrack/manifest.json` shows the JSON.

> Any static HTTPS host works instead: Vercel, Netlify, Cloudflare Pages, Azure Static Web Apps, or a company web server. The manifest must be readable from Trimble's site (CORS); GitHub Pages already allows this. `localhost` and Kaggle do **not** work for shared use.

### 2. Add it to a Trimble Connect project
You must be a **Project Administrator**.
1. Open the project in **web.connect.trimble.com**.
2. Go to **Settings › Apps & Capabilities › + Add Custom › Manifest URL**.
3. Paste `https://<your-username>.github.io/sbd-scantrack/manifest.json` and click **Add**.
4. When asked *"Allow extension 'SBD ScanTrack' to access Trimble Connect on your behalf?"*, click **Allow**.
5. Refresh the page. **SBD ScanTrack** appears in the left menu.

Once added, the extension is available to **all project members**. Each user clicks **Allow** the first time they open it.

### 3. First use
1. Open **SBD ScanTrack › Sync & Connection**. The badge should show **Synced HH:MM**.
2. Replace the sample lists in **Setup & Lists** with your own.
3. Optionally import an existing `SBD_ScanTrack.xlsx` on **Sync & Connection**.
4. Check that **Data › SBD ScanTrack Data › sbd_scantrack_data.json** appears in the project after your first change.

---

## Updating the app
1. Edit the files in GitHub and commit. Pages republishes in about 1 minute.
2. Browsers may keep the old version for **up to 10 minutes**. Wait, then reload the Trimble page.
3. Changing only `index.html` needs nothing else. If you change the hosting address, **remove and re-add** the extension in Trimble so it reads the new manifest.

---

## Data and backup
- **Export to Excel**: three sheets: *Daily Scanning Log*, *Equipment Register*, *Project Areas*.
- **Download data file (.json)**: a complete backup that you can restore with **Load data file**, which merges it into the current data.
- Trimble Connect keeps the version history of `sbd_scantrack_data.json`.
- One data file is kept **per Trimble project**. Pick one "home" project for all sites if you want a single combined dashboard.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Extension not in the left menu after adding | Page not refreshed, or browser blocking embedded pages (e.g. Brave Shields) | Ctrl+F5; turn Shields off for `web.connect.trimble.com`, or use Chrome/Edge; switch the extension off and on |
| Status shows "Enabled", not "SBD ScanTrack ready" | App hasn't been opened yet | Open it from the left menu |
| `401 INVALID_SESSION` in Diagnostics | Token not active yet, or access was denied | **Sync now**; or **Request access token** and click Allow |
| `400` on `/files/fs/commit` | Old upload code (see Changes below), or an old cached copy | Make sure Fix 1 is in `index.html`; wait 10 min and reload |
| Badge says "Read-only" | No write permission in the project | Ask a project admin for edit rights |
| "Sync problem – changes kept on this device" | Any sync error | Read **Diagnostics**; your changes are safe in the browser and upload on the next successful sync |
| Manifest URL shows 404 | Pages not enabled or wrong path | Check **Settings › Pages** and the spelling of the URL |

In **Diagnostics**: `200`/`201` = OK, `400` = bad request, `401` = not authorised, `403` = no permission.

---

## Changes from the original version
1. **Upload commit fix.** `POST /files/fs/commit` now sends only `{ "uploadId": … }`, which matches Trimble's official `trimble-connect-sdk`. The original also sent `parentId`, `parentType` and `name`, and Trimble rejected that with *400 Failed to read request*.
2. **Local-date check.** The "date cannot be in the future" check now uses the local date instead of UTC. Before, entries dated today were rejected between 00:00 and 04:00 Dubai time (UTC+4).

## Known limitations / to-do
- **Excel import:** log rows with the same date, asset, project and area, and with no file reference or remarks, get the same ID and overwrite each other.
- **Calibration expiry** isn't imported from Excel; enter it in the Equipment tab.
- **SheetJS 0.18.5** has a known prototype-pollution issue when reading crafted `.xlsx` files. Upgrade to 0.19.3 or later (from `cdn.sheetjs.com`).
- **No locking:** if two people upload at exactly the same moment, one person's change can briefly drop out of the shared file. It comes back on their next sync.
- **Standby days** on the dashboard are counted for the whole team (calendar days), not per asset per day.
- **Region "Auto"** guesses from the project's location text; set the region manually if sync fails.

## Possible next steps
These come from the *Trimble Connect Integration Study*:
- Move the daily log and metrics to a small backend database.
- Keep the equipment register in Trimble **Organizer**.
- Build a task module on the **Issues API**; Core ToDos are being deprecated.
- Add retries and monitoring.

---

## References
- Trimble Connect for Browser – Project Extensions: https://docs.browser.connect.trimble.com/projects/project-settings/extensions
- Extend Trimble Connect (developer guide): https://developer.trimble.com/docs/connect/guides/extend/
- Workspace API: https://components.connect.trimble.com/trimble-connect-workspace-api/index.html
- Trimble Connect SDK (npm): https://www.npmjs.com/package/trimble-connect-sdk
