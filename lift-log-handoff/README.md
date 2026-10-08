# Lift Log: build handoff

Everything Claude Code needs to build **Lift Log**, an Android app (React Native / Expo) for logging gym sessions and food, backed by a custom Frappe app on an ERPNext site, with Claude features (photo calorie estimates, meal suggestions, weekly review).

Nothing here has been built yet. This folder is the spec.

## What's in the folder

| Path | What it is |
|---|---|
| `CLAUDE.md` | Rules and conventions for Claude Code. Copy it to the root of each repo. |
| `docs/01-product-spec.md` | What the app does, screen by screen, and the business rules (progression, effort, phases, targets). |
| `docs/02-design-spec.md` | Colors, type, spacing, components and per-screen layout notes. |
| `docs/03-backend-erpnext.md` | The `lift_log` Frappe app: DocTypes, fields, permissions, server logic. |
| `docs/04-api-contract.md` | Every endpoint the mobile app calls, with request and response JSON. |
| `docs/05-ai-features.md` | The three Claude features: prompts, JSON schemas, models, cost limits. |
| `docs/06-migration.md` | Moving existing logs from the current web log into ERPNext. |
| `docs/07-build-plan.md` | Milestones in build order, each with acceptance criteria. |
| `design/screens/*.dc.html` | The 15 screen designs (412 x 915) as markup with inline styles, for exact values. They need the design canvas to render; view them there. |
| `data/program-block2.json` | The full 12-week training program as data. |
| `data/exercises.json` | Exercise master list: type, weight increment, form-video link. |
| `data/foods-seed.json` | 30 common Nairobi foods with typical portions, kcal and protein. |
| `data/export/` | Real logs exported from the current web log on 7 Oct 2026, for migration. |

## How to use it with Claude Code

Two repos, built in this order:

1. **`lift_log`** (Frappe app, installed on `dashboard.erpkenya.com`). Start Claude Code in a new folder with this handoff folder available, then:
   > Read `CLAUDE.md`, `docs/01`, `docs/03`, `docs/04`, `docs/05` and `docs/07`. Build milestone M1 (backend) of `docs/07-build-plan.md`. Stop at the end of M1 and list what to test.
2. **`lift-log-mobile`** (Expo app). Once the backend endpoints answer:
   > Read `CLAUDE.md`, `docs/01`, `docs/02`, `docs/04` and `docs/07`, and the screens in `design/screens/`. Build milestones M2 onward, one at a time.

Work one milestone per session and check the acceptance criteria before moving on.

## Decisions already made

- Android first. React Native with Expo (managed workflow), TypeScript.
- Backend: a custom Frappe app on an existing ERPNext site. No other paid services.
- The Claude API key lives only on the server (Frappe `LL Settings`). The app never sees it.
- Offline-first logging: sets and food save on the phone first and sync when online.
- Effort rating per set (Easy / Right / Hard) feeds both the add-weight rule and the weekly review.
- Install path: EAS build to an APK you sideload. Play Store later, same code.

## Settled with the user (7 Oct 2026)

- **Site:** `dashboard.erpkenya.com`.
- **Profile:** 72 kg, age 25. Height will be entered later in Settings; calorie target recalculates automatically when it is (docs/01, "Targets").
- **6 Oct duplicate dinner:** removed from the web log before export. The export is clean.
