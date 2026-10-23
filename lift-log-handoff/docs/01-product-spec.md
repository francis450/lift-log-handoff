# 01. Product spec

## Purpose

One person (Njoro, 72 kg, Nairobi) follows a 12-week gym block, Monday to Friday at 6 PM, with two goals: lose fat and get stronger, while keeping leg size down and growing the forearms. The app makes three things effortless:

1. Logging every set during the workout, with a clear call on when to add weight.
2. Logging food fast enough that it actually happens (search, photo, describe, repeat), against calorie and protein targets.
3. A weekly review by Claude that reads training, effort, food and check-ins together and proposes concrete changes for next week.

The data model supports more than one user (each user sees only their own data), but there is no social or sharing feature.

## Screens

Design files are in `design/screens/`. Bottom tabs: **Today, Train, Food, Progress**.

| Screen | Design file | Reached from |
|---|---|---|
| Sign in | `SignIn.dc.html` | First launch, sign out |
| Profile and targets | `Setup.dc.html` | After first sign-in |
| Today | `Main.dc.html` | Tab |
| Settings | `Settings.dc.html` | Gear icon on Today |
| Train (block and week) | `Train.dc.html` | Tab |
| Workout in progress | `Workout.dc.html` | Start workout (Today or Train) |
| Workout summary | `WorkoutDone.dc.html` | Finishing a workout; a past session in History |
| Home deload | `HomeDeload.dc.html` | Week 4 days on Train/Today |
| History | `History.dc.html` | History button on Train |
| Food day | `Food.dc.html` | Tab |
| Add food: photo | `AddFood.dc.html` | Add food button, Photo tab |
| Add food: search | `AddFoodSearch.dc.html` | Add food, Search tab (Describe and Custom tabs follow the same sheet) |
| Progress | `Progress.dc.html` | Tab |
| Sunday check-in | `CheckIn.dc.html` | Progress, Sunday reminder |
| Weekly review | `WeeklyReview.dc.html` | Progress, after check-in |

The weekly review design shows example numbers. Real content comes from the review endpoint.

## The program

`data/program-block2.json` is the source of truth: 12 weeks from Mon 5 Oct 2026, phases Build (wk 1–3), Home deload (wk 4, exam week 26–30 Oct), Strength (wk 5–8), Peak (wk 9–11), Test (wk 12). Each week maps weekdays to a routine (list of exercises with sets and a rep target) or to nothing (rest). Sunday is check-in day.

- Program week for a date: `floor((date - start_date) / 7) + 1`.
- Rep targets are strings: `"6-8"` is a range, `"5"` a single value, `"max"` means as many as possible.
- Exercises flagged `finisher: true` are the forearm finisher and display under a "Forearm finisher" label.
- Week 12 routines are tests: the app shows "Work up to your heaviest clean 5" instead of an add-weight call.
- The program is data, not code. A later block is a new JSON loaded on the server.

## Logging a workout

