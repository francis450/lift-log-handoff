# 03. Backend: the `lift_log` Frappe app

## Hosting

**Site: `dashboard.erpkenya.com`** (chosen by the user). `lift_log` needs only the Frappe framework, so it installs on that site whether or not ERPNext is installed there.

- `bench get-app <lift_log repo>` then `bench --site dashboard.erpkenya.com install-app lift_log`, then the seed command.
- HTTPS must be on (`bench setup lets-encrypt` if it isn't); the app refuses plain `http://` servers.
- Anyone with System Manager on that site can read Lift Log records in desk. Keep that list to people you're happy to share training and food data with.

## App layout

```
lift_log/
  hooks.py
  lift_log/            # module "Lift Log"
    doctype/...        # DocTypes below
  api/
    auth.py  profile.py  today.py  sessions.py  food.py  checkins.py  progress.py  ai.py  export.py
  ai/
    client.py          # Anthropic client, limits, logging
    prompts.py         # prompt builders (docs/05)
  setup/
    seed.py            # loads data/*.json
    install.py         # creates role, settings defaults
  tests/
```

## Role and permissions

- Role **Lift Log User** (desk access off). Every user DocType below grants this role read, write, create and delete with **If Owner** checked.
- Every whitelisted method uses `frappe.session.user` as the user and rejects Guest.
- Seed library records (`LL Food Item` with `is_seed = 1`, `LL Exercise`, `LL Program`) are readable by Lift Log User, writable only by System Manager.

## DocTypes

Naming: `LL` prefix. `user` fields are Link to User, default `frappe.session.user`, read-only in forms.

### LL Profile (one per user, named by user)
| Field | Type | Notes |
|---|---|---|
| user | Link User | unique |
| weight_kg | Float | latest known; updated by check-ins |
| height_cm | Int | optional |
| age | Int | optional |
| goal | Select | Lose fat, get stronger / Build muscle / Maintain |
| kcal_target | Int | |
| kcal_target_manual | Check | 1 once the user edits it by hand |
| protein_target_g | Int | |
| protein_target_manual | Check | 1 once the user edits it by hand |
| targets_note | Small Text | last automatic change, shown once by the app, then cleared |
| session_time | Time | default 18:00 |
| training_days | Data | e.g. `mon,tue,wed,thu,fri` |
| time_zone | Data | default Africa/Nairobi |
| active_program | Link LL Program | |
| ai_enabled | Check | default 1 |
| reminders | JSON | `{workout: true, dinner: true, checkin: true}` |

### LL Program (shared library)
| Field | Type | Notes |
|---|---|---|
| program_id | Data | unique, e.g. `block-2-2026` |
| program_name | Data | |
| start_date / end_date | Date | |
| definition | JSON | the whole of `data/program-block2.json` |

### LL Exercise (shared library)
| Field | Type | Notes |
|---|---|---|
| exercise_name | Data | unique, matches program JSON |
| exercise_type | Select | load / bodyweight / time / distance |
| equipment | Select | blank / barbell / dumbbell / cable / machine / bodyweight |
| increment_kg | Float | the equipment's weight step; suggested weights are multiples of it (dumbbells 2.5, barbell 2.5, cable stack 5) |
| pair | Check | two dumbbells at once: kg is per dumbbell and counts twice in total lifted |
| form_url | Data | |

### LL Workout Session (one per user per date)
| Field | Type | Notes |
|---|---|---|
| user | Link User | |
| session_date | Date | unique together with user |
| program | Link LL Program | |
| week | Int | |
| routine_key | Data | e.g. `lower`, `home_deload`, `test_bench` |
| routine_name | Data | |
| status | Select | in_progress / done |
| started_at / finished_at | Datetime | UTC |
| note | Small Text | |
| sets | Table LL Set Entry | |
| client_updated_at | Datetime | for last-write-wins sync |

### LL Set Entry (child of LL Workout Session)
| Field | Type | Notes |
|---|---|---|
| exercise | Link LL Exercise | |
| set_no | Int | 1-based within exercise; an L and R row of a per-side set share it |
| side | Select | blank / L / R, for per-side exercises |
| kg | Float | blank for bodyweight/time; per dumbbell for dumbbell exercises |
| amount | Float | reps, seconds or metres per exercise type |
| effort | Select | blank / easy / right / hard |
| done | Check | |

### LL Food Day (one per user per date)
| Field | Type | Notes |
|---|---|---|
| user | Link User | |
| food_date | Date | unique together with user |
| entries | Table LL Food Entry | |
| client_updated_at | Datetime | |

### LL Food Entry (child)
| Field | Type | Notes |
|---|---|---|
| client_id | Data | generated on the phone; unique within the day |
| meal | Select | breakfast / lunch / dinner / snacks |
| food_name | Data | |
| portion | Data | |
| qty | Float | portions |
| kcal_per_portion | Float | |
| protein_per_portion | Float | grams |
| source | Select | search / custom / describe / photo / suggestion / migrated |
| added_at | Datetime | UTC |

### LL Food Item (library: seed + per-user custom)
| Field | Type | Notes |
|---|---|---|
| food_name | Data | |
| portion | Data | |
| kcal | Float | per portion |
| protein_g | Float | per portion |
| is_seed | Check | seed items have no owner restriction |
| user | Link User | blank for seed items |

### LL Check In (one per user per program week)
| Field | Type | Notes |
|---|---|---|
| user | Link User | |
| checkin_date | Date | |
| program | Link LL Program | |
| week | Int | unique together with user and program |
| weight_kg | Float | |
| waist_cm / forearm_cm | Float | optional |
| sleep / energy | Int | 1–5, optional |
| photo | Attach Image | private file, optional |

### LL Weekly Review
| Field | Type | Notes |
|---|---|---|
| user | Link User | |
| program / week | Link / Int | unique together with user |
| period_start / period_end | Date | |
| stats | JSON | computed server-side (docs/05) |
| went_well / to_change | Small Text | from Claude |
| changes | Table LL Plan Change | |
| model | Data | |
| generated_at | Datetime | |

### LL Plan Change (child of LL Weekly Review)
| Field | Type | Notes |
|---|---|---|
| change_type | Select | weight / nutrition / schedule / other |
| title / why | Data / Small Text | |
| exercise | Link LL Exercise | weight changes only |
| delta_kg | Float | weight changes only |
| applies_from_week | Int | |
| accepted | Check | set by the apply endpoint |

The mobile add-weight function reads accepted weight changes for the coming week and adds `delta_kg` to its suggestion, with the reason "From your weekly review".

### LL Settings (Single)
| Field | Type | Default |
|---|---|---|
| anthropic_api_key | Password | |
| ai_enabled | Check | 1 |
| model_quick | Data | `claude-haiku-4-5-20251001` (describe estimates, suggestions) |
| model_vision | Data | `claude-sonnet-5` (photo estimates) |
| model_review | Data | `claude-sonnet-5` (weekly review, follow-ups) |
| daily_ai_calls_per_user | Int | 40 |
| max_image_mb | Float | 5 |

Model names are settings so they can change without a release. Check current names at docs.claude.com.

### LL AI Log
One row per Claude call: user, feature (estimate_text / estimate_photo / suggest / review / review_followup), model, input_tokens, output_tokens, latency_ms, ok, error_code, created. Used for the daily limit and to see cost. No prompt or photo content is stored.

## Server logic

- **Upserts.** `sessions.save` and `food.save_day` replace the child table with what the client sends, keyed by `(user, date)`. Reject a write whose `client_updated_at` is older than the stored one (return the stored record so the client can merge).
- **Food entry dedupe.** Within one day, entries with the same `client_id` collapse to one.
- **Check-ins** update `LL Profile.weight_kg`.
- **Automatic targets.** `LL Profile.validate` recalculates `kcal_target` and `protein_target_g` with the formulas in docs/01 whenever `weight_kg`, `height_cm` or `age` changed, skipping any target whose `_manual` flag is set. If a value changed, it writes `targets_note` (for example "Targets updated: 2,150 kcal, 130 g protein (height added)"). A check-in that changes the weight therefore updates the targets too. Unit-test: no height gives 2,200; adding height recalculates; a manual target is left alone.
- **Week calculation** uses the program's `start_date` and the profile's time zone.
- **Seeding.** `lift_log.setup.seed.run` loads `program-block2.json` into LL Program, `exercises.json` into LL Exercise, `foods-seed.json` into LL Food Item (`is_seed = 1`). Idempotent.
- **Install.** Creates the Lift Log User role and LL Settings defaults. Give the role to each person in desk (User > Roles); the app's sign-in refuses users without it, with the message "This account isn't set up for Lift Log yet."
- **Tests.** One test module per API file. Cover: permission isolation (user A cannot read B's sessions), upsert idempotency, stale-write rejection, client_id dedupe, AI daily limit.
