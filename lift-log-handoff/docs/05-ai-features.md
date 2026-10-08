# 05. AI features

Three features, all called from the Frappe server with the official `anthropic` Python SDK and the Messages API. The phone never talks to Claude directly and never holds the key.

## Shared client (`lift_log/ai/client.py`)

- Reads the key and model names from `LL Settings`.
- Enforces the per-user daily limit by counting today's `LL AI Log` rows.
- Asks for JSON and parses tolerantly: whole reply, else the first fenced block, else the first `[`/`{` to the last `]`/`}`. On a parse failure, retry once with "Reply with only the JSON." appended; then return HTTP 502 "Couldn't read the estimate. Try again."
- Validates every number (finite, ≥ 0, kcal < 5000 per item, protein < 300 g) and drops items that fail.
- Logs model, token counts, latency and outcome to `LL AI Log`. Never logs prompt text or images.
- Timeouts: 30 s for estimates and suggestions, 90 s for reviews.
- Check model names and SDK usage against docs.claude.com before building; they are settings for this reason.

## 1. Food estimate (`ai.estimate`)

Model: `model_quick` for text only, `model_vision` when an image is present. Image sent as a base64 image content block; the phone resizes to ≤ 1568 px long edge and JPEG quality 0.8 first.

Prompt:
```
You estimate nutrition for food eaten in Kenya: Nairobi home cooking and local eateries.
Stews and vegetables are cooked with oil. Split the meal into separate foods.
For each food give a short name, the portion as eaten, and your estimate of kcal and grams
of protein for that whole portion.
{if image} The attached photo shows one meal. Judge portions from what you see (plate size,
piece counts). {if text} Extra details from the user: {text}
{if no image} What the user ate: {text}
Reply with only a JSON array, for example
[{"food_name":"Chapati","portion":"2 medium","kcal":360,"protein_g":8}]
```
Text is cut to 1,500 characters.

## 2. Meal suggestions (`ai.suggest`)

Model: `model_quick`. The server builds the context itself:

- targets, eaten so far (meal: food x qty list), remaining kcal and protein
- the day's routine name and session time, or "rest day"
- the user's 15 most frequent foods in the last 30 days (fallback: chapati, rice, beef stew, chicken, lentils, beans, eggs)
- goal and body weight

Prompt:
```
You suggest meals for a {weight} kg man in Nairobi, Kenya, who lifts weights {n} days a week
and wants to {goal}. Suggest 3 different options for his {meal} (or the rest of today)
using cheap, easy-to-find Kenyan foods and home cooking.
Targets today: {kcal} kcal and {protein} g protein. Remaining: {kcal_left} kcal and
{protein_left} g protein. Eaten so far: {eaten}. Training today: {training}.
Foods he often eats: {usual}.
Close the protein gap without going over the remaining calories. If calories are nearly
used up, suggest lean, high-protein, low-calorie options. Keep portions realistic.
Each option lists its foods with portion, kcal and grams of protein for that portion, and
one short sentence on why it fits.
Reply with only a JSON array:
[{"title":"...","why":"...","items":[{"food_name":"...","portion":"...","kcal":0,"protein_g":0}]}]
```
Keep at most 4 options, each with at least one valid item.

## 3. Weekly review (`ai.weekly_review`)

Model: `model_review`. Runs on demand; one stored review per user per week (regenerate replaces it).

### Input the server assembles

Computed in Python first (these are also the four stat tiles, so they never depend on the model):

```json
{
  "week": 1, "phase": "build", "period": ["2026-10-05", "2026-10-11"],
  "targets": { "kcal": 2200, "protein_g": 130 },
  "sessions": {
    "planned": 5, "done": 4, "missed": ["Upper A (Mon)"],
    "items": [
      { "date": "2026-10-06", "routine": "Lower + Walk", "note": "...",
        "exercises": [ { "name": "Barbell back squat", "target": "3 x 6-8",
          "sets": [ {"kg": 40, "reps": 8, "effort": "easy"} ],
          "rule_call": { "call": "hold", "kg": 40 } } ] }
    ]
  },
  "effort": { "easy": 7, "right": 4, "hard": 1, "unrated": 0 },
  "food": { "days_logged": 6, "avg_kcal": 1890, "avg_protein_g": 98, "protein_days_hit": 1,
            "by_day": [ { "date": "...", "kcal": 0, "protein_g": 0 } ] },
  "checkin": { "weight_kg": 72.0, "weight_change_kg": null, "waist_cm": null, "forearm_cm": null, "sleep": 3, "energy": 4 },
  "previous_review_changes": [ { "title": "...", "accepted": true } ],
  "goals": "Lose fat and get stronger; keep thighs from growing; grow forearms."
}
```

`rule_call` is the add-weight rule's output for each exercise, computed by porting the same rule to Python (keep both implementations covered by the same test table). Claude can agree with it or argue for a change.

### Prompt

```
You are reviewing one week of a lifting and nutrition log for the person described in the data.
Be direct and specific; cite numbers from the data. No generic advice.
Effort ratings: easy = 3+ reps left, right = 1-2 left, hard = nothing left.
Write:
- went_well: 1-3 sentences.
- to_change: 1-3 sentences.
- changes: up to 4 concrete changes for next week. Allowed types:
  weight (exercise name exactly as in the data, delta_kg in steps of the exercise increment,
  at most one step per exercise per week), nutrition, schedule, other.
  Only propose a weight increase when reps and effort both support it; propose a decrease
  only after two hard weeks. Respect the goals (no added leg volume).
Reply with only JSON:
{"went_well":"...","to_change":"...","changes":[{"change_type":"weight","title":"Squat: +2.5 kg",
"why":"...","exercise":"Barbell back squat","delta_kg":2.5}]}

DATA:
{json}
```

The server validates changes: weight changes must name a known exercise, `delta_kg` must be ±1 increment, and there are at most 4 changes. `applies_from_week` = week + 1.

### Follow-up questions (`ai.review_followup`)

Sends the same DATA, the stored review, and the user's question (≤ 500 characters) to `model_review`, asking for a plain-text answer under 120 words. Not stored.

## Cost guard

- Daily limit per user (default 40 calls) across all features.
- Photos capped at `max_image_mb` after resizing.
- The suggestion card is fetched at most once per 3 hours automatically (docs/01); "Other ideas" is manual.
- `LL AI Log` gives a monthly token total per feature; add a simple desk report "AI usage by month".
