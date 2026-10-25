# Workouts tab — the muscle figure

Status (25 Oct): **Body tab is live** — Ethan uploaded the purple build himself
(index.html sha256 181dc2c68284…, body-front.png b9856bb778c3…, body-back.png
f1e6621396a2…; verified on raw.githubusercontent and GitHub Pages). **The tracker
build (index.html 0202af616cf0…) is ready and only needs index.html uploaded**;
the images are unchanged. I can't upload to GitHub from the cloud workspace: Chrome
file uploads need a file on his computer, and folder access / a Supabase storage
hand-off were both refused by the auto-mode permission check, so Ethan uploads.

## In-app tracking (from 25 Oct — replaces Trainerize)

Ethan (25 Oct): "for the rest of the year … instead of using trainerize ill switch to
this app so id like body to have a dashboard inside of itself and then be able to
track the current day, and a seperate spot for the old data". His trainer will send
the next cut phase soon; he'll add it himself and may change programmes through the
year. No preference on logging style, calories or dashboard → recommended defaults.

Body tab now has three views (segmented control):
- **Dashboard**: today card (Start / Resume / Done ✓ / Rest day + what's next), this
  week's figure lit purple with a Mon–Sun strip (✓ done, dashed = missed), tiles
  (sessions x/planned, gym time, calories burned, streak = weeks in a row hitting the
  plan; the current week only breaks it once it can't be hit), muscle balance (sets a
  week over the last 4 weeks, least-trained three named), programme card (Edit / New).
- **Today**: pick the session (defaults to today's weekday), figure previews what it
  works, Start workout → live tracker: timer, sets ticked/planned, one 46px button per
  set (+ for an extra set), cardio "Mark done", add an exercise mid-workout, tag
  muscles for an unknown exercise, Finish sheet (date, start time, minutes, calories,
  avg HR, note), Discard. "Log one I've already done" = same tracker without a timer
  for a past date. Done sessions can be reopened with Edit.
- **History**: the old phase/week browser, now including app sessions (finished ones).
- Programme editor (from the programme card): name, phase (or + New phase…, inserted
  into workout_phases), start date, days (name, type, weekday), exercises (name, sets,
  rest/notes, strength/cardio; reorder, edit, remove). A new programme ends the running
  one the day before it starts; one with a future start takes over on that date.

Data:
- Programmes live in `app_settings.workout_programs` (jsonb array) — no new tables, so
  no new RLS policies were needed. Seeded with the current programme "Sessions 1–5"
  (lean bulk, from 12 Oct; Mon S1 Pull, Tue S2 Push, Wed S3 Legs, Thu S4 Pull,
  Fri S5 Push) from the 12–16 Oct Trainerize sessions.
- App sessions: `workout_sessions` with source='app', started_ts/finished_ts
  (open = finished_ts null), program_id, program_day; split gets "-2" if the same split
  is logged twice in a day (unique key is date+source+split). Exercises are copied
  from the programme day at Start, prescription "N sets · rest/notes".
- Set ticks are optimistic and queued in localStorage (`bd-pending-sets`), flushed in
  order, retried every 8 s and on `online`; Finish waits for the queue. Database
  refusals (2x/4x/PGRST codes) are dropped, network errors retried.
- Effective sets are now computed in the browser from sessions × `exercise_muscles`
  (the `workout_muscle_sets` view is no longer read), so the figure lights as sets are
  ticked. New exercise muscle shares are written from the app (1 main / 0.5 helper).

Tested end to end with Playwright against live reads and an in-memory fake for writes
(`cc/test/fake.js`, `t7.js`, `t8.js`): start, tick/untick, extra set, offline tick and
recovery, add unknown exercise with muscles, finish with calories, history, rest-day
"train anyway", discard, log a past workout without timer, edit programme, new
programme with a new phase. No page errors.

**Rule from Ethan (17 Oct): keep exercises and times only.** No reps, no weights
— for every workout. What is kept: exercise name, the prescription text (set
count, rest time, coaching note), how many sets were done (as empty
`workout_sets` rows, so the figure still knows how much each muscle worked),
cardio minutes and heart-rate band, session start time and duration, and (from
24 Oct) the session's calories and heart rate from his watch. `workout_sets`
holds 0 rows with reps or weight.

## Trainerize history (loaded 24 Oct)

Ethan asked (24 Oct): "open trainerize on my browser and scroll back to may and
document all workouts with there calories paired with certain workouts", then
"organise everything when done".

Scraped read-only in his Chrome from Progress → Review by workout
(`/app/client/29049382/progress/workout?startAt=…&endAt=…&page=N`; ranges over
~3 months come back empty, so May–Jul and Aug–Oct were done separately). Each
workout's modal gives name, start time, duration, calories, active calories,
avg/max HR and each exercise with its prescription and the count of logged
"Set N" lines. Weights were stripped in the page before anything left it. Raw
rows sit in `trainerize_import` (110 workouts, 1 May – 16 Oct 2026).

Pipeline (project docs): `claude/workouts-trainerize-common.py` (name cleanup,
key), `claude/workouts-trainerize-organise.py` (→ sessions.json),
`claude/workouts-muscles-new.py` (shares for 53 new exercises),
`claude/workouts-trainerize-load.py` (PostgREST upserts with the anon key, no
DELETE), `claude/nutrition-weeks.py` (calories eaten + goals).

Result: 110 sessions, 709 exercises, 2,494 set rows, every strength exercise
mapped to muscles. The ten hand-entered sessions (18–22 May, 8–12 Jun) kept
their exercises; only time, duration, calories, HR, phase and note were
refreshed from Trainerize.

Cleanup rules applied:
- Duration ≥ 5h = watch left running (1 Jun 14h 22m, 3 Jul 14h 11m): duration,
  calories and HR dropped.
- Duration < 20 min = logged after the session: no start time or duration (the
  "time" is when he ticked it off); calories kept if the watch had them.
- All strength sets unticked (1 Jun, 18 Jun, 9 Jul, 15 Jul, 14 Aug, 5–6 Oct,
  14 Oct): planned set counts used, note says so. Superset followers ("Rest for
  60 sec") take the superset's count; prescription rewritten to "Superset · N
  sets · Rest 60 sec".
- Single unticked exercise in a logged session = skipped (hack squat 10 & 17 Jun,
  neutral grip row 24 Jul).
- Names: "CHEST PRESS MACHINE TUTORIAL 💪…" → Chest Press Machine; "How To"
  prefixes/suffixes and the Hunter Labrada video title removed; ALL CAPS fixed.
- Oct "Session N" split by contents: 1 Pull, 2 Push, 3 Legs, 4 Pull, 5 Push
  (title "Session 4 · Pull").

Calories: 52 of 110 sessions have watch calories. `nutrition_weeks` holds the
weekly average eaten (Trainerize Nutrition Center, synced from MyFitnessPal),
13 Apr – 19 Oct. `nutrition_goals` holds the goals from Trainerize emails:
11 Apr 1458, 20 May 1110, 3 Jun 1510 (Matty: start a lean gain), 1 Jul 1230,
15 Jul 1475, 29 Jul 1765. The current goal is 2445 (175P/335C/45F) but no email
dates it, so it isn't in the table and the tab doesn't show goals.

## Phases — partly Ethan's, partly a guess to confirm

Grouped by training template (the programme changes with the phase):

| Template | Dates | Phase | Basis |
|---|---|---|---|
| 1. Push / 2. Pull / 3. Legs / 4. Upper / 5. Lower | 1–28 May | Lean cut | Ethan labelled 18–22 May |
| Push 1 / Pull 1 / Legs / Push 2 / Pull 2 | 1 Jun – 3 Jul | Heavy cut | Ethan labelled 8–12 Jun |
| Chest / Back / Shoulders / Legs / Upper | 6 Jul – 14 Aug | Lean cut | guess: his order puts lean cut after heavy cut; eating ~1,400–1,670 |
| Push / Pull / Legs / Arms | 17 Aug – 7 Oct | Bulk | guess: eating rose 1,730 → 2,300 |
| Sessions 1–5 | 8 Oct → | Lean bulk | guess: last in his order |

Ethan's labels (May = lean cut, June = heavy cut) run opposite to his stated
phase order (starting, heavy cut, lean cut, bulk, lean bulk). Starting has no
sessions (April wasn't in scope). To move a block: one UPDATE on
`workout_sessions.phase_id` by date range.

## The tab

`claude/body-tab-patch.py index.html` adds it (checks the 14 Oct live file's
sha256, applies an embedded diff, checks the result's sha256). Upload
`body-front.png` and `body-back.png` next to index.html in the same commit.

- Phase track (1–5), week stepper, day chips (Week / Mon Push / Tue Pull ...;
  "Session N ·" dropped on chips). Same-day sessions ordered by start time.
- Front + back figures side by side. The body stays white; trained muscles glow
  **light neon purple** (Ethan, 24 Oct: "leave the image white as usual but
  highlight in a light neon purple color for each day") — NEON = vec3(0.86,0.62,1.0)
  ≈ #DB9EFF, used for strokes, fill tint and bloom. Lit = the selected day (full at 4.5
  effective sets) or week (full at 12). Size = 1 - e^(-cumulative sets / 600)
  (was 300; with six months loaded, 300 had the upper body at ~0.8 already),
  counting every session up to the end of the selected week.
- Tiles: Sessions, Burned (watch kcal; day view adds active and max HR; week
  view says "n of m sessions tracked"), Sets, Time, Eaten (that week's average
  kcal/day and meals logged), Cardio.
- Effective sets by muscle: ranked bars; untrained muscles stay listed.
- Session detail: date + title, then start · duration · kcal · avg HR on a second
  line, the session note (cleanup reasons, heavy back-off), then each exercise
  with prescription, muscles, and a "skipped" / "N sets done" chip.
- `workout_muscle_sets` is read with `sbAll()` paging (Supabase caps responses at
  1000 rows; it's 603 rows now).

## Back view

From the back-view template Ethan sent on 16 Oct (768x1376, pencil on paper,
with the artist's construction lines). Pipeline in `claude/figure-back-*.py`,
run in order: guides -> silhouette -> segment -> bake. Rebuilds byte-identical.

- **guides**: Hough finds the long straight guides; only faint pixels
  (g >= 150) are inpainted, so contours under guides stay.
- **silhouette**: flood the paper in from the border; arm-torso gaps seeded.
- **segment**: watershed as on the front. Groups: traps (incl. shoulder blades),
  rear delts, lats, triceps, forearms, lower back, glutes, hamstrings, an outer-
  thigh quads sliver, calves (to the ankle).
- **bake**: affine map matches the front figure's height and top offset and
  widens it 12% about the centre line.

**Rear delts are their own group** (`reardelts`). The front figure's shoulder cap
is front/side delt, the back figure's is rear delt.

## Data (Supabase)

Tables: `workout_phases`, `workout_sessions` (unique on date+source+split; kcal,
active_kcal, avg_hr, max_hr, started_at, duration_min, trainerize_name),
`workout_exercises` (generated `key` strips a trailing " - pf" for muscle
lookup), `workout_sets` (one row per set done; reps and weight empty),
`exercise_muscles` (key -> muscle, share 1 / .75 / .5 / .25), `nutrition_weeks`,
`nutrition_goals` (select-only for anon), `trainerize_import` (raw scrape).
View `workout_muscle_sets` = effective sets per session per group.

**Connector gotcha:** the Supabase MCP holds any statement containing `DELETE`
for human approval and times out at 180s. On 24 Oct a `DROP POLICY` migration
was held the same way. Loaders upsert on unique keys instead. Bulk loads go
through PostgREST from the sandbox (anon key) to keep big payloads out of SQL.

**Open:** `trainerize_import` still has anon select/insert/update policies
(`timp_sel`, `timp_ins`, `timp_upd`) from the scrape. Drop them once Ethan
approves the held migration (or in the dashboard).

## What it is

Ethan's own reference drawings (front 14 Oct, back 16 Oct), inverted to glow
white on black, rendered by one WebGL2 fragment shader per view. Lit groups go
to full brightness with a cool bloom; accumulated volume warps the image outward
around each muscle belly (gaussian push, one blob per belly).

## How the front is built (reproducible — verified byte-identical)

Needs the reference saved as `model.png` (1152×2048, grey #7f7f7f background).
1. `figure-silhouette.py` → `body.npy`
2. `figure-segment.py` → `mlab.npy`, `mnames.json` (watershed seeded at each
   muscle belly; left seeds mirrored about x=579)
3. `figure-bake.py` → `body-front.png` (576×1024; R = art, G = silhouette,
   B = group index×20) and `body-front.json` (groups + warp blobs)
4. `figure-body.js` — standalone renderer (the tab inlines its own copy).

## Ideas parked

- lit decaying over ~72h since a group was last trained
- 3D body (possible via spatial weight maps; parked because image→3D from a line
  drawing tends to look mushy)
