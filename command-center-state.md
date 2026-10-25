# EBS Command Center — state

Single-file HTML app on GitHub Pages, backed by Supabase. Eight tabs: Home, Money,
Worth, Social, Tasks, Sleep, Body, Cal.

- Repo: `EthanTrades/Ethans-Day-To-Day` → https://ethantrades.github.io/Ethans-Day-To-Day/
- Supabase project ref: `ldeuhjjpqlizhsbbgbul` (ap-southeast-2)
- Edge functions: `sync-windsor`, `sync-prices`, `log-sleep`
- Data changes are live immediately (the page reads Supabase). Code changes need
  `index.html` re-uploaded to the repo.

## Sleep

**Sleep Cycle CSV is the source of truth.** The Apple Health Shortcut was
abandoned: once Sleep Cycle was set to write into Apple Health, Health held both
an "In Bed" span and the Core/Deep/REM/Awake segments covering the same hours, so
the Shortcut's `Sum of Duration` counted the same minutes two or three times and
posted a 17-hour night. Apple Health was only ever re-reporting Sleep Cycle
anyway, so there was no independent second source to compare against.

To import a new export: run `claude/import_sleepcycle.py` over the CSV and apply
the SQL it prints. It upserts on `(night, source)`, so re-importing a longer
export refreshes the overlap and adds the new nights — no need to trim the file.

`log-sleep` (v3) now refuses anything over 16h asleep / 18h in bed, prefers the
bedtime→wake span over a summed duration, and caps asleep at in-bed time. The
Shortcut can keep firing harmlessly; nothing it sends will land.

A night is named after the **evening** you went to bed (noon cutoff), so a 12:25am
bedtime belongs to the night before. The importer and the endpoint agree on this.

## Workouts / Body tab

Eighth tab, **live since 24–25 Oct 2026** (Ethan uploaded it). A faceless muscle
figure (front and back) from his own drawings; muscles trained in the selected day or
week glow light neon purple, and the body grows with accumulated volume. Full
Trainerize history (110 workouts, 1 May – 16 Oct 2026) with watch calories, plus
weekly calories eaten. **From 25 Oct he logs workouts in the app instead of
Trainerize**: Dashboard / Today (live set ticking) / History views and a programme
editor; the tracker build is waiting for Ethan to upload index.html. **Workouts keep
exercises, set counts and times only — never reps or weights** (Ethan, 17 Oct).
Notes: `claude/workouts-figure.md`; patch: `claude/body-tab-patch.py`; pipeline:
`claude/workouts-trainerize-*`.

**Deploying:** I can't push to GitHub from the cloud workspace (no token; Chrome
uploads need a file on his computer; folder access was refused). Ethan drags files
into github.com/EthanTrades/Ethans-Day-To-Day/upload/main; I verify by sha256 against
raw.githubusercontent.com and the Pages URL.

## Conventions worth keeping

- Chart palette `#3987e5,#d95926,#199e70,#c98500` is validated (dataviz six checks)
  against surface `#0a0a0b`. Re-run the validator if the surface moves.
- Colour is reserved for data identity. Interface chrome is white on black.
- RLS: explicit `to anon` policies on app tables; `app_secrets` has RLS on with no
  policies, so only the service role can read it.
- The page reloads its data on `visibilitychange`, and a swipe down at the top of
  any tab re-reads Supabase. The header ⟳ button is the *heavy* refresh — it runs
  the Windsor and price syncs first.
- Supabase caps a response at 1000 rows; page with `sbAll()` for anything that
  can outgrow that.

## Open

- **Supabase Auth.** The repo is public, so the anon key is discoverable and grants
  read/write to the finance data. Raised several times, not yet actioned. ~30 min.
- **Windsor connectors** for Instagram, Meta Ads and TikTok Ads aren't connected, so
  ad spend reads $0.
- **`trainerize_import` anon policies** (select/insert/update) are still on from the
  24 Oct scrape; the migration that drops them was held for approval.

## Shipped

- 2026-09-29 — white neon theme (true black, white chrome), drawn EBS icon and
  apple-touch-icon, tabular figures, branded first paint, pull-to-refresh. The
  earlier blue variant is kept in the workspace as `index.blue.html`.
