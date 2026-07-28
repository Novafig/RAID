# RAID — project context for Claude Code

## What this is

RAID (Resistance · And · Inverted · Development) is a **single-file offline workout tracker**.
It is one file: `index.html`, roughly 220KB, served by GitHub Pages and pinned to an iPhone
home screen as a PWA.

## Non-negotiable architecture rules

Read these before changing anything.

1. **It must stay ONE file.** All CSS lives in the single `<style>` block, all JS in the single
   `<script>` block. Do NOT split into `style.css` / `app.js` / modules. The single-file design is
   deliberate: it makes the app trivially portable and fully offline.
2. **No build step. No npm. No bundler.** Plain HTML/CSS/JS only.
3. **No external network requests** except outbound YouTube tutorial links (`https://youtu.be/...`)
   that open in the browser. No CDNs, no Google Fonts, no analytics, no API calls. The app must work
   in airplane mode.
4. **No frameworks.** No React/Vue/jQuery. Vanilla DOM only.
5. **Fonts are embedded as base64** (Rajdhani, OFL licensed) inside the `<style>` block. Don't
   replace them with a CDN link.
6. **Icons are embedded as base64** data URIs in the inline PWA manifest.
7. **Never use `localStorage`-adjacent browser APIs for anything but the one storage key** (below).
8. **Never use `confirm()`, `prompt()`, or `alert()` for required flows.** iOS webviews block or
   silently cancel them. Native dialogs already caused one bug here. Use in-app UI instead
   (the level-up button uses a two-tap arm; restore-from-backup uses a DOM textarea).

## Layout of the script block, in order

1. Constants: `KEY`, day-of-week helpers, date utils
2. `SKILLS` — the 7 calisthenics skills (frog, crow, elbow, pistol, headstand, handstand, lsit),
   each with an inline SVG `figure()`, form text, a step ladder, and a tutorial URL
3. `RECOMP_DAYS` — the flagship program's literal day table (Mon Arms → Sun Rest)
4. `MORNING_A` / `MORNING_B` — the flagship's alternating morning core routines
5. `LIB` — shared exercise library (~74 entries) used by all non-flagship programs.
   Each entry: `{name, gear:[tokens], fig, kind, form, tip?, side?, cardio?, lv:[6 levels]}`
6. `PROGRAMS` — 7 selectable programs keyed by id. `recomp` is the flagship (`flagship:true`,
   `skills:true`) and uses `RECOMP_DAYS` verbatim. The other 6 are built from `LIB` via
   `buildEx([libId, sets, lo, hi, weight])`.
7. `GEAR_CATS` — gear options in 4 categories (Free weights, Machines, Cardio, Bands & bodyweight),
   plus `GEAR_HOME` / `GEAR_GYM` presets
8. `applyProgram()` — **the engine.** Repopulates the mutable globals `DAYS` and `PROG` from
   `store.program`. All render code reads `DAYS` / `PROG`, never `PROGRAMS` directly.
   Must be called after any change to `store.program`, `store.emphasis`, or `store.schedule`.
9. `figure()` / `exFigure()` — inline SVG movement sketches, shared by movement pattern
10. Render functions: `renderHome`, `renderToday`, `renderSkills` (the "Guide" tab),
    `renderHistory`, `renderSettings`
11. `render()` — dispatcher. Only scrolls to top on tab change or Today date change.
12. A single delegated click handler keyed on `data-a="..."` attributes
13. Boot: `store = load()` → `applyProgram()` → `applyTheme()` → `render()`

## Tabs

Home · Today · Guide · Log · Setup.
Note: the Guide tab's internal id is `"skills"` for historical reasons; its label is "Guide".

## State and storage

- `store` = persisted user data, single localStorage key **`loadout.v1`**
- `state` = ephemeral UI state (open panels, selected date, pickers). Never persisted.
- `load()` migrates and backfills older saves — **always add a default there when you add a
  `store` field**, or existing users will break on upgrade.
- `hydrate()` does the same backfill for an imported backup.
- Progress is **per-device, not per-user, and never synced.** No accounts, no server.
  Backup/restore is manual copy-paste JSON in Setup → Data.

`store` shape: `{bodyweight, theme, level, skillDays, skillSteps{}, exLevels{}, overrides{},
anim, program, location, gear{}, emphasis, schedule, logs{}}`

Log shapes inside `logs[date]`:
- lifts → `exId: {sets:[{v, w, done}]}`
- ruck → `{min, load, done}`
- skills → `sk_<skillId>: {rounds, result, done, step}`
- manual days → `man_<id>: {sets}`

`overrides[date]` = `{type:"skip"}` or `{type:"manual", items:[...]}`

## Level ladders

Every flagship exercise and every `LIB` exercise has `lv: [...]` — 6 progression steps described
in text. Current level per exercise is `store.exLevels[exId]` (0-indexed).
Barbell/machine ladders progress **technique, range, and tempo**, not load — actual weight is
logged per set, so a ladder step that just says "add weight" is useless. Keep that convention.

## Testing changes

There is no test framework. To sanity-check a change:

1. Confirm the JS parses — extract the `<script>` contents and run `node --check` on it.
2. Open the file in a browser and click through **all five tabs**, then switch through **all seven
   programs** in Setup, since program switching rebuilds `DAYS` and is the most common source of
   breakage.
3. Check both `ampm` and `eve` schedule modes on the flagship.
4. Verify a fresh load works with empty localStorage (incognito window), which exercises
   the `load()` defaults path.

## iOS gotchas learned the hard way

- Use `style="fill:var(--x)"` in SVG, **not** presentation attributes like `fill="var(--x)"`.
  Safari ignores `var()` in presentation attributes.
- Animations are default-ON with a Setup toggle. Do not gate them behind
  `@media (prefers-reduced-motion)` — that check misbehaves in home-screen webviews.
- After deploying, iOS caches aggressively. Hard-refresh, or delete and re-add the home screen icon.

## Style

Tactical/HUD aesthetic. Amber accent, dark default, Rajdhani for UI chrome, system font for body
text. Per-muscle-group accent colors. Keep copy terse and direct — no marketing voice, no emoji.
