# Boomer Facetime

A tiny, single-file local calendar for logging each day as **office**, **remote**, or **holiday**
and tracking a configurable in-office quota — a target percentage of each period's working days that
must be spent in office. Pick the period (**monthly**, **quarterly**, or **yearly**) and the target
(default **60%**, quarterly); the summary shows how many office days you still owe and your current
in-office ratio.

It's one `index.html` — vanilla HTML/CSS/JS, no build step, no dependencies, and runs in any modern
browser. On desktop Chrome/Edge it saves to a real `work_log.json` on your disk (via the File System
Access API); everywhere else — mobile, Firefox, Safari — it saves in the browser itself (IndexedDB),
with Export/Import to move data between them.

## Running it

**Use it now:** [rishikhaneja.github.io/boomer-facetime](https://rishikhaneja.github.io/boomer-facetime/) — hosted on GitHub Pages, nothing to install, works on desktop and mobile.

Prefer to run it yourself? Open `index.html` directly, or serve the folder and open `http://localhost:5199`:

```
python -m http.server 5199
```

Days default to remote on weekdays and holiday on weekends, so you only click the ones that differ —
a day is always one of the three. How your entries are stored depends on the browser:

- **Desktop Chrome / Edge** — you pick a real `work_log.json` on disk and the app writes to it live.
  First run: click **Create a new log** (or **I already have a file → Open an existing log** to adopt
  one). Later runs: click **Continue** to re-grant access. That file *is* your data.
- **Mobile, Firefox, Safari** — no file picker exists, so the app saves in the browser (IndexedDB)
  with no setup. But that data lives only in that browser: **clearing site data erases it**, so back
  up with **File ▸ Export**.

**Backups & moving data:** **File ▸ Export JSON / Export CSV** saves a timestamped snapshot; **File ▸
Import…** replaces current data from a JSON backup — the way to carry your log between browsers or devices.

## The summary

- **Period** — the window covered: monthly, quarterly (default), or yearly. Set it (and the target %)
  from the controls beside the summary heading; both are saved with your data.
- **Working days** — weekdays (Mon–Fri) in the period not marked holiday. Weekdays are remote unless
  marked office; weekends are holidays by default.
- **Office required** = `ceil(quota × working days)`.
- **Office days still needed** = required − office days already logged — the headline number.
- **In-office ratio** = office ÷ working days; turns green only once the true ratio reaches the quota
  (a value just under never rounds up to look compliant).
- A red warning shows when the days still needed can no longer fit in the working days left.

## Development

All logic is inline in `index.html`; there are no other source files, and no lint or test tooling.
To syntax-check after an edit, run the inline script through Node:

```
awk '/^<script>/{f=1;next} /^<\/script>/{f=0} f' index.html | node --check -
```

Architecture:

- **Persistence (hybrid)** — `HAS_FSA` feature-detects the File System Access API. Desktop Chrome/Edge
  writes to the user-chosen `work_log.json`; every other browser stores the state JSON in IndexedDB
  (both share the `worklog` database). `persist()` saves via whichever path is active and `isActive()`
  reports whether a usable store is ready; writes serialize through `writeChain` so rapid clicks can't
  clobber each other. In file mode the browser needs a user gesture to re-grant write permission —
  hence the **Continue** button when a saved file's permission is stale. The connected `work_log.json`
  is gitignored.
- **State** — one object `{ version, quotaPct, period, days, holidays }`: `days` maps ISO date →
  `"office"|"remote"|"holiday"`, `holidays` maps ISO date → name. `normalize()` is the single place
  defaults are applied — route every file-load and import through it. `quotaPct` and `period` persist
  with the data, not just in the UI.
- **Backward compatibility** — any change to the state shape must keep older exports loading cleanly.
  A file written by an earlier version, or one missing newer fields, must import without error or
  data loss: `normalize()` fills every gap with a default and drops nothing it doesn't recognise. Add
  migrations there (keyed off `version`) rather than assuming a field is present, and never rename or
  repurpose an existing field's meaning without a migration that maps the old value forward.
- **Implicit defaults (not stored)** — weekdays default to remote, weekends to holiday; explicit
  `days` entries override. `statusOf` returns `"weekend"` only for lighter styling — it counts as a
  holiday in the math.
- **Summary** — `periodBounds(year, month, period)` yields the window and its heading label;
  `computeSummary` walks it to count working/office/remaining days, with
  `required = ceil(quotaPct/100 × working)`. The displayed ratio is floored so it never turns green
  below the quota.
- **Rendering** — `render()` rebuilds the app's `innerHTML` and re-wires listeners on every change;
  there's no diffing, so every mutation must update `state`, `await persist()`, then `render()`
  (see `setDay` / `setQuota` / `setPeriod`).
- **Visual theme** — a deliberate retro Windows-95 look: silver (`#c0c0c0`) window faces, a navy
  gradient titlebar, a beige desktop, and box-shadow bevels — raised for buttons and frames, sunken
  for input wells and group boxes, inverted on `:active` to read as pressed. Fonts are Tahoma /
  MS Sans Serif with font-smoothing off, and focus shows as a dotted rect. Keep new UI in this idiom:
  reuse the bevel shadow recipes and the `--face` / `--navy` / `--desktop` variables rather than
  reaching for flat modern styling.
