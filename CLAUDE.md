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
   `skills:true`) and is built from `RECOMP_DAYS`. The other 6 are built from `LIB` via
   `buildEx([libId, sets, lo, hi, weight])`.
7. `GEAR_CATS` — gear options in 4 categories (Free weights, Machines, Cardio, Bands & bodyweight),
   plus `GEAR_HOME` / `GEAR_GYM` presets
8. `EMPH_UPPER` / `EMPH_LOWER` / `bonusFor()` — the emphasis engine. `bonusFor` returns up to
   `EMPH_N` (2) accessories to append to a day, rotated by weekday so different days get
   different pairs. It is **additive only** — nothing is ever removed, because dropping an
   exercise would make `exLog()` truncate sets the user already logged. It skips gear the user
   doesn't own and any id already on that day (duplicate ids share one log entry and fight over
   the set count). Everything it returns is tagged `bonus:"upper"|"lower"`, which the Today card
   and the Setup note use to show what the emphasis added. `UPPER_KEYS`/`LOWER_KEYS` both
   include `"full"`, so full-body programs (Toning, Endurance) count as both.
9. `applyProgram()` — **the engine.** Repopulates the mutable globals `DAYS` and `PROG` from
   `store.program`. All render code reads `DAYS` / `PROG`, never `PROGRAMS` directly.
   Must be called after any change to `store.program`, `store.emphasis`, `store.schedule`,
   **or `store.gear` / `store.location`** (emphasis bonuses are gear-filtered).
   It always builds fresh day objects — never assign `DAYS = RECOMP_DAYS`, or appending a bonus
   would mutate that constant for the rest of the session.
10. `PALETTES` + `applyTheme()` — the theme system (see **Themes** below)
11. `exMeta()` / `logDayInfo()` / `stampProgram()` — resolve exercises and day labels for logs
    that were recorded under a program you've since switched away from
12. `figure()` / `exFigure()` — inline SVG movement sketches, shared by movement pattern
13. Render functions: `renderHome`, `renderToday`, `renderSkills` (the "Guide" tab),
    `renderHistory`, `renderSettings`
14. `render()` — dispatcher. Only scrolls to top on tab change or Today date change.
15. A single delegated click handler keyed on `data-a="..."` attributes. `LOG_ACTS` lists the
    actions that write to `logs[date]`; they call `stampProgram()` first. **Add new
    log-mutating actions to `LOG_ACTS`.**
16. Boot: `store = load()` → `applyProgram()` → `applyTheme()` → `render()`

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

`store` shape: `{bodyweight, theme, palette, level, skillDays, skillSteps{}, exLevels{},
overrides{}, anim, program, location, gear{}, emphasis, schedule, logProgram{}, setupDone,
restSec, restBeep, logs{}}`

- `theme` is the light/dark **mode** only (`"dark"|"light"`); `palette` is the colour theme id.
- `logProgram[date]` = the program id that was active the first time that date was logged.
  Written by `stampProgram()`, read by `logDayInfo()` so the Log tab can label an old session
  with the program you actually trained. Dates missing from it (logs from before this existed)
  fall back to the current program.
- `setupDone` gates the first-run wizard — see **First-run wizard** below.
- The "Erase all logs" handler rebuilds `store` field by field. **Add new non-log fields there
  too**, or erasing logs will silently reset them (this already nearly shipped a bug where
  erasing logs reset the theme, and another where it relaunched onboarding).

Log shapes inside `logs[date]`:
- lifts → `exId: {sets:[{v, w, done}]}`
- ruck → `{min, load, done}`
- skills → `sk_<skillId>: {rounds, result, done, step}`
- manual days → `man_<id>: {sets}`

`overrides[date]` = `{type:"skip"}` or `{type:"manual", items:[...]}`

## Themes

Two independent axes: **mode** (`store.theme` = dark/light) and **colour theme**
(`store.palette`). `applyTheme()` stamps `data-theme` and `data-palette` on `<html>`, then reads
the resolved `--bg2` back out with `getComputedStyle` to set the `theme-color` meta — so palettes
never need their background duplicated in JS.

