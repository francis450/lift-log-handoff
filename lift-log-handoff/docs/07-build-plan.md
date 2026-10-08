# 07. Build plan

One milestone per Claude Code session. Each ends with its checks passing and a short "test this on your phone" list for the user.

## M1. Backend foundation (`lift_log`)

Build: the app skeleton, all DocTypes (docs/03), Lift Log User role, LL Settings, seed loader, and these endpoints from docs/04: `auth.*`, `profile.*`, `today.get`, `sessions.*`, `food.*`, `checkins.*`, `progress.summary`, `export.all`. AI endpoints return HTTP 501 for now.

Done when:
- `bench --site <site> run-tests --app lift_log` passes, including permission isolation between two test users, upsert idempotency, 409 on stale writes and client_id dedupe.
- Seed loads 1 program, 32 exercises, 30 foods, and running it twice changes nothing.
- `curl` with a token returns `today.get` for 2026-10-07 with routine "Conditioning + Core", week 1.

## M2. Mobile foundation (`lift-log-mobile`)

Build: Expo app with TypeScript, fonts, theme tokens and the shared components from docs/02, the four-tab layout, Sign in, Profile and targets (with the target formulas), Settings (targets, session time, sign out), the API client with the token in SecureStore, and the local SQLite store with a sync queue (retry with backoff; 409 → take the server copy and show "Updated from another device").

Done when:
- Sign in against the M1 site works on a real Android phone; a user without the role sees the role message.
- Targets: 72 kg with no height/age gives 2,200 kcal and 130 g; entering height and age recalculates.
- Today shows real data from `today.get`.
- `npm test`, `npm run lint` and `npm run typecheck` pass.

## M3. Training

Build: program engine (week, phase, routine for a date, from the program JSON), Train screen, Workout screen (set rows, prefill, effort picker, rest timer, add set, notes, keep-awake, resume after kill), the add-weight rule with its full test table, Workout summary, History calendar and day detail, Home deload screen, missed-session states.

Done when:
- The rule's tests cover every branch in docs/01 (at least 15 cases), including: first session with a starting weight, top of range, two easy sessions, two hard sessions, bodyweight, test week, and a weekly-review weight change added on top.
- Logging the 6 Oct example (squat 40 x 8, 7, 7) then opening the next lower day shows "Stay 40 kg, beat your reps".
- Airplane mode: a whole workout logs, and syncs when back online with no duplicate session.
- Week 4 dates show the home deload on Mon, Wed and Fri only.

## M4. Food logging (without AI)

Build: Food day, meters, meals, edit/delete entries, Add food sheet with Search and Custom tabs, recent foods, save to my foods, undo snackbar, the 10-second double-add guard, offline sync.

Done when:
- Adding a food updates both meters immediately and survives an app restart offline.
- Tapping Add twice quickly asks "Add it again?" and a confirmed repeat produces two entries; a network retry of the same write produces one.
- Search "cha" lists Chapati and Chai.

Then run the migration (docs/06) and compare totals: 6 Oct and 7 Oct food days and the 6 Oct session match the export.

## M5. AI estimates and suggestions

Build: `lift_log/ai/*`, `ai.estimate` and `ai.suggest` with limits and logging (docs/05); Photo and Describe tabs; the suggestion card on Food.

Done when:
- A photo of a plate returns editable items in under 15 s on mobile data; Cancel stops it.
- Without an API key set, the AI tabs show "AI features aren't set up on your server" and the rest of the app works.
- The 41st call in a day returns the limit message.
- No prompt text or image is stored on the server (check LL AI Log and the File list).

## M6. Check-in and progress

Build: Sunday check-in (with measurement-due logic and optional private photo), Progress screen (stats, protein bars, lifts table, weight and forearm charts).

Done when:
- A check-in updates the profile weight and appears on Progress.
- Protein bars match the food days for the week; the lifts table shows the program start weight until a set is logged.

## M7. Weekly review

Build: `ai.weekly_review`, `ai.review_followup`, `ai.apply_changes`, the Python port of the add-weight rule (sharing the M3 test table), Weekly review screen, accepted weight changes feeding the next week's suggestions.

Done when:
- A seeded test week produces a review whose stat tiles equal the server-computed numbers.
- Applying "Squat: +2.5 kg" makes the next lower session suggest 2.5 kg more, with the reason "From your weekly review".
- Invalid changes from the model (unknown exercise, 5 kg jump) are dropped.

## M8. Reminders, export and release

Build: local notifications (docs/01), Export my data through the share sheet, app icon and splash (accent blue, dumbbell mark), EAS `preview` profile, a short `INSTALL.md` for sideloading the APK.

Done when:
- An APK from `eas build -p android --profile preview` installs on the user's phone and passes a full day: workout with effort, three meals (one by photo), and a check-in.
- Turning a reminder off in Settings cancels its scheduled notifications.
