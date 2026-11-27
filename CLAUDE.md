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
  - **Goal weight** (Chris, Nov 26, 2026): set in pounds in Settings → My weight goal and stored only on the phone (`pack_goal_lb`, `goalLb()`).
    - The member record syncs just the % from the season's starting weight (`goalPctFor` / `syncGoalPct`). The % stepper stays as a fallback when no pounds goal is set.
    - The Pack is the one place to set it: Fuel reads `pack_goal_lb` for its protein target and shows its own box as "set in the Howler Pack".
  - Never put pounds in a synced record, a label or an AI prompt about someone else.
- **Kids:** no weight, food or body talk anywhere on a Pup's screen, card or coach prompt.
- **Secrets:** the Gist token (`pack_gist_token`) and Claude key (`pack_anthropic_key`, falling back to Fuel's `fuel_anthropic_key`) live only in localStorage. Never commit a key or token.
- **The join link** (`#join=` + base64 of the gist id and token) carries the token; it is only shared within the family.
- **Fuel** (https://ortizzle.github.io/fuel/, repo `ortizzle/fuel`) reads this app's data read-only through the shared origin's `pack_gist_id` / `pack_gist_token` / `pack_me` keys. Don't rename those keys without updating Fuel.

## Game rules (where to change them)

- **Rules and Merit values:** `RULES`. **One Merit scale for everyone** (Chris chose this, Nov 25, 2026):
  - Showing up pays the same: Active Day +15, Standard +50, chain, together, cheers, goals, Primus, race.
  - **No cliff at 20 minutes** (Chris, Nov 26, 2026): a day short of an Active Day earns part of the +15, about ¾ point a minute (or by MET-minutes for parents), as "Showing up · N of 20 min". The 20-minute line still decides Active Days for the Standard, Primus and the race.
  - **Stretching never makes an Active Day by itself** (`gentle: true` in `ACTIVITIES`). It earns its effort points, and it tops up real movement up to as many minutes as you actually moved (10 walk + 10 stretch = an Active Day; 30 stretch alone isn't). Meditation still doesn't count at all.
  - Re-ran the Dec–Mar fairness simulation after this change: equal effort still gives equal Merit within ~10%.
  - **The Standard is 4 Active Days a week for everyone, kids included** (Chris, Nov 26, 2026). A Dec–Mar simulation showed that with the old 3-day kids' Standard, the kids earned nearly twice the adults' Merit for the same workouts at 3 days a week and took almost every Primus crown. At 4 days for everyone, equal effort gives equal Merit (within ~10%) and the crowns spread out. Don't reintroduce a separate kids' Standard without re-running that comparison.
  - **Holiday weeks** (`HOLIDAYS`, `holidayFor` / `stdDays`): the Standard drops to 3 days for everyone, and missing it doesn't cost a chain link. Dec 21 – Jan 3 is listed; add the girls' spring break when the dates are known. Cards show "of 3 days 🎄" and the Codex lists them.
  - Effort: Praetorians use MET × min ÷ 10; Pups use 1 point per 2 minutes. **Both cap at 60 a day** (raised from 40, Chris, Nov 26, 2026), so a 30-min kid workout ≈ a 30-min adult one at average intensity.
  - **Intensity matters for the adults; volume for the kids** (Chris's goal). Harder efforts should clearly out-earn easier ones (a 60-min run ≈ 59 vs a walk ≈ 26), so don't flatten MET differences for Praetorians. Pups' effort stays minutes-based.
  - **Praetorians' best effort week yet** (+25, `RULES.growth.bestWeek`) rewards beating your own weekly effort (MET × min ÷ 10, uncapped), from your 3rd week on. A harder week counts as a better week.
  - **Pups' growth points** (`RULES.growth`) balance the parents' weight-progress points: best week yet (by minutes), longest outing yet, taekwondo stripe/belt and personal bests. `milestone` records are added via `openMilestone`.
  - Kid bonus points may come later; Chris expects the kids to push harder instead.
- **The Codex** (`openCodex`) shows every rule and number. **It opens with "Two things to know first"** (Chris, Nov 27, 2026): Merit (the whole year, never resets, climbs your rank; "like a piggy bank") vs the Primus (this week's crown, everyone starts at 0 Primus points each Monday; "like a race every week"), in kid words, before any numbers. Keep it first and keep it kid-readable. It's generated from `RULES` / `RANKS`, so it can't drift. Linked from the Pack tab, Settings, and "How points work ›" under The Pack.
- **Activities and MET values:** `ACTIVITIES`.
- **Taekwondo is about 35 minutes of actual training** (`TKD_MIN`), used by the kids' big button, the TKD check-in and sample data.
- **Ranks:** `RANKS`, **one ladder for everyone, built all year** from Dec 1 and never reset (`score().year` = this season plus earlier seasons; Practice Week stands alone).
  - **Names** (Chris, Nov 26, 2026): 🔗 Lowborn → 🐺 Howler → 🗡 Lancer → 🛡 Centurion → ⚔️ Legate → 👑 Gold → ✦ Peerless Scarred. They're book-flavored but explain themselves to non-readers: each has an icon and a one-line `motto` shown in the tour, the Codex and the badge tooltip.
    - They replaced the old color names, which clashed with everyone's chosen Color, and the original Howlers' Latin ranks (Tiro, Decurion…), which meant little to non-readers. Keep `Gold` as a rank name: `goldEarned` looks for it.
  - It's progressively harder, like the original Howlers: 0 / 600 / 2,000 / 4,500 / 8,000 / 12,500 / 18,000. Peerless by year's end takes beating the Standard (~300 Merit a week).
  - Seasons keep things fresh with races, monthly and season goals, and the Hall of Honors (the color at each season's end).
- **The Pack heading** is just "The Pack" with **How points work ›** on the right (Chris, Nov 27, 2026: the two-line explainer under it took too much room). Don't add an intro paragraph back.
- **Pack cards** are sorted by year Merit and show it with the place ("Merit · =1st" on ties), plus progress to the next rank (`nextRankStat`, e.g. "18 to Howler" with a bar in that rank's color). The % to Peerless is inside the expanded card.
- **Cheers** go through `sendCheer` (5 s Undo). Your own cheers can be taken back from the Cheer Wall; the +2 goes with them.
  - **Cheers point at a workout** (Chris, Nov 27, 2026: "clear what is being praised"). A cheer record carries `workout` (the recipient's own copy), `about` (a label like "🚴 Peloton Bike · Ride · 30 min", kept even if the workout is deleted) and `aboutDate`. `cheerFor(c)` renders "for your … (Tue)".
    - The Pack tab feed's emoji buttons cheer that row's workout. Each workout row can be cheered on its own.
    - "Cheer Kat" (`openCheerNote(to, pick)`) asks **What are you cheering?** with their last 3 days of workouts (`recentFor`), the newest picked, plus "🐺 Just cheering Kat on".
    - The cheer card, the cheers lists, the phone notification (`sw.js`) and the Annals entry (`cheersOn(w)`) all show what was cheered. Scoring is unchanged: still +2 per cheer you start.
  - **Only cheers you start earn the +2** (Chris, Nov 27, 2026). Every **❤️ back** button goes through `heartBack`, which saves a cheer with `reply: true`. It still shows and notifies, but scoring skips it and it doesn't use up the daily cheer limit.
  - **New cheers must be impossible to miss** (Chris, Nov 27, 2026; he got none of his on an iPhone).
    - `unseenCheers()` (since `pack_cheers_seen_at`, last 7 days) feed four places: the top of Needs you ("📣 2 NEW CHEERS!" with ❤️ back), a red count on the Pack tab, the app-icon badge (`navigator.setAppBadge`, while the app is open), and and **a big cheer card** (`cheerToast`: slides down from the top with the emoji, who, their message, ❤️ back and See all; ✕ or 10 s to dismiss; it sits above small toasts) shown once per new cheer, when the app opens or a sync brings one in (`toastNewCheers`). Chris asked for it big, Nov 27, 2026.
    - **The Pack tab opens with a banner** (`packBanner`, same style as Today's; Chris said the top looked empty, Nov 27, 2026).
      - New cheers lead it. That's where the Pack-tab badge points, so visiting marks them seen and the badge disappears right away. They stay listed for that visit.
      - Otherwise it shows the family's week ("N Active Days between you, M of 4 met the Standard, K together") with the count of cheers this week.
      - Below it are three small tiles: active days → Explore (Active Days, everyone), met the Standard → this week's recap, together → Explore (Together, everyone). They replaced the old wrapping "ACTIVE DAYS THIS WK" tiles.
    - The **Cheers** tile on a Pack card opens `openCheersFor(id)`: your cheers (opening them marks them seen), or someone else's plus "📣 Cheer them".
  - **Background notifications only exist on Android Chrome** (Periodic Background Sync in `sw.js`; Chrome picks the cadence). iOS has no periodic sync, and real push would need a small push server, which the app doesn't have. `enableNotifications` says so on iPhone.
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
  - **Weekly classes** (`member.weekly: [{act, days, min}]`, Settings → My weekly class): `classPrompts()` checks in after noon on the day, or for up to two days after. Kat's Sunday pickleball is pre-set at 45 min (`migrateWeekly`; it also fixes the first 90-min preset once). **The check-in learns the real length** (`learnedMin` / `classMin`): once you've logged that activity 3+ times in 60 days, it uses your most common length (rounded to 5 min) instead of the preset, and Settings shows "(your usual)".
  - **Usuals are suggestions, never blind logs** (Chris, Nov 25, 2026): tapping one opens the log sheet pre-filled (`quickUsual` → `openLog({ act, min, cls })`).
  - `topUsuals(id)` ranks your most-logged activity + class type over the last 60 days (2+ times, at your most common length, rounded to 5 min). Usuals saved in Settings fill the spare spots, 6 in all; a saved Tread/Bike usual with no class type gives way once history has that machine.
  - `migrateUsuals()` adds a Walk usual once per parent.
  - **★ Pinned usuals** (`pin: true`, toggled in Settings → My usuals) always come first.
  - **Pups get their own favorites:** their top 3 from history sit above the big buttons in `openPupChooser`.
- **"⏱ Got a few minutes?" calculator** (`openCalc`, Chris, Nov 25, 2026): ranks every activity and class type by the Merit it would earn *today* for the chosen minutes.
  - `previewGain` does a dry run through the real scoring (temporary `__calc` record, never saved), so Active Day, Standard, chain and the 60/day cap are all included.
  - It uses your own pace from your latest entry with speed/incline/watts (`myPace`).
  - Tapping a row opens the log sheet pre-filled.
  - When the 60/day effort cap makes several rows tie, it says so ("N left today") and each capped row shows **maxes out in N min**. Ties sort by that, so the hardest workout comes first.
  - Pups see one line ("any N minutes = +X"; every activity counts the same for them) plus their favorites, with no MET talk.
- **Where & how far** (Chris, Nov 25, 2026): every log sheet, kids' included, shows a route/place box (`title`) and, for distance activities, miles (`detail.miles`) right after "When". They're visible, never required, and recent places (`recentPlaces`) are one-tap chips so they're nearly always answered. These fields are not in "More stats".
- **Editing:** tapping an Annals entry calls `openLog({ edit: record })`, which updates that record in place.
  - **Only the person who logged a workout (`by`) can change its shared facts** (activity, class type, minutes, date, place, distance, dogs) and use Apply to everyone (Chris, Nov 25, 2026). `canChange` / `canApplyAll`.
  - Anyone else opening their copy gets a locked summary ("Logged by Chris…"). They can still set how it felt, heart rate, calories and notes, or tap **I wasn't there** to remove their copy.
  - **Parents keep fix-up rights on the kids' entries.** When a parent fixes a kid's copy of a group someone else logged, the change stays on that copy.
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
- **Weekly Primus:** `primusScores` / `primusPoints`. **It goes to whoever earns the most Primus points that week** (Chris, Nov 27, 2026: "it needs to be based on points… clear to understand").
  - Primus points are plain workout Merit: Active Day / showing-up points, the Standard bonus and effort. Cheers, goals, quests, chain bonuses, stripes and weight points don't count.
  - The Primus card and the Pack card show "N pts". The old "beat your own target" rule was dropped because nobody could read it.
  - **Tap a Primus row for the breakdown** (`openPrimusBreakdown` / `primusBreakdown`; Chris, Nov 27, 2026): the total, three chips (Active Days · The Standard · Effort) and each day's entries. Kids' sheets use minutes wording, never MET.
  - **Watch the kids in December** (Chris, Nov 27, 2026). He expects the adults to get competitive and out-train the kids, so the Primus may later need extra points for the kids. Don't change it until he has seen December's results.
  - In the Dec–Mar simulation the crowns still spread (realistic family 5 · 3 · 5 · 4), and the harder trainers win more often.
  - On the same easy walk the kids out-earn the parents slightly (kids' effort counts minutes), which is the agreed scale.
  - **Ties are never broken alphabetically.** A tie at the top (`primusLeaders`) shows no crown and an "anyone can take it" note. If the week ends tied, each tied person gets a shared crown (`aw_primus_<week>_<id>`, "(shared)") and the +25.
  - **Expect the girls to tie often** (Nov 27, 2026). Kids' Primus points come only from minutes, so shared workouts give them identical totals. In the simulation they tied 14 of 17 weeks when they did nearly everything together, and 0 with some solo workouts each. Chris chose to **keep shared crowns through December and watch**. If shared crowns feel flat, the next step is tie-breakers: more Active Days, then more minutes, then shared. Don't add them before he's seen December.
- **Coaches:** `COACHES`. Dancer, Mickey and Matteo come from the original Howlers; **Sevro** (loud, funny, loyal, loves a rivalry; never swears) was added Nov 26, 2026 to fill the "hype and fun" style the girls may want.
  - **Everyone picks their coach by how they like to be encouraged** (`openCoachQuiz`): 4 questions, each answer pointing to one coach's style (`COACH_QUIZ`). The majority wins, ties go to the "after a great workout" answer. The result shows the match first plus all coaches (`COACH_FOCUS`: style, blurb, focus), and any of them can be picked.
  - The quiz is offered on Today until done (`coachPrompt`, `member.coachPicked`) and anytime from Settings → My coach. The model is `claude-opus-5-5` with effort `low`, browser-direct, with server-side fallbacks.
  - **Coaches follow the books as the family reads them.** All four appear in *Red Rising* (book 1). **Ragnar** (calm, wise strength) is saved for year 2, once the story reaches him (Chris, Nov 26, 2026). Don't add characters from later books before then.
  - **After Practice Week the coach lives in the season banner** (`coachBanner`), the place Chris hopes everyone reads daily (Nov 26, 2026).
    - **Split layout (option 1, Chris, Nov 26, 2026):**
      - **The banner keeps the Practice Week banner's look** (Chris preferred it, Nov 26, 2026): the season as the title, the coach's note as the one sentence (`coachWord`: one Claude message a day saved in `pack_coach_day` when there's a key, else the built-in `coachLine`), "— Dancer · notes ›" underneath, and the days-left counter on the right (`.cb-classic`). Tapping it opens the notes. Don't turn it back into a busier card.
      - **Three tiles that never move:** days this week (✦ when the Standard is met, 🎄 in holiday weeks), Primus place (♛ when leading), and **a goal** (Chris, Nov 26, 2026; it replaced "Merit to next rank").
        - The goal tile (`goalTile` / `bannerGoal`) shows the goal you 📌 pinned on the Goals tab (`pack_banner_goal`, per phone), else your first unfinished monthly goal, else a season goal, e.g. "3/8 🐕 dog walks".
        - With no goals it shows "🎯 Set a goal".
      - **The tiles are small shortcut buttons** (44px, `jumpTo`). Days opens My week on Progress (`.my-week`), Primus opens the Primus tab under This week, and the goal tile opens that goal on the Goals tab (`.goal-<scope>-<type>`). The target flashes gold.
    - Tapping the banner opens `openCoachNotes`: the note, every insight (`bannerInsights`: holiday week, Standard, challenge, family quest, next rank, Primus race, chain, race checkpoint) and, with a key, "Ask for a fresh note".
    - The separate coach card shows only before Dec 1. Kids' banner content follows the no-weight/food/body rule.

## Today and Progress layout (Chris, Nov 26, 2026)

- **Today has three zones (option A).** Chris also liked B (a Me | Pack switch) and C (choose your own cards) for later.
  1. **The coach banner.**
  2. **The Pack and the log box**: Log a workout, usuals, a two-button tools row (⏱ Got a few minutes? / ✨ Paste or snap), and the weigh-in for parents.
  3. **🔔 Needs you** (`needsCard`, a `.zone`, not a `.card`): every prompt in one place, time-sensitive first (added for you, taekwondo, stripes, weekly class, challenge invites, quest ideas, active quests, recap), then setup (Color, coach), then goals. Two show; the rest sit behind "Show N more" (`.needs-more`).
  4. **This week** (`weekTabs`): Primus | Race | Challenges tabs. Inactive panels stay in the DOM but hidden; challenge invites live in Needs you.
  - Cheers moved to the Pack tab. "This week" (`myWeekCard`) and Fuel (parents) moved to Progress.
  - Don't reuse `.more-btn` for anything but the log sheet's "More stats"; tests and the sheet depend on it.
- **Progress starts with 📊 Explore my data** (`exploreCard`): one chart and a swipeable row of choices (`EX_METRICS`).
  - **Moving:** minutes, Active Days, effort (parents), Merit.
  - **Together:** together workouts, dog walks, family quests.
  - **Activities:** one activity (pick its icon).
  - **Wins:** Primus crowns, challenges won, stripes & belts (kids).
  - **⚡ Am I getting faster?** (`fastOptions`: mph from speed or miles ÷ time, watts for the Bike; bootcamps skipped; a dashed least-squares trend, plus a plain sentence comparing the first and last up-to-3 workouts).
  - **🧩 Where my points came from** (by category, this week or this month).
  - Pickers: by week (8) / by month (6), just me / me vs everyone (legend with names, tap a bar for its number, "Show the numbers" table). "☆ Make this my chart" saves `pack_explore_fav`, which opens first.
  - Kids get friendly emoji labels and never weight data.

## Look: Stone & Paper (Chris, Nov 26, 2026)

- **Parchment cards laid on warm carved stone.** It's one CSS block at the end of `<style>`, chosen from prototypes A–D.
  - Paper cards have a faint fiber texture, a gold inner rule (`outline-offset: -6px`) and hand-cut corners. Headings get a ✦ and a gold rule.
  - Soft carved buttons; a rounded, embossed **Log a workout** (crimson for parents, orange for Pups).
  - Swipe rows (`.ex-metrics`, `.ex-scroll`) fade at the edges.
- **Chris said no rivets, no pennant banners, and a regular button (not a banner shape).** Keep it that way.
- Cards nested in Needs you / This week drop their frame (`.need .card`, `.tab-panel .card`), so there's no double border.
- Textures are inline SVG noise data URIs; no image files.

## Family quests (Chris, Nov 26, 2026)

- **"Something we all do together from time to time."** `quest` records, **designed only by parents in Settings → Family quests** (`openQuestPlanner`; Chris, Nov 26, 2026). Parents can also remove one there; its Merit goes with it. Everyone else sees quests read-only on Today and the Pack tab.
  - **The girls can suggest quests** (`openQuestSuggest`, Pack tab): a `quest` with `status: 'suggested'`, at most 2 waiting per kid.
    - Parents see them on Today (`ideaPrompt`) and in Settings, with **Approve** (`approveIdea`), **Edit** (the planner opens with the idea and saves under the same id) or **Not now** (`status: 'declined'`; the kid sees "maybe another time").
    - `quests()` excludes suggested and declined ones, so only approved quests count or award Merit.
  - **Special outing** (`kind: 'event'`): the listed members do one activity together (a single group entry) by the date. The whole pack earns +50 each; if it ends with only some of them, +15 each for those who came.
  - **Team total** (`kind: 'total'`): the family's minutes reach a target by the date, +30 each for everyone who moved.
  - Values are in `QUEST`.
- `syncQuestAwards()` (run from `renderToday`) writes deterministic `aw_quest_<quest>_<member>` awards and removes stale ones if a workout is deleted.
- Today shows active quests with who's joined and a "Log it together" button that opens the log sheet with the activity, everyone and the place filled in (`openLog` takes `crew` / `title`). The recap and banner mention them, and the Codex lists the values.

## Colors (Chris, Nov 26, 2026)

- **Everyone picks their own Color** from `COLORS` (12 Colors from the books, each with a kid-language meaning). Stored as `member.house`, with `colorPicked`.
  - Picked in the welcome tour, from a Today prompt until chosen (`colorPrompt`), or in Settings → My Color.
  - Brown is left out on purpose.
- **Gold can't be picked; it's earned.** Reaching the Gold rank makes you Gold (`goldEarned` / `colorOf` / `colorLabel`), shown as "GOLD · BORN VIOLET", with a gold card. Peerless shows "Peerless Scarred".
  - Chris may adjust when Gold is earned at each quarterly status review.
- **Gold shimmer:** the weekly Primus leader's row and the Primus card shimmer (`.gold-shimmer`, still under reduced motion).

## Weekly recap and challenges

- **Week in review** (`recapCard` / `recapBody`): on Today from Sunday 5pm, and through Monday and Tuesday for the week just ended, until dismissed (`pack_recap_seen`).
  - Shows the Primus line (gold shimmer), each person's days vs Standard, minutes and week Merit (`weekMerit` = `score().byWeek`), challenge results, and workouts done together.
  - **📣 Cheer the pack** cheers everyone who met the Standard (within the daily cheer limit).
  - The Pack tab has "This week so far" / "Last week" buttons.
- **Head-to-head challenges** (`challenge` records: `ch_<week>_<from>_<to>`, `metric` minutes|days|merit, `status` open|on|no).
  - Anyone can challenge anyone for the current Mon–Sun week, once per pair. The other person accepts or picks "Not this week".
  - The Today card shows live standings.
  - **Bragging rights only: no Merit changes hands** (keeps the scoring simulation valid).

## Welcome tour

- **`openTour(id, mode)`:** swipeable cards (Back/Next, dots) shown once per person per phone (`pack_tour` → `w_<id>` / `p_<id>`).
  - Pups get 7 kid-language cards: the pack and dogs, minutes → Merit, the 4-day Standard with their own bar, the color ladder, "Yes, you can beat Mom and Dad" (Primus), more ways to earn, and how to log. No weight, food or body words.
  - Praetorians get 3 cards.
- **"The Passage begins"** card shows once for everyone on/after Dec 1. It's appended to the welcome for anyone opening for the first time then.
- **Replay** from the Codex ("▶ Replay the welcome tour").
- **`maybeTour` runs at the end of `render()`.** It skips sample-data mode and automated browsers (`navigator.webdriver`) unless `pack_tour_test` is set, so other tests aren't blocked by the modal.

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
