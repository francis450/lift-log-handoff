# 04. API contract

Base: `https://<site>/api/method/lift_log.api.`. JSON in, JSON out. Frappe wraps responses as `{"message": ...}`; the shapes below are the `message` value. Errors use Frappe's standard error response; the app shows `_server_messages` text when present.

## Auth

All calls after sign-in send `Authorization: token <api_key>:<api_secret>`.

### `auth.sign_in` (POST, allow_guest)
```json
{ "usr": "njoro@example.com", "pwd": "..." }
```
Verifies the password (`frappe.auth.LoginManager`), checks the Lift Log User role, then creates or rotates the user's API key and secret.
```json
{ "api_key": "abc", "api_secret": "xyz", "user": "njoro@example.com", "full_name": "Njoro", "has_profile": false }
```
Errors: wrong credentials (401), missing role (403, "This account isn't set up for Lift Log yet.").

### `auth.sign_out` (POST)
Rotates the secret so the stored token stops working. Returns `{ "ok": true }`.

## Profile

### `profile.get` (GET)
Returns LL Profile fields plus `program` (the active program's `definition` JSON).

### `profile.save` (POST)
Body: any LL Profile fields. Sending `kcal_target` or `protein_target_g` sets that target's `_manual` flag; sending `{"kcal_target_manual": 0}` switches it back to automatic. The server recalculates automatic targets (docs/03) and returns the saved profile, including `targets_note` when the targets just changed. The app shows the note once, then calls `profile.save` with `{"targets_note": ""}`.

## Today

### `today.get` (GET) `?date=2026-10-07`
```json
{
  "date": "2026-10-07", "week": 1, "phase": "build",
  "routine": { "key": "conditioning", "name": "Conditioning + Core", "note": "...", "items": [ ... ] },
  "session": null,
  "food": { "kcal": 720, "protein_g": 19, "entries": 3 },
  "targets": { "kcal": 2200, "protein_g": 130 },
  "week_strip": [ { "date": "2026-10-05", "trained": false, "fed": false }, "..." ],
  "last_session": { "date": "2026-10-06", "routine_name": "Lower + Walk", "highlight": { "exercise": "Barbell back squat", "sets": [ {"kg": 40, "amount": 8}, "..." ] } }
}
```

## Sessions

### `sessions.get` (GET) `?date=`
Returns the LL Workout Session for the date or `null`.

### `sessions.save` (POST)
```json
{
  "session_date": "2026-10-06", "week": 1, "routine_key": "lower", "routine_name": "Lower + Walk",
  "status": "done", "started_at": "2026-10-06T15:02:00Z", "finished_at": "2026-10-06T15:54:00Z",
  "note": "…", "client_updated_at": "2026-10-06T15:54:10Z",
  "sets": [ { "exercise": "Barbell back squat", "set_no": 1, "kg": 40, "amount": 8, "effort": "easy", "done": true } ]
}
```
Upsert by `(user, session_date)`. Returns the stored session. If the stored `client_updated_at` is newer, returns HTTP 409 with `{ "stored": <session> }`.

### `sessions.history` (GET) `?exercise=Barbell back squat&limit=6`
The last N sessions containing that exercise, newest first, each with only that exercise's sets. Feeds the add-weight rule when the local cache is empty.

### `sessions.month` (GET) `?month=2026-10`
`[ { "date": "2026-10-06", "routine_name": "Lower + Walk", "sets": 12, "status": "done" } ]` for the History calendar.

## Food

### `food.get_day` (GET) `?date=`
`{ "food_date": "...", "entries": [ { "client_id", "meal", "food_name", "portion", "qty", "kcal_per_portion", "protein_per_portion", "source", "added_at" } ] }` or an empty entries list.

### `food.save_day` (POST)
Body: `{ "food_date", "client_updated_at", "entries": [...] }`. Upsert, dedupe by `client_id`, same 409 rule as sessions.

### `food.search` (GET) `?q=cha&limit=20`
```json
{ "recent": [ ...LL Food Item-shaped... ], "mine": [ ... ], "library": [ ... ] }
```
Recent = distinct foods from the user's last 30 days of entries, most recent first.

### `food.save_item` (POST)
`{ "food_name", "portion", "kcal", "protein_g" }` creates or updates the user's custom food with that name.

## Check-ins

### `checkins.get` (GET) `?week=1`
### `checkins.save` (POST)
`{ "week", "checkin_date", "weight_kg", "waist_cm", "forearm_cm", "sleep", "energy" }`. Photo: upload with Frappe's `upload_file` (`is_private=1`, `doctype=LL Check In`, `docname=<name>`) after saving.

## Progress

### `progress.summary` (GET) `?week=1`
```json
{
  "sessions": { "done": 1, "planned_to_date": 2 },
  "protein_days_hit": 0,
  "protein_by_day": [ { "date": "2026-10-05", "protein_g": null }, { "date": "2026-10-06", "protein_g": 125 } ],
  "lifts": [ { "exercise": "Barbell back squat", "start_kg": 40, "best": { "kg": 40, "amount": 8, "date": "2026-10-06" } } ],
  "bodyweight": [ { "week": 1, "kg": 72.0 } ],
  "forearm": [ { "week": 1, "cm": null } ]
}
```

## AI

All AI endpoints check `LL Settings.ai_enabled`, the profile's `ai_enabled`, and the daily limit (HTTP 429 with "Daily AI limit reached. It resets at midnight."). Details and prompts in docs/05.

### `ai.estimate` (POST)
```json
{ "text": "2 chapati and beef stew", "image_base64": null, "image_mime": null }
```
Either or both of text and image. Returns:
```json
{ "items": [ { "food_name": "Chapati", "portion": "2 medium", "kcal": 360, "protein_g": 8 } ], "model": "..." }
```

### `ai.suggest` (POST)
`{ "date": "2026-10-07", "meal": "lunch", "exclude": "[\"Bean stew\"]" }`. The server reads the day's entries, targets, recent foods and the day's routine itself. `exclude` (optional, a JSON list of titles already shown, last 12 kept) is sent by "More ideas" so the new options differ. Returns:
```json
{ "options": [ { "title": "Chicken stew, rice and ndengu", "why": "...", "items": [ { "food_name", "portion", "kcal", "protein_g" } ] } ],
  "budget": { "kcal": 514, "protein_g": 49 } }
```
`budget` is this meal's share of what's left (docs/05).

### `ai.weekly_review` (POST)
`{ "week": 1, "regenerate": false }`. Returns the LL Weekly Review (existing one unless `regenerate`).

### `ai.review_followup` (POST)
`{ "week": 1, "question": "Why move squats up?" }` → `{ "answer": "..." }`. Not stored.

### `ai.apply_changes` (POST)
`{ "week": 1, "accepted": [0, 2] }` (indexes into the review's changes). Marks them accepted. Returns the review.

## Export

### `export.all` (GET)
Everything the user owns as one JSON document: profile, sessions, food days, custom foods, check-ins (photo URLs, not files), reviews.
