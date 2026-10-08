# 02. Design spec

The 15 mockups in `design/screens/` are the visual reference (412 x 915, Android). They are HTML with inline styles: read the styles for exact values. They do not render on their own (they rely on the design canvas runtime); the user can share screenshots from the canvas if needed. This file lists the system they share so it can be built once as React Native components.

## Colors (light theme)

| Token | Hex | Use |
|---|---|---|
| `bg` | `#EEF1F5` | Screen background |
| `surface` | `#FFFFFF` | Cards, sheets, nav bar |
| `surface2` | `#F5F7FA` | Inputs, table headers, done-set fields |
| `track` | `#E6EAF0` | Meter tracks, row dividers |
| `line` | `#D6DCE5` | Card borders |
| `lineStrong` | `#B9C2CF` | Secondary button borders, dashed add buttons |
| `ink` | `#16202C` | Primary text; also the dark surface (rest timer, snackbar, sign-in) |
| `ink2` | `#4A5667` | Secondary text |
| `ink3` | `#5F6B7E` | Labels and captions (passes 4.5:1 on white) |
| `accent` | `#2350A8` | Primary buttons, active tab, calories meter, selection |
| `accentSoft` | `#E3EBFA` | Active tab pill, suggestion card, "start" chips |
| `up` | `#1D7A4A` | Protein meter, done ticks, "add weight" text, Easy effort text |
| `upSoft` | `#E2F3EA` | Add-weight chip and Easy effort background |
| `hold` | `#8A5A00` | "Stay" text, Hard effort text, warnings |
| `holdSoft` | `#FBF0D9` | Stay chip, Hard effort background, hint banners |
| `danger` | `#A1271E` | Sign out |
| `scrim` | `#5A6472` | Behind bottom sheets |

Phase colors on the 12-week bar (Train): Build `accent`, Home deload `#B9C2CF`, Strength `#1D6B5F`, Peak `#8A5A00`, Test `ink`.

Dark theme: not in version 1. Keep every color in one theme file so it can be added.

## Type

| Role | Font | Size / line height |
|---|---|---|
| Screen title | Barlow Condensed 700 | 34 / 34 (26 on pushed screens with a back button) |
| Card title | Barlow Condensed 700 | 22–28 |
| Big numbers (meters, stats, weight) | Barlow Condensed 700 | 28–34; check-in weight 60 |
| Body | IBM Plex Sans 400/500 | 15 / 22 |
| Secondary | IBM Plex Sans 400 | 13 / 19 |
| Labels (eyebrows) | IBM Plex Sans 600, uppercase, letter-spacing 0.08em | 12 |
| Numbers in tables and inputs | IBM Plex Mono 500 | 13–18 |

## Spacing and shape

- Screen side padding 20. Card padding 16–18. Gap between cards 12–14.
- Radii: cards 20, inner cards 14–16, inputs and small buttons 12, chips 8, sheets 28 (top corners), phone frame 28.
- Primary button: height 52, radius 16, accent fill, white 600 text 16.
- Secondary button: height 44–52, radius 12–16, white fill, `lineStrong` border.
- Minimum tap target 44 x 44.

## Components to build once

| Component | Notes |
|---|---|
| `BottomNav` | 80 tall, 4 items, active item has a 64 x 32 `accentSoft` pill behind the icon (Material 3 style). Icons: home, dumbbell, utensils, bar chart. |
| `ScreenHeader` | Eyebrow label + title; optional right action. Pushed variant has a 48 x 48 back button. |
| `WeekStrip` | 7 day cells, selected cell outlined in accent, two 7 px dots (training green, food accent; hollow grey when empty). |
| `Card` | `surface`, 1 px `line` border, radius 20. |
| `Meter` | Label, big number with "/ target unit", 8 px track, caption "X of Y left". Variants: calories (accent, amber over target), protein (green). |
| `CallChip` | up / hold / start / down, colors above, radius 8, 13 px 600. |
| `SetRow` | Set number, kg field, reps field (both 48 tall, mono 18), 48 x 48 tick button. Done state: grey fields, green tick. Active state: accent border on fields. |
| `EffortPicker` | Three equal buttons under a done set: Easy (green), Right (accent), Hard (amber). Unselected: white with `line` border. 36 tall. |
| `RestTimer` | Dark (`ink`) card, mono 30 countdown, "+30 s" outline button and white "Skip" button. |
| `MealSection` | Title (Barlow 20) + total, entry rows (name, qty, kcal mono grey, protein mono green), "Nothing yet". |
| `ExtendedFab` | 56 tall, radius 16, accent, camera icon + "Add food", shadow. Sits above the snackbar. |
| `Snackbar` | `ink` background, radius 12, white text, accent-light action "Undo" (`#9DBBF2`). |
| `BottomSheet` | Grab handle, title + close, segmented tabs (`bg` track, white selected segment). |
| `Segmented` | 4 options max, 40 tall segments. |
| `Toggle` | 52 x 32 track (accent on, `#C9D0DA` off), 24 px white knob. |
| `RatingRow` | Five 44 tall buttons labelled 1–5, selected accent. |
| `StatTile` | Small card with Barlow number and 11–13 px caption. |
| `ProteinBars` | 7 bars, dashed target line at the target value, values above bars. |

## Screen notes

Only what isn't obvious from the mockups:

- **Today:** the food card is one tappable area that opens the Food tab. The hint banner uses `holdSoft`/`hold` when protein is behind pace (less than 40% of target by 14:00, or less than 70% by 19:00), otherwise it is hidden.
- **Workout:** keep the screen awake while a workout is open (`expo-keep-awake`). The effort legend sits under the set rows. "Up next" lists the remaining exercises; tapping one jumps to it.
- **Workout summary:** reached automatically on finish. "Done" returns to Today.
- **Food:** the snackbar appears above the bottom nav and pushes the FAB up while visible.
- **Add food sheet:** opens at about 85% height. The photo tab shows the taken photo as a 96 x 96 thumbnail beside the result summary; while estimating, show "Estimating…" with a cancel action.
- **Check-in:** if a check-in exists for the week, the screen opens in edit mode with its values.
- **Weekly review:** while generating, show the four stat tiles (computed locally) and a "Claude is reviewing your week…" placeholder for the text and changes.
- **Sign in:** server address field defaults to `https://`. Show the server's error message on failure (wrong password, site unreachable).

## Copy rules

Plain, short, second person. Buttons say exactly what happens ("Add 3 foods · 610 kcal · 34 g", "Apply 2 changes to week 2"). No exclamation marks, no emoji.
