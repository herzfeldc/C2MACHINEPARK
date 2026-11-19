# C2 Machine Park – Complete Documentation

Version 1.1.0 · Windows x64 desktop application · Last updated 2026-11-19

C2 Machine Park is a desktop tool for monitoring and maintaining the C2 (ctwo) virtual machine
park. It shows every C2 machine and every resource group. Busy, free, offline and parked states are
visible at a glance, along with live session timing. Users can move, remove, park and comment on
machines with drag and drop.

---

## Contents

1. [Quick start (end users)](#1-quick-start-end-users)
2. [Deployment package](#2-deployment-package)
3. [Signing in](#3-signing-in)
4. [Screen layout](#4-screen-layout)
5. [Machine status and colors](#5-machine-status-and-colors)
6. [Session timing: pending, runtime and AV PENDING](#6-session-timing-pending-runtime-and-av-pending)
7. [Hover information](#7-hover-information)
8. [Filtering, searching and sorting](#8-filtering-searching-and-sorting)
9. [Parallel-process frames](#9-parallel-process-frames)
10. [Display options](#10-display-options)
11. [Save and load settings](#11-save-and-load-settings)
12. [Auto refresh](#12-auto-refresh)
13. [Selecting machines](#13-selecting-machines)
14. [Machine actions (add, remove, park)](#14-machine-actions-add-remove-park)
15. [Comments and Resource JSON](#15-comments-and-resource-json)
16. [Resource groups](#16-resource-groups)
17. [C2 API calls used](#17-c2-api-calls-used)
18. [Security and data storage](#18-security-and-data-storage)
19. [Troubleshooting](#19-troubleshooting)
20. [Developer guide](#20-developer-guide)
21. [Change history](#21-change-history)
22. [Image credits](#22-image-credits)

---

## 1. Quick start (end users)

1. Download `C2-Machine-Park-<version>-win-x64.zip` from the repository's **Releases** page.
2. Right-click the ZIP → **Properties** → tick **Unblock** (if shown) → **OK**.
3. Right-click → **Extract All…** and choose a folder you can write to, for example
   `C:\Users\<you>\C2-Machine-Park` or a USB drive. Do **not** extract into `C:\Program Files`.
4. Open the extracted `C2-Machine-Park` folder and double-click **`C2 Machine Park.exe`**.
5. Enter your C2 authentication token and the API address, then click **Connect**.

No installation, administrator rights, Node.js, browser or runtime is required. Everything the app
needs is in the `C2-Machine-Park` folder.

Windows SmartScreen may warn that the app is unrecognized because it is not code-signed. Choose
**More info → Run anyway** only if the ZIP came from the official repository.

To update, close the app, extract the new ZIP and copy your old `data` folder into the new
folder if you want to keep browser-side state. To uninstall, delete the folder.

---

## 2. Deployment package

The ZIP contains exactly one top-level folder:

```text
C2-Machine-Park\
├── C2 Machine Park.exe      ← start the application here
├── START-HERE.txt           ← short instructions
├── docs\
│   ├── C2-Machine-Park-Documentation.html   ← this document (open in any browser)
│   └── C2-Machine-Park-Documentation.md
├── resources\app.asar       ← the application (interface and logic)
├── locales\, *.dll, *.pak, *.bin, *.dat   ← bundled Electron/Chromium runtime
└── data\                    ← created on first start (cache, browser storage, logs)
```

* **Self-contained:** the Electron runtime (Chromium and Node.js) is bundled. Nothing is written
  to `%APPDATA%` or the registry. All runtime data goes to the `data` subfolder next to the EXE.
* **Portable:** the folder can be moved or copied. It can run from any writable location,
  including a network share or USB drive. Run it from a location where the user has write access.
* **Requirements:** Windows 10/11 x64, and network access to the C2 API server. The API must
  allow requests from the desktop app (CORS for the `file://` origin).
* **Integrity:** each release includes a `.sha256` file. To verify a download, run
  `Get-FileHash .\C2-Machine-Park-<version>-win-x64.zip -Algorithm SHA256` in PowerShell and compare
  the result with that file.

---

## 3. Signing in

On start, the app shows **Machine Park Maintenance** with two fields:

| Field | Meaning |
|-------|---------|
| Authentication Token | Your personal C2 API bearer token. Required. Masked while typing. |
| ctwo API Address | Base URL of the C2 public API, pre-filled with the production address. |

The desktop app does **not** ship with any token. The token is kept in memory only and is
not saved to disk or to settings files. You enter it again each time the app starts.

---

## 4. Screen layout

**Header (top)**

* Title *C2 Machine Park* and subtitle.
* **Save settings / Load settings** (see [section 11](#11-save-and-load-settings)).
* **Auto Refresh** switch and **Frequency** (1–300 seconds).
* Drop zones: **Remove VM** (recycling bins), **Park VM** (parking lot).

**All C2 Machines (upper section)**

* Header row: section title, name filter, **Sort machines by**, shown/total count and
  **Display options**.
* Counters: total machines, busy, free, offline, parked, **PENDING** and **AV PENDING**.
* **VM with comments only** switch and **Unselect All**.
* The machine grid shows every visible, non-deleted C2 machine. The section grows to fit all
  machines, and the page scrolls.

**Resource groups (lower section)**

* One collapsible card per resource group, with the group's counters, **Refresh** and **Save VMs**.
* **Sort by** for groups (name or machine count, ascending/descending) and a shared
  **Sort machines by** for machines inside groups.

Every machine tile is the same size (134.4 × 61.6 CSS px) and scales with zoom (Ctrl + mouse
wheel). Long names are shortened with "…", and the full name is in the hover popup. Names ending
in ` *` have a comment.

---

## 5. Machine status and colors

| Status | Rule (from the C2 API) | Counter color |
|--------|------------------------|---------------|
| **Busy** | `isOnline = true`, `isOutaged = false` and at least one *Undecided* session | Blue |
| **Free** | Online, not outaged, no Undecided session | Green |
| **Offline** | `isOnline = false` | Red/grey |
| **Parked** | `isOutaged = true` (planned downtime) | Parked color |

* Total = free + busy + offline + parked. Machines whose status has not loaded yet are
  excluded until it is known.
* Busy machines are **not** counted as free.
* Busy detection uses `sessions?outcomes=Undecided&resourceIds=<id>` per online, non-outaged
  machine at each refresh. Failed checks are reported and never mark a machine busy.
* Statuses and sessions load before the lists appear, so colors and sorting are correct
  from the first screen.

---

## 6. Session timing: pending, runtime and AV PENDING

All timing uses the **first Undecided session** of a busy machine, in API order. Later sessions
are never substituted.

### Time zone handling

The C2 API returns timestamps without a time-zone marker (for example `2026-11-18T14:56:21.374`).
These are **UTC**, and the app reads them as UTC. Timestamps with `Z` or an explicit offset keep
their stated zone. The current time is taken as an absolute instant, so the PC's local time zone
and daylight saving time never affect results.

### Pending badge (upper-left corner)

* Value: `started − created` in seconds, rounded to the nearest second.
* Shown only when the value is **≥ 2 s**. 0 s and 1 s are hidden.
* `--` means the session is missing, not started, or has invalid timestamps. The reason is
  shown on hover.

### Runtime badge (upper-right corner)

* Value: `now − started`, **rounded down to whole minutes** (for example `4m`). Under a minute
  shows `0m`.
* Updates at each full-minute boundary without any API call.
* `--` with a clock warning means the start time is in the future (clock difference) or invalid.
* With a **runtime threshold** set, only runtimes ≥ threshold are shown.
* In **dot mode**, a large red dot replaces the number once the threshold is reached.

### PENDING counter (red number)

The number of sessions returned by `sessions?take=500&outcomes=Undecided` with `started = null`.
Sessions of workflow **670** are excluded. This counts sessions, not machines, and filters do not
affect it. It shows N/A if the check failed.

### AV PENDING counter (blue)

The average start delay (`started − created`) over all unique busy machines. Zero delays are
included, while missing or invalid timings are excluded. The value is rounded to whole seconds,
and N/A means no valid timings.

---

## 7. Hover information

Hovering over a machine in either list shows:

* Name, ID, online/outaged status and resource groups.
* For busy machines, when **Show workflow/task names and session timestamps** is on:
  * the workflow name (with a leading `C2:` removed),
  * the workflow task name (from `workflowTaskId`),
  * `Created (UTC)` and `Started (UTC)` of the first session, or *Unavailable*.
* Maintained custom fields (`linkedCustomFields`) as `name: value`, with empty or NULL values
  omitted.

Possible messages are *No linked workflow*, *No linked task*, *Workflow unavailable* and
*Task unavailable*. The last two include error details. Each distinct workflow or task is
fetched once per refresh.

---

## 8. Filtering, searching and sorting

* **Name filter** (All C2 Machines): case-insensitive partial match. It shows the number
  of matches and does not affect counters, groups or selections.
* **Status counters** are clickable filters. Click one to filter, or **Ctrl+Click** to combine
  several statuses.
* **VM with comments only**: shows only machines with a non-blank `Comment`.
* **Sort machines by** (All C2 Machines, and separately for groups):
  * **Status** (default): busy → free → parked → offline → unknown. Busy machines are sorted
    by first-session **Process ID** (numeric IDs as numbers), then by name. Missing Process IDs
    come last among busy machines.
  * **Name**: alphabetical.
* **Group Sort by**: name or machine count, with an Ascending/Descending toggle. Equal counts
  are ordered by name.

---

## 9. Parallel-process frames

In **Status** sorting, busy machines whose first session shares a **Process ID** are kept
together and outlined with a 4 px frame:

* **vivid orange** in All C2 Machines,
* **black** inside resource groups.

If a group does not fit in the rest of a row, it starts on the next row. Larger groups wrap,
and the frame follows the outline across rows. Corner badges and dots are drawn above the frame.
Frames do not block clicks or drag selection, and they update after filtering, refreshing,
expanding, resizing and zooming. Single machines and missing Process IDs get no frame.

---

## 10. Display options

Click **Display options** in the All C2 Machines header. These settings apply to both lists.

| Option | Default | Effect |
|--------|---------|--------|
| Show workflow/task names and session timestamps | On | Workflow/task names plus Created/Started (UTC) in hovers. Off skips these API lookups; turning it on again refreshes statuses. |
| Show pending time | On | Upper-left start-delay badge. |
| Show runtime | On | Upper-right runtime badge. Off stops its timers. |
| Runtime threshold (minutes) | 0 | Show runtime only from this many whole minutes. |
| Runtime dot mode | Off | Large red dot instead of the number when the threshold is reached. |
| Show parallel-process frames | On | Frames around machines sharing a Process ID. Sorting is unaffected. |

Only the workflow/task option causes extra API calls. All other displays reuse loaded data.

---

## 11. Save and load settings

**Save settings** writes a JSON file (default name `C2-Machine-Park-settings.json`) through a
Save As dialog. **Load settings** opens a file chooser and restores it.

Saved settings: display options, runtime threshold and dot mode, name/status/comment filters,
all sort controls, auto-refresh on/off and frequency, and resource-group expansion state.

**Never saved:** tokens, API address, machine selections or machine data. Files are versioned
(`format: c2-machine-park-settings`, `version: 1`). Invalid, unknown or tampered files are
rejected with an error and no setting is changed.

---

## 12. Auto refresh

Turn on **Auto Refresh** and set **Frequency** (1–300 s). Each cycle refreshes machine statuses,
session activity, PENDING and, if enabled, workflow/task names. Overlapping refreshes share
one request cycle. Errors are collected in a refresh error dialog.

---

## 13. Selecting machines

* **Ctrl+Click** a tile to add it to or remove it from the selection.
* **Drag a rectangle** starting from empty space to select all visible tiles it touches.
  Collapsed or clipped tiles are excluded. **Esc** cancels the rectangle.
* **Unselect All** clears the selection.
* Dragging any selected tile drags the whole selection.

---

## 14. Machine actions (add, remove, park)

All write actions ask for confirmation before any API request is sent.

| Action | How | API |
|--------|-----|-----|
| Add to group | Drag machine(s) onto a resource group | `POST resourceGroup/{id}/addResource` |
| Remove from group | Drag machine(s) from a group onto **Remove VM** | `POST resourceGroup/{id}/removeResource` per machine |
| Park | Drag machine(s) onto **Park VM**, enter a reason and confirm | `POST plannedDowntimes` from **now** to **now + 1 year**; the exact UTC start and end are shown before sending |
| Save VMs | **Save VMs** on a group card | Downloads `<group>_machines.txt`; no API write |

Bulk operations report each machine's result.

---

## 15. Comments and Resource JSON

* **Double-click** a machine to open the **Comment** editor. Only the custom field named
  `Comment` can be edited.
  **Save Comment in C2** (after confirmation) sends
  `POST customfields/AddOrUpdateCustomFieldToEntity` with `entityType: "Resource"`. An empty text
  clears the comment. Failures keep your text and are not retried automatically.
* **Ctrl+double-click** opens a read-only **Resource JSON** window with a fresh
  `GET resource/{id}` response. Close it with **Esc** or **Close**.

---

## 16. Resource groups

* Groups marked `deleted: true` and BOOT-named groups are excluded.
* Group machine lists exclude `visible: false` or `deleted: true` resources.
* Click a group name or its arrow (or press Enter/Space) to collapse or expand it.
* **Refresh** on a group reloads only that group.

---

## 17. C2 API calls used

All calls go to the configured API address with `Authorization: Bearer <token>`.

| Purpose | Method and path |
|---------|----------------|
| Machines | `GET resources?skip=0&take=1000`, `GET resource/{id}` |
| Groups | `GET resourcegroups?skip=0&take=300`, `GET resources?resourceGroupId={id}&skip=0&take=200` |
| Busy detection | `GET sessions?outcomes=Undecided&resourceIds={id}` |
| PENDING | `GET sessions?take=500&outcomes=Undecided` |
| Workflow name | `GET workflows/{workflowId}` (optional) |
| Task name | `GET workflowTasks/{workflowTaskId}` (optional) |
| Add to group | `POST resourceGroup/{id}/addResource` (after confirmation) |
| Remove from group | `POST resourceGroup/{id}/removeResource` (after confirmation) |
| Park | `POST plannedDowntimes` (after confirmation) |
| Comment | `POST customfields/AddOrUpdateCustomFieldToEntity` (after confirmation) |

---

## 18. Security and data storage

* No credentials are included in the package. The token is entered at start and kept in
  memory only.
* The packaging script refuses to build a ZIP if a known token is found in the build output.
* Chromium security stays enabled: sandbox, context isolation, no Node.js in the page, web
  security on, and certificate errors are not bypassed. New windows, navigation and permission
  requests are blocked.
* Local data (cache, storage, logs) stays in `C2-Machine-Park\data`. Delete this folder to reset.
* The EXE is unsigned. Distribute only through the private repository.

---

## 19. Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| SmartScreen warning | Unsigned app. Use **More info → Run anyway** for official downloads. |
| App does not start, or files are missing | The ZIP was not fully extracted, or it was started from inside the ZIP. Extract everything first. |
| "Access denied" or nothing is saved | The folder is read-only (for example Program Files). Move it to your user folder. |
| Network/CORS error after Connect | Check VPN/network and the API address. The API must allow the desktop origin. |
| 401/403 errors | The token is wrong or expired. Restart and enter a new token. |
| Runtime shows `--` with a clock warning | The PC clock is ahead of or behind the server. Sync Windows time. |
| Workflow names missing | Display option is off, or the lookups failed (see the refresh error dialog). |
| Settings file rejected | The file was edited or is from an unsupported version. Save a new one. |

---

## 20. Developer guide

**Technology:** React 19 + Vite 8 (UI), Electron 45 (desktop shell), electron-builder 26,
oxlint, and Node's built-in test runner.

```text
C2MACHINEPARK\
├── desktop\main.cjs          Electron main process (secure window, portable data folder)
├── scripts\package-zip.ps1   Creates the deployment ZIP and runs the token check
├── docs\                     This documentation and START-HERE.txt
├── src\App.jsx               Header, sign-in, auto refresh, remove/park dialogs
├── src\components\           MachineManager, Zone, MachineBox, DisplayOptions, SettingsFiles, …
├── src\services\api.js       API helpers, UTC timestamp parsing, timing and hover formatting
├── src\processConnections.js Process-frame geometry
├── src\viewSettings.js       Settings model and file validation
└── src\*.test.js             Unit tests
```

| Command | Purpose |
|---------|---------|
| `npm install` | Install development dependencies (developers only) |
| `npm run dev` | Web development server with hot reload |
| `npm run desktop` | Build the desktop UI and start Electron |
| `npm run lint` | oxlint |
| `node --test src/*.test.js` | Unit tests |
| `npm run build:folder` | Build `release\win-unpacked` |
| `npm run package:zip` | Build the folder and create `release\C2-Machine-Park-<version>-win-x64.zip` + `.sha256` |
| `npm run build:exe` | Alternative single-file portable EXE |

**Release procedure**

1. Bump `version` in `package.json` and update [Change history](#21-change-history).
2. Run the tests, then `npm run package:zip`.
3. Start `release\package\C2-Machine-Park\C2 Machine Park.exe` and check sign-in.
4. Create a GitHub release `v<version>`, then attach the ZIP and its `.sha256` file.

---

## 21. Change history

**1.1.0 (2026-11-19)**

* Deployment ZIP with a single self-contained folder, portable `data` folder and documentation.
* Runtime badge (whole minutes), threshold and red-dot mode.
* UTC-correct timestamp handling (fixed runtimes that were one hour too long).
* Display options dialog. Save/Load settings files.
* Parallel-process frames (orange or black, 4 px), replacing the earlier connection bars.
* Created/Started (UTC) in hovers. Compact filter and sort controls.
* Pending-time badges, AV PENDING and PENDING counters, Process ID sorting.

**1.0.0 (2026-11-06)**

* First portable Windows build: machine lists, resource groups, busy detection,
  add/remove/park, comments, Resource JSON, filters, sorting and auto refresh.

---

## 22. Image credits

All bundled images are public domain (CC0) from publicdomainpictures.net:
*Empty Parking Lots* by Circe Denyer (Park VM), *Recycling Bins* by Lynn Greyling (Remove VM) and
*Floppy disks* by Petr Kratochvil (Save VMs).
