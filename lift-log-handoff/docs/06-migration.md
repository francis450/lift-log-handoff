# 06. Migrating the current web log

The current log is a Claude artifact page. Its data was exported on 7 Oct 2026 into `data/export/`:

| Folder | Contents | Maps to |
|---|---|---|
| `sessions/<date>.json` | One workout per date: `routine`, `week`, `note`, `sets: { "<exercise>": [ {kg, amt} ] }`, `savedAt` | LL Workout Session + LL Set Entry |
| `meals/<date>.json` | `items: [ {id, meal, n, portion, qty, k, p, src} ]`, `savedAt` | LL Food Day + LL Food Entry |
| `settings/myfoods.json` | `items: [ {n, portion, k, p} ]` | LL Food Item (user's custom foods) |
| (none yet) | Check-ins: none logged at export time | LL Check In |

Each file is wrapped as `{ "id", "data", "version", "updatedAt" }`; the record is under `data`.

## Field mapping

**Sessions**
- `session_date` = file name; `week`, `note` from data; `routine_name` = `routine`; `routine_key` from the program's routine with that name for that weekday; `status = done`; `finished_at` = `savedAt`; `client_updated_at` = `savedAt`.
- For each exercise in `sets`, in order, `set_no` = index + 1, `kg` (blank string → null), `amount` = `amt`, `effort` blank (the old log had no effort), `done = 1`.

**Food**
- `food_date` = file name; for each item: `client_id` = `id`, `meal` = lowercased `meal` ("Snacks" → `snacks`), `food_name` = `n`, `portion`, `qty`, `kcal_per_portion` = `k`, `protein_per_portion` = `p`, `source = migrated`, `added_at` = `savedAt`.

**Custom foods**
- `food_name` = `n`, `portion`, `kcal` = `k`, `protein_g` = `p`, `user` = the signed-in user.

## Script

`lift_log/setup/migrate_web_log.py`, run with
`bench --site dashboard.erpkenya.com execute lift_log.setup.migrate_web_log.run --kwargs "{'folder': '/path/to/data/export', 'user': '<njoro's login email>'}"`.
Idempotent (upserts by date and client_id). Prints a summary: sessions, sets, food days, entries, custom foods.

The script also creates the user's LL Profile if missing: weight 72 kg, age 25, height blank, goal "Lose fat, get stronger", training days Mon–Fri, session time 18:00, active program `block-2-2026`. Targets then follow the auto-calculation rule in docs/01 (2,200 kcal until height is entered, 130 g protein).

## The duplicate dinner on 6 Oct (resolved)

The 6 Oct dinner was logged twice by a double tap. At the user's request it was removed from the web log on 7 Oct, before this export, so `meals/2026-10-06.json` is already correct: 6 entries, 1,645 kcal, 125 g protein. No special handling needed.

## Keep using the web log until

The app's M4 milestone (food logging) passes on the phone. After the final migration run, stop logging in the web page.