1. Opening a training day (from Today, Train or History) shows a preview: each exercise with its target and add-weight call, no timer, and nothing saved. **Start workout** opens the routine at the first exercise and creates the session record with `status = in_progress` and `started_at` = now. A session that was opened but has nothing logged counts as not started: it shows Start (not Resume), opens as a preview, and Start resets its `started_at`.
2. Each exercise shows: name, target (`3 × 6–8 reps`), a Form button (opens the exercise's `form_url`), the add-weight call (below), and one row per target set.
3. Weight fields are prefilled with the suggested weight. Reps fields are empty, with last session's reps as placeholder text.
4. Ticking a set marks it done. A row of three effort buttons appears under it: **Easy** (3+ more reps were possible), **Right** (1–2 left), **Hard** (nothing left or missed reps). Effort is optional.
5. Ticking a set starts the rest timer (default 90 s; 2 min for exercises flagged `main`). The timer offers +30 s and Skip, and vibrates at zero.
6. "+ Add set" adds a row. Rows with no reps entered are not saved.
7. Next exercise moves on; the progress bar shows exercise N of M. A note can be added at any point.
8. Finishing saves `status = done`, `finished_at`, and opens the summary.
9. Everything saves locally as it is entered. Killing the app mid-workout resumes where it left off.

### Workout summary

Duration, sets logged, total kg lifted (sum of kg × reps over load exercises), the effort split as a bar (easy / right / hard counts), a one-line read of the effort ("Most sets felt easy. There's room to go heavier."), a "Next time" call per exercise (the add-weight rule applied to this session), and the note.

### Add-weight rule

One pure function, `nextCall(exercise, target, history) -> { call: 'up' | 'hold' | 'down' | 'start' | 'test' | 'bodyweight', kg?: number, reason: string }`. Unit-test every branch.

Definitions: `S` = target sets, `[a, b]` = rep range (single value means `a = b`), `inc` = the exercise's `increment_kg`, `L` = the most recent session containing this exercise, `W` = heaviest kg in `L`, *work sets* = sets in `L` at `W`.

1. Exercise in a test routine: `test`, "Work up past {best} kg if it moves well".
2. Exercise type `bodyweight` or `time`, or target `max`: `bodyweight`, "Beat last time" plus last session's numbers.
3. No history: `start` with the program's `starting_weights_kg` value if present ("Start around 30 kg"), else "Pick a weight you can do with clean form".
4. Work sets count ≥ `S` and every work set's reps ≥ `b`: `up`, `W + inc`, "Hit the top of the range".
5. Early bump from effort. In each of the last two sessions with this exercise at weight `W`: every work set's reps ≥ `a`, at least two thirds of work sets rated Easy, none rated Hard. Then `up`, `W + inc`, "Felt easy two sessions running".
6. Struggling. In each of the last two sessions at `W`: at least half the work sets rated Hard and at least half below `a` reps. Then `down`, `W - inc` (not below 0), "Two hard sessions: drop a step and rebuild".
7. Otherwise `hold`, `W`, "Beat your reps".

`distance` exercises (farmers carry) use metres as reps with the same rule. Deload-week sessions are ignored as history for gym exercises. After a phase change the current week's rep targets apply.

Display: up = green chip, hold = amber chip, start = blue chip, down = amber chip with the reason.

### Home deload (week 4)

Monday, Wednesday and Friday only. A three-round checklist of bodyweight moves with a round counter, no weights and no add-weight calls. Finishing saves a session (`routine_key = home_deload`).

### Missed sessions

A weekday routine whose date has passed with no session is shown as Missed on Train and History. If a weekly review change "Move a missed Monday to Saturday" was accepted, a missed weekday routine also appears on that week's Saturday.

## Food

### Food day

- Two meters: calories eaten vs target, protein vs target, with "X of Y left". The calorie meter turns amber when over target; the protein meter stays green.
- A suggestion card ("Suggested lunch") from the suggestion endpoint, requested when the Food tab opens if the day has ≥ 300 kcal left and no suggestion was fetched in the last 3 hours. Each option is sized to that meal's share of what's left, so the meals still to come keep theirs (docs/05). The options sit side by side: swipe to compare, with an "N of M" counter and page dots. Each shows "Leaves {kcal} kcal · {g} g protein for later". The last page is "More ideas", which adds new options after the ones shown (up to 12) rather than replacing them. "Add to {meal}" logs its foods.
- Meals: Breakfast, Lunch, Dinner, Snacks, each with entries (name, quantity × portion, kcal, protein) and a per-meal total. Long-press an entry to edit quantity or delete.
- Add food button (extended FAB with camera icon) opens the Add food sheet with the meal preset by time of day: before 11:00 Breakfast, before 16:00 Lunch, before 21:00 Dinner, else Snacks.
- Previous / next day arrows.

### Add food sheet: four tabs

- **Photo:** camera opens immediately. The photo plus optional text goes to the estimate endpoint. Results list each food with portion, kcal, protein and a checkbox; numbers are editable. "Add N foods" logs the checked ones.
- **Describe:** free text ("2 chapati, beef stew, chai with sugar") to the same estimate endpoint.
- **Search:** searches the user's recent foods, saved custom foods, then the seed library. Pick one, set portions with − / + in 0.5 steps, add.
- **Custom:** name, portion, kcal and protein per portion, number of portions, and "Save to my foods" (on by default).

### Undo and double-add guard

- Every add shows a snackbar for 5 s: "Added {food} to {meal}" with Undo.
- Adding the same food to the same meal within 10 s of an identical add asks "Add it again?" first. (The current web log recorded one dinner twice from a double tap; this prevents that.)
- Entries carry a client-generated `client_id`; the server ignores a second write of the same id.

## Check-in (Sunday)

- Body weight with − / + in 0.1 kg steps, prefilled with last week's value.
- Waist and forearm (cm), marked "Due this week" when the last measurement is 14+ days old or missing.
- Sleep and gym energy, 1–5 each.
- Progress photo every 4 weeks (optional, stored privately on the server).
- Save leads to the weekly review.

## Weekly review

Generated on demand after the Sunday check-in, or from Progress for any finished week. Shows four stats (sessions done of planned, average protein, average kcal on logged days, weight and change), "Went well" and "To change" paragraphs, and up to four proposed changes with switches. "Apply N changes to week X" saves the accepted ones as plan adjustments (docs/03) that the app applies to the next week's suggestions. A follow-up question box sends one question plus the review context to the same endpoint. Details in docs/05.

## Progress

- Sessions logged vs planned to date in this block; days the protein target was hit this week.
- Protein this week: seven bars with a dashed target line; bars at or over target are dark green, under target light green, empty days a flat grey stub.
- Lifts table: Back squat, Romanian deadlift, Hip thrust, Bench press, Deadlift, Row, Overhead press. Start = first logged top set, or the program's starting weight until one is logged. Best = heaviest kg (ties broken by reps).
- Body weight and forearm line charts once there are two or more check-ins; before that a card pointing to the next check-in.
- Buttons to Weekly review and Check in.

## Today

Date and program week, phase pill, settings gear, week strip (two dots per day: training logged, food logged), tonight's session card with Start workout (or "Rest day" / "Check-in day"), food so far with both meters and a one-line hint (protein left), and the last session's key lift with its next call.

## Targets

- Protein target: `round(1.8 × weight_kg / 5) × 5` g (130 g at 72 kg).
- Calorie target with height and age: Mifflin-St Jeor for men, `10 × kg + 6.25 × cm − 5 × age + 5`, times 1.55, minus 500, rounded to the nearest 50. Without height or age: 2,200 kcal. (Example: 72 kg, age 25, 175 cm gives 2,150 kcal.)
- **Targets recalculate automatically** whenever weight, height or age changes: entering height later in Settings, or a new Sunday weigh-in. The server does this (docs/03), so every device sees the same numbers. When targets change, the app shows a one-time note on Today: "Targets updated: 2,150 kcal, 130 g protein (height added)."
- Editing a target by hand in Settings switches that target to manual, and it stops recalculating. A "Use automatic target" button switches it back.
- Settings shows: "Lower by 150–200 if your weight stalls for 2 weeks."
- Known now: weight 72 kg, age 25. Height is still to be entered.

## Reminders (local notifications)

- 30 min before session time on training days.
- 20:00 "Log dinner" if dinner is empty that day.
- Sunday 09:00 check-in.
Each can be switched off in Settings.

## Settings

Daily targets, program and session time, units (kg only for now), reminder switches, server and connection status, AI features on/off, Export my data (JSON of everything the user owns, from the export endpoint, saved through the share sheet), Sign out.

## Out of scope for version 1

iPhone build, Play Store listing, social features, barcode scanning, wearables, multiple programs running at once, imperial units.
