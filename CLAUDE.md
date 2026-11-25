# The Howler Pack: notes for Claude

The Ortiz family's fitness app: Chris and Kat (parents, "Proctors") and Sedona (13) and River (10) (kids, "Pups"). The family follows the family-app standards skill (`family-app-standards`); read it before changing anything. This file covers what is specific to this repo.

## Deploying = pushing to `main`

GitHub Pages serves `main` at https://ortizzle.github.io/howler-pack/. **Every push to main is live for the whole family.**

- Suggest alternatives, then get Chris's explicit OK before pushing.
- Work on a branch until then.
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

- **Rules and Merit values:** `RULES`.
- **Activities and MET values:** `ACTIVITIES`.
- **Ranks:** `RANK_BASE`, per 13-week season, scaled to season length; they reset each season.
- **Seasons, races, lore:** `SEASONS`.
  - Seasons 2–4 have `loreDraft: true`. The family is still shaping that lore, so adding or rewriting stations is welcome.
  - Keep lore kid-friendly, accurate to the books, and short. "Real Mars/Moon fact" stations must be true.
  - Race goals assume about 2.5 Pack Steps a day when everyone meets their Standard. Holiday checkpoints (`by`) sit at or just under that pace.
- **Weekly Primus:** `primusScores`. It goes to whoever beats their own target by the most, so kids and parents compete fairly.
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
