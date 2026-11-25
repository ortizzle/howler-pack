# The Howler Pack: notes for Claude

The Ortiz family's fitness app: Chris and Kat (parents, "Praetorians"; the internal role id stays `proctor`) and Sedona (13) and River (10) (kids, "Pups"). The family follows the family-app standards skill (`family-app-standards`); read it before changing anything. This file covers what is specific to this repo.

## Deploying = pushing to `main`

GitHub Pages serves `main` at https://ortizzle.github.io/howler-pack/. **Every push to main is live for the whole family.**

- **Chris's standing instruction (Nov 25, 2026): always publish** once a change is finished and every test suite passes. Don't wait for an OK.
  - **Exception:** if his message asks questions, answer them first and publish only after he responds.
  - Still mention alternatives worth considering when reporting what was published.
- Do the work on a branch, test it, then fast-forward `main` to it.
- After a deploy, remind him that Pages takes about a minute. The service worker fetches the page network-first, so closing and reopening the app picks up changes.

## Shape of the code

- **`index.html` is the whole app**: inline CSS and JS, no build step, no framework. Sections are marked with `// ────` banners: helpers, store, dates, sync, data, rules, scoring, race, primus, goals, awards, AI, UI, logging, reader, screens, settings, boot.
- **`sw.js`**:
  - offline shell
  - the share target: Peloton app → Share → Howler Pack, posted to `./share-target` and handed over through the `pack-share` cache
  - Periodic Background Sync checks (cheers, evening nudge)
  - Bump `SHELL` when the cached file list changes.
- **`ortiz-drive.js`** is shared by every Ortiz app. Copy it from another app; don't fork it.
- **DOM:** build with `h()` / `s()`, attach events with `addEventListener`. No `innerHTML` with data, no inline `onclick`, no `alert`/`confirm`/`prompt` (use `openModal` / `confirmModal` / `toast`).
- **Dates:** always `AZ.today()` (America/Phoenix) and the key helpers (`addDays`, `weekKey`, …). Never `toISOString` for dates.
- **Primary target:** Pixel phones in Chrome. Keep 44px tap targets and test at Pixel 7 size (412 px wide).

## Data and privacy rules (don't break these)

- **Sync:**
  - One private Gist file, `howler-pack.json`: `{ records: { id: {...} } }`.
  - Every record has `type` and `updatedAt`. Change records only through `put` / `putMany` / `del`; they stamp `updatedAt` monotonically.
  - Deletes are tombstones. Sync is fetch → merge (newest wins) → write.
- **Weigh-ins never leave the phone** (`pack_weighins` in localStorage):
  - Only the derived Merit `award` records sync, plus a `pct` record when that parent turns sharing on.
  - Never put pounds in a synced record, a label or an AI prompt about someone else.
