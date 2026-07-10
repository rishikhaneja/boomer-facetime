# Boomer Facetime

A tiny, single-file local calendar for logging each day as **office**, **remote**, or **holiday**
and tracking a configurable in-office quota — a target percentage of each period's working days that
must be spent in office. Pick the period (**monthly**, **quarterly**, or **yearly**) and the target
(default **60%**, quarterly); the summary shows how many office days you still owe and your current
in-office ratio.

It's one `index.html` — vanilla HTML/CSS/JS, no build step, no dependencies. Chrome or Edge only: it
saves straight to a file on disk via the File System Access API, which Firefox/Safari lack (they get
a "Chrome required" gate).

## Running it

**Use it now:** [rishikhaneja.github.io/boomer-facetime](https://rishikhaneja.github.io/boomer-facetime/) — hosted on GitHub Pages, nothing to install (Chrome or Edge).

Prefer to run it yourself? Double-click `index.html`, or serve the folder and open `http://localhost:5199`:

```
python -m http.server 5199
```

**First run:** click **Create a new log** and choose where to save your `work_log.json` (or use
**I already have a file → Open an existing log** to adopt one). Weekdays default to remote and
weekends to holiday, so you only click the days that differ — a day is always one of the three.
Every change writes straight to `work_log.json`; that file *is* your data.

**Later runs:** the browser remembers the file — click **Continue** once (browsers require a gesture
to re-grant write access) and your data loads.

**Backups:** `work_log.json` is a plain file, so copy it like any other. **File ▸ Export JSON /
Export CSV** saves a timestamped snapshot; **File ▸ Import…** replaces current data from a JSON backup.

## The summary

- **Period** — the window covered: monthly, quarterly (default), or yearly. Set it (and the target %)
  from the controls beside the summary heading; both persist in `work_log.json`.
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

- **Persistence** — data writes straight to a user-chosen `work_log.json` via the File System Access
  API. The file handle is cached in IndexedDB (db/store `worklog`) so it survives reloads, but the
  browser needs a user gesture to re-grant write permission — hence the **Continue** button on the
  connect screen when permission is stale. Writes serialize through `writeChain` so rapid clicks can't clobber each
  other. `work_log.json` *is* the data and is gitignored.
- **State** — one object `{ version, quotaPct, period, days, holidays }`: `days` maps ISO date →
  `"office"|"remote"|"holiday"`, `holidays` maps ISO date → name. `normalize()` is the single place
  defaults are applied — route every file-load and import through it. `quotaPct` and `period` persist
  in the file, not just UI state.
- **Implicit defaults (not stored)** — weekdays default to remote, weekends to holiday; explicit
  `days` entries override. `statusOf` returns `"weekend"` only for lighter styling — it counts as a
  holiday in the math.
- **Summary** — `periodBounds(year, month, period)` yields the window and its heading label;
  `computeSummary` walks it to count working/office/remaining days, with
  `required = ceil(quotaPct/100 × working)`. The displayed ratio is floored so it never turns green
  below the quota.
- **Rendering** — `render()` rebuilds the app's `innerHTML` and re-wires listeners on every change;
  there's no diffing, so every mutation must update `state`, `await writeToFile()`, then `render()`
  (see `setDay` / `setQuota` / `setPeriod`).