A palette tints exactly **three** things, and nothing else:

| var | what it colours |
|---|---|
| `--bg2` | the top of the page gradient |
| `--line` | every border, including the card borders |
| `--amber` | the accent |

**Card and text neutrals are mode-level, never per-palette.** `--surface` is `#0A0A0C` in dark
and `#FFFFFF` in light, so cards read black/white in every theme; `--surface2`, `--ink`,
`--muted`, `--faint` and the `--bg` gradient base are likewise fixed per mode. Don't move them
into a palette block — that was the original design and it made every theme tint the cards.

The page gradient is painted by `body::before`: `position:fixed; inset:0; z-index:-1`, running
`--bg2` → `--bg` top to bottom. Fixed to the viewport so the tint stays at the top of the screen
regardless of page length or scroll — `background-attachment:fixed` is unreliable in iOS Safari.
It's a **linear** gradient, not radial, so every pixel of the top row is exactly `--bg2` and the
sticky `header` (which fades from `--bg2`) meets it with no seam. Change one, change the other.

CSS lives in blocks like `html[data-palette="zora"][data-theme="dark"]`. The two-attribute
selector outranks the base `:root` / `html[data-theme="..."]` block. `default` has no block at
all — it *is* the base.

`--onacc` is the text colour for anything sitting **on** an `--amber` background (`.cb`,
`.logtoggle.on`): dark in dark mode, white in light mode, because light-mode accents are dark.
Backgrounds that use the fixed `--go`/`--gold` instead keep a literal dark label — don't switch
those to `--onacc` or they turn white-on-green in light mode.

What deliberately does **not** change per palette: `--go`, `--gold`, `--red`, `--blue` and the
per-muscle-group colours. Those are semantic — "done", "rest", "skip" and each muscle keep one
meaning everywhere. Several glows/tints are also hardcoded rgba of those colours, so re-huing
them per palette would desync a border from its glow.

In the base blocks, `--arms: var(--amber)` and `--rest: var(--faint)` so they follow the palette.
`PALETTES` in JS holds only the picker label and one accent swatch hex per mode — adding a
palette means a CSS block pair **and** a `PALETTES` entry.

## Rest timer

`store.restSec` (0 = off, default 90) and `store.restBeep`. Ticking a set starts a countdown in
`#restbar`, which lives **outside `#app`** in the static markup and is painted by `paintRest()`
on an interval — never through `render()`, which rebuilds all of `#app` and would fight the
number inputs every second. It has its own click listener for the same reason.

The end time is stored **absolutely** (`_restEnd = Date.now() + …`), so backgrounding the app and
returning re-derives the remaining time instead of drifting.

The beep is **scheduled ahead** via `osc.start(ctx.currentTime + delay)` at the moment of the tap,
not played when the countdown reaches zero: the tap is the user gesture that unlocks the
AudioContext on iOS, so scheduling inside it is far more reliable. Adjusting or dismissing has to
`cancelBeeps()` and re-schedule. Beeps are generated with an OscillatorNode — no audio file,
which would break the no-network rule.

**This is foreground-only and that is a platform limit, not an oversight.** iOS suspends timers
once the app is backgrounded, and no offline web app can schedule a real notification: the
Notification Triggers API was never shipped by Safari, and Web Push (incl. iOS 18.4's Declarative
Web Push) needs a live server to send each message. A service worker is doubly impossible here —
its script must be a real same-origin URL, and `blob:` is rejected, so a one-file app cannot
register one. Don't "fix" this by adding a server or a second file. Say so plainly in the UI.

## Auto-scroll

`scrollToNextEx(id)` runs after `render()` in the `done` handler, but only when a set-tick just
*completed* an exercise (never on unticking). Cards carry `data-ex="<exId>"` as anchors; the
scroll target is offset by `header.offsetHeight` because the sticky header would otherwise cover
the next card's title. Honours the animations toggle by falling back to `behavior:"auto"`.

## First-run wizard