- **Kids:** no weight, food or body talk anywhere on a Pup's screen, card or coach prompt.
- **Secrets:** the Gist token (`pack_gist_token`) and Claude key (`pack_anthropic_key`, falling back to Fuel's `fuel_anthropic_key`) live only in localStorage. Never commit a key or token.
- **The join link** (`#join=` + base64 of the gist id and token) carries the token; it is only shared within the family.
- **Fuel** (https://ortizzle.github.io/fuel/, repo `ortizzle/fuel`) reads this app's data read-only through the shared origin's `pack_gist_id` / `pack_gist_token` / `pack_me` keys. Don't rename those keys without updating Fuel.

## Game rules (where to change them)

- **Rules and Merit values:** `RULES`. **One Merit scale for everyone** (Chris chose this, Nov 25, 2026):
  - Showing up pays the same: Active Day +15, Standard +50 (Praetorians 4 days, Pups 3), chain, together, cheers, goals, Primus, race.
  - Effort: Praetorians use MET × min ÷ 10; Pups use 1 point per 2 minutes. Both cap at 40 a day, so a 30-min kid workout ≈ a 30-min adult one at average intensity.
  - **Pups' growth points** (`RULES.growth`) balance the parents' weight-progress points: best week yet, longest outing yet, taekwondo stripe/belt and personal bests. `milestone` records are added via `openMilestone`.
  - Kid bonus points may come later; Chris expects the kids to push harder instead.
- **The Codex** (`openCodex`) shows every rule and number. It's generated from `RULES` / `RANKS`, so it can't drift. Linked from the Pack tab, Settings, and "How points work ›" under The Pack.
- **Activities and MET values:** `ACTIVITIES`.
- **Ranks:** `RANKS`, **one ladder for everyone, built all year** from Dec 1 and never reset (`score().year` = this season plus earlier seasons; Practice Week stands alone).
  - It's progressively harder, like the original Howlers: 0 / 600 / 2,000 / 4,500 / 8,000 / 12,500 / 18,000. Peerless by year's end takes beating the Standard (~300 Merit a week).
  - Seasons keep things fresh with races, monthly and season goals, and the Hall of Honors (the color at each season's end).
- **Pack cards** are sorted by year Merit and show it with the place ("Merit · =1st" on ties), plus progress to the next rank (`nextRankStat`, e.g. "18 to Gray" with a bar in that rank's color). The % to Peerless is inside the expanded card.
- **Cheers** go through `sendCheer` (5 s Undo). Your own cheers can be taken back from the Cheer Wall; the +2 goes with them.
- **Practice Week:** `PRACTICE` (Nov 23–30, 2026).
  - It's deliberately not in `SEASONS`. `seasonFor` / `currentSeason` return it for those dates.
  - Points, ranks, Primus and a 15-step warm-up race work, then everything restarts Dec 1 with The Passage.
  - `practice()` is true during it. Goals point at December (`goalDay`).
  - Practice Primus crowns don't count in `primusCount`.
  - Practice workouts stay in the Annals.
- **Seasons, races, lore:** `SEASONS`.
  - Seasons 2–4 have `loreDraft: true`. The family is still shaping that lore, so adding or rewriting stations is welcome.
  - Keep lore kid-friendly, accurate to the books, and short. "Real Mars/Moon fact" stations must be true.
  - Race goals assume about 2.5 Pack Steps a day when everyone meets their Standard. Holiday checkpoints (`by`) sit at or just under that pace.
- **Peloton class types** (Chris, Nov 25, 2026): Tread and Bike carry `types` in `ACTIVITIES` (Tread: Run, Walk, Hike, Bootcamp; Bike: Ride, Low Impact, Bootcamp), stored as `detail.cls`.
  - The type sets the effort baseline when no numbers are entered; speed/incline (Tread) or watts (Bike) still replace it. Walk and Hike use the ACSM walking equation up to 5 mph, Run the running one from 4 mph (`acsmMet(mph, grade, gait)`).
  - Tread speed comes from distance ÷ time when only miles are entered. A Hike with no incline uses whichever is higher, the equation or the class baseline.
  - Bootcamps count as half machine, half floor (MET 5). They ignore kJ, because total output covers only the machine half.
  - The log sheet shows "Class type" chips under "What" (kids too), defaulting to your last type for that machine (`lastCls`). Usuals can carry `cls`.
  - Older Tread entries without a type read it from their speed (`clsOf`). The CSV import sets the type from the discipline and title. Tread/Bike Bootcamp rows keep their old `_bootcamp` id suffix, so re-imports still match.
  - A changed class type on an imported record is an override, with ↺ Peloton.
- **Adult workout stats:** `STAT_FIELDS` / `STATS_FOR` / `STATS_ALL`.
  - Speed, incline, watts (or kJ → watts) and distance refine MET.
  - Heart rate and device calories are stored and shown but never scored; keep it that way for fairness across bodies.
  - Kids never see the stats section.
- **Activity groups and "With anyone?"** (Chris, Nov 25, 2026): two groups in `ACTIVITIES`.
  - **Peloton** (Tread, Bike, Strength, Yoga, Stretch, Meditation, Bootcamp) is solo: no who question (`isSolo`).
  - **Outside & Classes** (everything else) asks **"With anyone?"**: starts as Just me; tap names to add people, or one-tap **↻ Same as last time** (`lastCrew`, from your last group entry of that activity). Nothing is pre-selected.
  - One entry becomes one `workout` per person, sharing a `group` id. Non-Peloton quick usuals open the sheet instead of logging in one tap.
  - People something was logged for see "📬 Added for you" with **I wasn't there** (removes only their copy). The service worker also notifies them.
  - **Park was folded into Active Play** (Chris's call). `park` stays in `ACTIVITIES` with `hidden: true` so old entries still read; don't show it in pickers.
  - **Meditation** (`mind: true`) earns 1 point per 10 min and is never movement: `moveMin()` returns 0 for it, so it doesn't count toward Active Days, week minutes, minutes goals or growth. **Stretch** is MET 2.3 (2024 Compendium). **Pickleball** is 4.1–6.5.
  - **Weekly classes** (`member.weekly: [{act, days, min}]`, Settings → My weekly class): `classPrompts()` checks in after noon on the day, or for up to two days after. Kat's Sunday pickleball is pre-set (`migrateWeekly`).
  - `migrateUsuals()` adds a Walk usual once per parent; up to 6 usuals.
- **Where & how far** (Chris, Nov 25, 2026): every log sheet, kids' included, shows a route/place box (`title`) and, for distance activities, miles (`detail.miles`) right after "When". They're visible, never required, and recent places (`recentPlaces`) are one-tap chips so they're nearly always answered. These fields are not in "More stats".
- **Editing:** tapping an Annals entry calls `openLog({ edit: record })`, which updates that record in place.
  - For a group activity, **Apply to everyone who joined** (on by default) copies the shared facts to the others: activity, minutes, date, dogs, title, distance, elevation.
  - Heart rate, calories, felt and notes stay personal.
- **Peloton CSV import** (`pelotonRows`):
  - Ids come from the row timestamp (`w_pel_…`).
  - A hand-logged entry on the same day, same activity and about the same length (±25% or 10 min) is **filled in** (`pelotonId` set) instead of duplicated.
  - **Peloton is final by default, with overrides.** Imported records keep `peloton: { min, detail }` (what Peloton said) and `overrides` (fields the person changed; `pelotonDiffs()` recomputes it on save).
    - When a hand-logged entry disagrees, the import preview shows both and defaults to "Use Peloton"; "Keep mine" stores the person's numbers as overrides.
    - Editing a Peloton record marks changed fields "edited" with a per-field ↺ restore and a "Restore all Peloton numbers" button.
    - This matters most for outdoor workouts, whose GPS numbers often need fixing.
  - Re-importing adds nothing and never undoes an override.
- **Redraws from a field's `change` event go through `redraw()`** (setTimeout 0). `change` fires on blur, and rebuilding the sheet mid-blur throws `removeChild` errors.
- **Weekly Primus:** `primusScores`. It goes to whoever beats their own target by the most, so kids and parents compete fairly.
  - **Ties are never broken alphabetically.** A tie at the top (`primusLeaders`) shows no crown and an "anyone can take it" note. If the week ends tied, each tied person gets a shared crown (`aw_primus_<week>_<id>`, "(shared)") and the +25.
- **Coaches:** `COACHES` (Dancer, Mickey, Matteo, as in the original Howlers). The model is `claude-opus-5-5` with effort `low`, browser-direct, with server-side fallbacks.

## Testing

- **Run it locally:** `python3 -m http.server 4410` in this folder.
- **Sample data:** Settings → *Preview with sample data* (or the welcome screen) loads a mid-December family in memory only. It's the quickest way to see every screen.
- **Before any deploy,** run the family-app smoke test (in the skill), using Playwright with a stubbed `api.github.com`:
  - fresh start
  - log, reload, delete, reload (still deleted after sync)
  - two-phone merge through a join link
  - bad token
  - Arizona date after 5pm
  - weigh-in pounds never in the Gist
  - no-key AI paths
  - kid view has no weight words
  - 44px tap targets
  - no JS errors