`store.setupDone` gates it. `renderWelcome()` runs a 3-step flow (program → gear → theme) and
`render()` short-circuits to it while the flag is false, with `html[data-wiz="on"]` hiding the
tab bar. The steps reuse `progBtnsHTML()` / `locBtnsHTML()` / `gearAccHTML()` / `modeBtnsHTML()`
/ `palChipsHTML()`, which the Setup tab also uses — so the wizard's controls are the *same*
controls and fire the same `data-a` actions. Nothing wizard-specific persists.

`setupDone` is backfilled to **true** in both `load()` and `hydrate()`, so an existing save or a
restored backup never sees onboarding — only a genuinely empty localStorage does. It is also
carried through the erase handler; forgetting that would relaunch onboarding on "Erase all logs".

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
3. Check both `ampm` and `eve` schedule modes on the flagship, and all three emphases.
4. Verify a fresh load works with empty localStorage (incognito window), which exercises
   the `load()` defaults path.

### If there's no `node` on the machine

Headless Chrome is a better check anyway — it proves the script parses *and* boots *and* renders.
Copy `index.html`, strip the trailing `</body></html>` (keep the app's own `</script>`), append a
`<script>` that drives the app and dumps assertions into a `<pre>`, then:

```
chrome.exe --headless --disable-gpu --no-sandbox --virtual-time-budget=20000 \
  --user-data-dir=<tmp> --dump-dom "file:///<path>" > dom.html
```

Assemble that copy with `cat`, **not** PowerShell `Get-Content`/`Set-Content` — PS 5.1 misreads
UTF-8 as ANSI and corrupts every `·`, `✓`, `‹`, `–` in the file.

Gotchas: the harness must be **synchronous** (`--dump-dom` captures before a `setTimeout` fires);
scope text assertions to `app.textContent`, not `document.body.textContent`, or they'll match the
harness's own source; and this Chrome clamps the headless viewport to ~500px wide, so
`--window-size=390,...` screenshots come out cropped rather than reflowed — measure overflow by
pinning `document.documentElement.style.width` instead.

## iOS gotchas learned the hard way

- **Bind gestures to `#app`, not to elements a render replaces.** The week-rail swipe used to
  re-attach `touchstart`/`touchend` to `.rail` at the end of every `renderToday()`. It stopped
  responding after scrolling: the chips are composited layers (`perspective` + `preserve-3d`)
  inside a `position:sticky` header, and WebKit hit-testing through that stack after a scroll
  stops delivering touches to the element you attached to. `initRailSwipe()` now binds **once**
  to `#app` (which survives every render) and matches the gesture start **geometrically** against
  the `.railwrap` rect, so it doesn't matter which descendant the browser thinks was touched.
- **`position:fixed; bottom:0` is not the bug you think it is.** The tab bar looks like it lifts
  off the bottom on the Log tab; it doesn't move at all (measured: identical rect on every tab).
  Safari keeps its bottom toolbar expanded on pages that can't scroll, so the bar sits above it,
  while scrollable tabs auto-collapse the toolbar. Nothing in CSS fixes that — and the pinned
  home-screen app has no toolbar, so it never happens there. `.empty` now fills the viewport
  (`dvh`, flex-centred) so short pages at least look deliberate.
- A swipe handler needs `touchmove` with `{passive:false}` and a direction lock: `preventDefault()`
  once the gesture is horizontal (otherwise iOS steals it and you get `touchcancel` instead of
  `touchend`), and stay passive when it's vertical so the page can still scroll. Handle
  `touchcancel` by resetting, never by committing.

- Use `style="fill:var(--x)"` in SVG, **not** presentation attributes like `fill="var(--x)"`.
  Safari ignores `var()` in presentation attributes.
- Animations are default-ON with a Setup toggle. Do not gate them behind
  `@media (prefers-reduced-motion)` — that check misbehaves in home-screen webviews.
- After deploying, iOS caches aggressively. Hard-refresh, or delete and re-add the home screen icon.

## Style

Tactical/HUD aesthetic. Amber accent, dark default, Rajdhani for UI chrome, system font for body
text. Per-muscle-group accent colors. Keep copy terse and direct — no marketing voice, no emoji.
