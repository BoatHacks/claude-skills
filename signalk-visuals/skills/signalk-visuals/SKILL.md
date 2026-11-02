---
name: signalk-visuals
description: Use when building or restyling the webapp of a SignalK server plugin so it matches the dark "tactical console" look used on the boat's Status Tiles display. Covers the day/night palette driven by environment.mode (dark in both modes, never a white mode), flat panels with corner brackets and state colours where missing data is neutral never green, a buildless stack (vanilla custom elements, or Preact for form-heavy apps) with vendored dependencies and an explicit browser floor such as Chromium 69 for Navico MFDs, components that update in place, one throttled WebSocket with backoff and an offline banner, display units from path meta.displayUnits with a safe formula parser, ship time versus UTC with Z, phone and nav-station layouts with 48 px targets, hardware-style forms, the favicon and theme-color rules, the house style and prompt template for generated plugin icons, and how to verify it headless.
---

# SignalK plugin webapp visuals

One look for every webapp served by a SignalK plugin on the boat: a dark hardware console that
stays readable in glare and does not ruin night vision. The base rules come from the
`signalk-visuals` guideline in [bergie/dotfiles](https://github.com/bergie/dotfiles/blob/main/pi/.pi/agent/context/signalk-visuals.md)
(written for the vessel *Lille Ø*). The sections marked **Decision** are choices made while
applying it in two plugins, `signalk-huawei-b311-221` (LTE router, a scrolling page with forms)
and `signalk-ais-statistics` (charts, radar, map). Where those two disagreed, the
recommendation here picks one and says why.

If a user's own guideline file exists in the repo or context, it wins over this skill.

## 1. Stack, structure and updates

- **Buildless, no bundler, no transpile step of your own.** Author native ES modules that the
  browser runs as served, with relative import paths. TypeScript is welcome as a checker only:
  `tsc --noEmit` with `checkJs` over JSDoc-annotated `.js`, so types are verified and nothing
  is emitted.
- **Pick one of two UI approaches and do not mix them in one webapp:**
  - **Vanilla custom elements** (the default of the bergie/dotfiles guideline, used by
    `signalk-huawei-b311-221` and `signalk-ais-statistics`): no framework, smallest download,
    you own the update discipline below.
  - **Buildless Preact** (used by [`hoeken/signalk-maintenance-tracker`](https://github.com/hoeken/signalk-maintenance-tracker/blob/main/docs/specification.md),
    spec draft v1): worth it for form-heavy, table-heavy apps. The stack there is Preact 10
    (React-compatible hooks, about 4 KB), **htm** for JSX-like tagged templates (no JSX compile
    step), **@preact/signals** with a small hand-rolled polling layer for data, a hash router
    (preact-router in hash mode, or a tiny one), **snarkdown** for Markdown with the output
    sanitised before insertion, **day.js** for calendar-aware date maths, and **Bootstrap Icons**
    as the single icon set (vendored CSS plus woff2, used as `<i class="bi bi-...">`). Tables,
    forms and modals are hand-rolled with plain CSS: no component library, because the
    sort/filter/paginate rules are app-specific and a heavy table or UI dependency costs more
    than it saves. Tests: vitest with @testing-library/preact on jsdom.
  - The guideline text bans React-style frameworks outright. Preact is a deliberate exception
    that keeps the intent (small, no build, granular DOM patching by the virtual DOM), so tell
    the user it is a deviation before choosing it, and check licences (below).
- **Vendor every runtime dependency.** Commit pinned ESM copies under `public/vendor/` and
  import them by explicit relative path (import maps need Chrome 89, so bare specifiers are out).
  Vendor stylesheets and font files the same way and load them with a `<link>`. No CDN, no remote
  fonts, nothing fetched at runtime: boat servers are often offline or metered. Optional map tiles
  are the one exception and must degrade (section 11). Keep each library's licence file, and any
  dependency must be compatible with **EUPL-1.2** (MIT, BSD, ISC, Apache-2.0 all are).
- **Browser floor.** Decide it up front. Navico / B&G multifunction displays run **Chromium 69**
  (September 2018) and the maintenance-tracker authors to that floor and enhances upward, never
  the reverse. If MFDs are a target:
  - Author to an ES2017 baseline. Avoid `?.`, `??`, logical assignment and private `#fields` in
    your own code (a transpiling reverse proxy can down-level vendored code, but do not rely on it).
  - Transpiling rewrites syntax only. It does not polyfill runtime APIs: no `structuredClone`,
    `Array.prototype.at`, `Object.fromEntries`, `Promise.allSettled`, `String.prototype.matchAll`,
    `globalThis` or `queueMicrotask` without your own polyfill.
  - CSS safe on 69: custom properties, flexbox, grid including grid `gap`, `position: sticky`,
    media queries, woff2. Avoid or treat as optional enhancement: flexbox `gap` (84), `:is()` /
    `:where()` (88), `aspect-ratio` (88), `:has()` (105), container queries (105), `@layer` (99),
    CSS nesting (112). Use grid `gap` or margins for spacing that must exist on the floor.
  - **`clamp()`, `min()` and `max()` arrive in Chromium 79**, so the fluid sizes recommended below
    need a plain declaration before them (`font-size: 1.7rem; font-size: clamp(...)`): an
    unsupported declaration is dropped and the earlier one stays.
  - Rule of thumb: an unsupported feature must degrade to acceptable, never to broken.
    `prefers-color-scheme` and `prefers-reduced-motion` are fine as pure enhancements.
  - Without a build to catch mistakes, compatibility is a review item. Smoke-test load, list,
    detail and one write flow on a Chromium 69 engine.
  - Not needed for phones and laptops only. Say so in the README if the floor is modern.
- **Backend stack is out of scope here.** The same spec also covers the server side (TypeScript
  compiled with `tsc`, `node:sqlite`, the Express router from `registerWithRouter`); see the
  `signalk-plugin` skill for plugin authoring.

- **Nothing from the internet at runtime.** Boat servers are often offline or metered. Ship the
  page's code, fonts and icons with the plugin. No CDN, no remote fonts. Optional map tiles are
  the one exception and must degrade (see section 11). Keep any third-party licence file.
- **Granular DOM updates.** With vanilla elements, build each component's DOM once, cache the node
  references, and afterwards only set `textContent` or attributes. Never reassign `innerHTML`
  on data changes. Charts update an SVG by setting `d` / `points` on cached elements.
  **Decision:** the huawei panels first used a `mount(node)` that replaces the whole body on
  every poll. That works at 10 s intervals but loses focus, scroll position and text selection
  in inputs. Prefer the cached-node style for anything that contains a form or updates faster
  than every few seconds. With Preact the virtual DOM does this for you, provided list items have
  stable `key`s.
- **Routing:** native `hashchange`, routes like `#/overview`, `#/charts?window=7d`. Parse the
  hash in a small pure function so it is testable. Phone-only tab bars (section 8) may be plain
  state instead of routes.

## 2. Day and night

The host sets `data-mode="day"` or `"night"` on `<html>` from `vessels.self.environment.mode`.
Day raises brightness and saturation to fight glare; night dims everything. The background
stays dark in both.

- Default to **day** when the path is absent, the request fails or the value is unknown, so a
  missing mode plugin never dims the screen by accident. Only the literal `night` selects night.
- Ship `<html lang="en" data-mode="day">` so the first paint is already themed.
- Source of the mode: a WebSocket subscription if the page has a socket anyway (ais-statistics),
  otherwise REST `GET /signalk/v1/api/vessels/self/environment/mode` every 30 s (huawei).

```css
:root {
  color-scheme: dark;
  --bg-base: #080a0c;          /* constant in both modes; also the theme-color meta */
  --bg-panel: #111414;
  --bg-panel-muted: #0a0c0c;
  --font-ui: system-ui, -apple-system, sans-serif;
  --font-data: ui-monospace, 'Fira Code', monospace;
}
:root, :root[data-mode='day'] {
  --color-green: #8dfcbb;  --color-green-rgb: 141, 252, 187;
  --color-teal:  #66c6db;  --color-teal-rgb: 102, 198, 219;
  --color-orange:#fca847;  --color-orange-rgb: 252, 168, 71;
  --color-red:   #ff5e5e;  --color-red-rgb: 255, 94, 94;
  --color-grey:  #666666;  --color-grey-rgb: 102, 102, 102;
  --text-main: #ffffff;    --text-muted: #a0a0b5;
}
:root[data-mode='night'] {
  --color-green: #4a7555;  --color-green-rgb: 74, 117, 85;
  --color-teal:  #33616b;  --color-teal-rgb: 51, 97, 107;
  --color-orange:#8a5318;  --color-orange-rgb: 138, 83, 24;
  --color-red:   #8f3333;  --color-red-rgb: 143, 51, 51;
  --color-grey:  #5a5a5a;  --color-grey-rgb: 90, 90, 90;
  --text-main: #c4c4c4;    --text-muted: #858596;
}
```

Night is meant to be dim, but not unreadable. Night grey is `#5a5a5a` (about 2.7:1 on
`--bg-panel`, in line with night teal and red) and night muted text `#858596` (about 5.1:1). The
earlier `#333333` and `#666677` were only 1.5:1 and 3.3:1, which made stale panels, whose label
is grey, nearly invisible exactly when they matter.

**Decision: define an `-rgb` triplet next to every colour.** Tints and borders need
`rgba(var(--color-green-rgb), 0.16)`, and CSS cannot split a hex variable. Both plugins do this.

**Decision: `--bg-base` is `#080a0c`.** Huawei used pure `#000000` to match the Status Tiles
display; ais-statistics used the guideline value. Use `#080a0c` and put the same value in the
`theme-color` meta so the phone browser chrome blends in.

A working example page with display and input elements in this style is in
[](example/index.html) (add  and  to its address;
screenshots in ).

A swatch sheet of every color below, with foreground and background pairs and where each is
used, is in [`color-palette.html`](color-palette.html) next to this file.

Every colour a page uses, including chart series and map overlays, must be one of these
variables (or derived from them) so night mode dims it too. ais-statistics maps class A to teal,
class B to orange, other to green, and p99 / p95 / median / average to teal / orange / green /
grey.

## 3. Panels and state

Flat. `border-radius: 0` everywhere (`* { border-radius: 0 }` is fine), no shadows, no
gradients. Panels are defined by a faint border, corner brackets that look like hardware
mounting, and an ultra-faint tint of the state colour.

- Panel state classes: `.theme-green`, `.theme-teal`, `.theme-orange`, `.theme-red`,
  `.theme-offline`. Each only sets a local `--theme-color` (and `--theme-color-rgb`); everything
  else (border, bracket, label colour, tint) derives from it.
- Border `1px solid rgba(var(--theme-color-rgb), 0.3)`, tint `rgba(var(--theme-color-rgb), 0.05)`,
  brackets via `::before` / `::after` on two opposite corners, 2px.
- **Decision (huawei): for wall-display legibility use a 4px border at 0.65 alpha, a 0.16 tint
  and 4px brackets, with sizes in `clamp()`.** This is the Status Tiles proportion. Use it for
  pages meant to be glanced at from across the cockpit; use the thin guideline version for dense
  nav-station pages.
- **Missing, stale or unreachable data is neutral grey, never green.** A panel only shows a
  state colour when the value is fresh and the source is healthy. Huawei goes neutral and shows
  a dash when the router link is not ok or the data is older than 60 s. Green on stale data is
  the worst failure a status display can have.
- A red panel may pulse border and tint (about 1.6 s). Always wrap it in
  `@media (prefers-reduced-motion: reduce) { animation: none }`.
- **Decision: chrome is subordinate.** A top band with the title, an "updated Ns ago" age, a link
  indicator (dot plus text; a pulsing red dot when the source is lost) and a clock, in muted,
  uppercase, tracked text with a hairline bottom border. The data panels get the colour, the
  chrome does not.

## 4. Typography

- Base UI `var(--font-ui)`; telemetry and numbers `var(--font-data)` with
  `font-variant-numeric: tabular-nums`.
- Labels and headings: uppercase, about `0.85rem`, bold, `letter-spacing: 0.1em`, coloured with
  `--theme-color` (muted when the panel is neutral).
- Payload values: large (2.5rem class), bold, `--text-main`. Use `clamp()` so they grow on a wall
  display and shrink on a phone, e.g. `clamp(1.7rem, 4.2vh, 3.2rem)`. Units next to a value go in
  a smaller muted `<small>`.

## 5. Live data

- **One WebSocket** to `ws(s)://<host>/signalk/v1/stream?subscribe=none`, then send a single
  subscription for the paths you need on `context: "vessels.self"`.
- **Always throttle.** Give every subscription `period` and `minPeriod` (ais-statistics: 5000 for
  mode, timezone offset and internet state). Only autopilot or steering views justify a high rate.
- **Reconnect with exponential backoff:** 1 s doubling to a 60 s cap with up to 20 % jitter so
  many clients do not return at once. Reset the attempt counter on `open`. Keep the function
  pure (`backoffDelay(attempt, random)`) and unit-test it.
- **Show the outage.** While disconnected, show a `.theme-offline` banner (with the seconds until
  the next attempt) and neutralise the data, as in section 3.
- Statistics that come from a plugin REST API instead of the stream: poll (10 to 60 s) and
  **pause while `document.hidden`**. Huawei polls status every 10 s, SMS every 30 s.
- Parse deltas defensively (`try/catch` around `JSON.parse`; ignore anything that is not an
  `updates[].values[]` shape).

## 6. Units

- **Publish SI, display in the user's units.** The plugin publishes metres, square metres,
  kelvin, radians, m/s, and sets default metadata with `app.setDefaultMetadata(path, { units,
  displayUnits: { category } })`. Do not publish nautical miles or km and then rename paths.
  **Decision (ais-statistics):** this meant renaming published paths from `...Nm` to SI before
  any release. Do it before the first release if you can; afterwards it breaks history.
- The webapp does not hardcode units. Fetch the path's meta (`/signalk/v1/api/vessels/self/<path
  with slashes>/meta`) and use `meta.displayUnits`: `formula` (such as `value * 0.000539957`) and
  `symbol`.
- **Never `eval` or `new Function` a server-supplied formula.** Use a small recursive-descent
  parser that accepts only numbers, `value`, `+ - * / ^` and parentheses, and returns
  `undefined` when it does not parse (then fall back to SI).
- **No display units, or an identity formula (`value`):** show SI with ISO prefixes: `1200 W` as
  `1.2 kW`, `15000 Wh` as `15 kWh`, `12300 m` as `12.3 km`, `4.1e6 m²` as `4.1 km²`. Round to
  about three significant figures, no grouping, no trailing zeros.
- **Decision (huawei): data volumes use decimal units** (kB, MB, GB at 1000), because routers
  and carriers count that way and the number must match the carrier's bill.
- No value: show `-` or `no data`, never `0` or `NaN`.

## 7. Time and dates

- A displayed time is either **ship local time with no zone text** (`14:30`) or **UTC with a
  trailing `Z`** (`04:30Z`). Never `LST`, `GMT+2`, `CEST`.
- Local is only possible when `environment.time.timezoneOffset` is published (a plugin such as
  signalk-ships-time does that). It is `(-)hhmm` encoded: `200` is +2 h, `-930` is -9 h 30.
  Convert to minutes with `sign * (floor(|n| / 100) * 60 + |n| % 100)`; accept a number or a
  numeric string.
- Without that path, fall back to UTC with `Z`. Do not use the browser's own timezone: the
  phone may be in a different place than the boat.
- Dates are `YYYY-MM-DD`. Shift the epoch by the offset and read the `getUTC*` fields so
  results do not depend on the viewer's zone. Keep these functions pure and test them.

## 8. Layout and touch

- **Phone is on-watch mode:** single column, targets at least **48 x 48 px** (wet hands, a moving
  boat). **Desktop is nav-station mode:** multi-column grids, dense layouts, map and data side by
  side.
- Use `clamp()`, `grid-template-columns: repeat(auto-fit, minmax(...))` and flex wrapping
  rather than only fixed breakpoints.
- **Decision (huawei): below 900 px show a tab bar (Status / SMS) that shows one group at a time;
  from 900 px up show everything.** Put an unread count on the tab. Cap long lists at about 60 %
  of the viewport height so the last item is not cut off.
- **Maps and canvases** sit inside a framed card. Controls float over the map as absolutely
  positioned, semi-transparent dark boxes (`rgba(17, 20, 20, 0.8)`) with sharp 1px borders;
  restyle the map library's own controls this way. Desktop height `calc(100vh - 140px)`, phone
  `min-height: 50vh` so the page still scrolls past it.

## 9. Forms and controls

- Strip browser styling (`appearance: none`). Inputs: transparent background, monospace text,
  2px `--color-grey` bottom border that turns to `--theme-color` on `:focus`.
- Buttons: transparent, 1px `--theme-color` border, uppercase monospace; on hover or active
  invert (background `--theme-color`, text `--bg-base`). Toggles are sharp rectangles or bracketed
  `[ ON ] / [ OFF ]`.
- Visible focus ring on everything (`outline: 2px solid var(--color-teal)`); never remove it.
- **Numbers:** prefer `type="text"` with `inputmode="numeric"` (integers) or `"decimal"`, and
  parse and validate in JavaScript. Signed fields (latitude, longitude, dBm) are the exception:
  iOS's compact keypads have no minus key, so use `type="number"` with no `inputmode`, or an
  unsigned field with an `N/S`, `E/W` toggle.
- **Decision (huawei):** `confirm()` before destructive actions (plan reset, SMS delete). Write
  controls are hidden after the first 403, so a read-only user (Signal K security on) does not
  see buttons that cannot work.

## 10. Console logs

For history, connection events and diagnostics, use an auto-scrolling pseudo-console: a 3-column
grid `timestamp | message | status`, all monospace; timestamps muted, messages `--text-main`,
right-aligned bracketed status (`[ OK ]`, `[ WARN ]`, `[ FAIL ]`) in the semantic colours. It is
a **circular buffer**: cap it (about 50 lines) and remove the oldest child when appending.

## 11. Branding and offline

- The favicon is the file named by `signalk.appIcon` in `package.json`, referenced relative to the
  webapp (`<link rel="icon" href="icon.png">`). One source of truth: do not copy it under another
  name. Extra sizes (`favicon.ico`, `apple-touch-icon.png`) are fine as additions.
  **Check:** the huawei page links `icon.svg` while `appIcon` is `icon.png`, which is the drift to avoid.
- `<meta name="theme-color" content="#080a0c">` matching `--bg-base`. One static value, because
  the background is dark in both modes.
- `<meta name="viewport" content="width=device-width, initial-scale=1">`, and a `<noscript>`
  line.
- **Decision (ais-statistics): remote tiles are optional.** Default to OSM / OpenSeaMap only when
  the browser is online and `network.internet.state` is not `metered` or `offline`; otherwise use
  the server's own chart provider (resources API `charts`). Remember a manual choice per browser,
  and grey out the server-chart button with a short note when the server has no charts.

## 12. Verify before calling it done

- **Unit-test the pure helpers** in a test runner: formula parser and unit conversion, ISO prefix
  formatting, time offset and date formatting, hash route parsing, backoff schedule, state
  thresholds (neutral on stale or missing).
- **Headless screenshots** in day and night, at desktop and phone width, plus an offline run with
  the WebSocket refused so the banner shows. Read the images; a passing build does not show
  glare or clipping. Do not claim a layout is checked on a phone unless it was looked at (huawei's
  phone layout was verified only by lint, typecheck and tests at the time of writing).
- Run it against a development SignalK server, not the boat's live one, and do not copy hostnames,
  addresses or credentials from a running server into fixtures or docs.

## 13. Plugin icons

Every plugin's `signalk.appIcon` follows one house style so the apps are recognisable side by
side in the SignalK app list. The style below was distilled from a ChatGPT image session that
produced about fifteen icons for BoatHacks plugins (radar statistics, music player, checklist,
VHF alarm, dead man's switch, ship's bell, inventory, package courier, boat logbook).

### Style

- **Palette: four flat colours.** Navy blue background (about `#002058`), gold for the main
  subject (about `#FDBB15`), plus one or two accents only where the subject needs them (brown for
  a package, white for a hat or highlight). Say "4 colors only" in the prompt and count them in
  the result.
- **Flat.** No shading, no gradients, no shadows, no highlights, no texture. If the first result
  has shading, ask for "flat colors only" and keep the pose.
- **Square, edge to edge.** The navy fills the whole canvas. No rounded corners (the system or
  the app store rounds them), no outer frame or ring unless it carries meaning.
- **No text.** No letters, numbers or labels in the picture. The one exception is a clock face,
  which may show a time (the bell icon uses 11:42). Ask for "remove any letters or text" if some
  sneak in.
- **One subject, drawn large.** A single mascot or emblem that fills most of the frame. If it is
  small in the frame, ask to zoom in on the part that carries the meaning (the squirrel's head,
  a bigger checklist).
- **Simple shapes, few details.** Thick uniform outlines and solid fills. Strip small hardware
  (no screws, no skull and crossbones on a lever, no fine ropework). A detail that is not
  readable at 72 px should not be there.
- **Readable at favicon size.** Judge every candidate scaled down to 72 px and 32 px, not at full
  size.

### Subject

- **A metaphor from the plugin's job, not a screenshot of its UI.** Examples that worked:
  radar sweep with dots and bars for AIS statistics; ship's bell plus a clock for a watch-bell
  plugin; magnifying glass over a storage shelf for inventory; a small skeleton pulling a wall
  lever for a dead man's switch.
- **The golden squirrel is the recurring mascot** for the BoatHacks plugins that want a
  character. It wears the accessory that matches the job: sailor or captain's hat (white),
  headphones and music notes (jukebox), a checklist, a package, a VHF handset beside an alarm
  siren. A plugin without a character uses a plain emblem in the same palette instead.
- **Check the animal.** Say "make sure it looks like a squirrel" and look for the bushy tail,
  ear tufts and upright pose before accepting it.
- **Relation to the webapp theme.** The icon navy is a brand colour for the icon only. It does
  not replace the console palette in section 2.

### Format and delivery

- Final file: **512 x 512 PNG, square, no transparency needed**, saved as the file named by
  `signalk.appIcon` (usually `public/icon.png`). The favicon uses the same file (section 11).
- Derive smaller sizes yourself (`favicon.ico` 16/32/48, `apple-touch-icon.png` 180) by scaling
  the 512 px master with PIL or ImageMagick. Do not ask the image model for each size; it does
  not return exact pixel sizes reliably.
- Keep the editable source (the prompt, and the SVG if one was made) next to the icon, for
  example `docs/icon-prompt.md`, so the style can be reproduced.

### Working with the image model

- **Start from a saved reference.** For a new icon in the same family, begin with "new icon, same
  color scheme and style, for <plugin>: <subject>" and attach or point at an existing icon.
- **One change per message.** Good iterations from the session: "remove the outer ring",
  "make the hat white", "zoom in on the squirrel's head", "make the checklist bigger and the
  squirrel sit on top of it", "put a decorative knot at the end of the bellrope",
  "remove the rounded corners, make it a square 512x512 pixel png". Several changes at once drift
  the pose.
- **Keep what works explicitly:** "like the pose, but use less shading", "same elements, frontal
  view", "remove all old components other than the squirrel and the notes".
- **Save, then restart.** When a direction is right, save it before asking for a new variant,
  and start a fresh prompt for a new subject so the old composition does not leak into it.
- **Mind the quota.** The free tier of the tool in the session ran out repeatedly (resets of
  about 24 h), which stalled iteration. Collect the changes you want and send them together, or
  use a paid tier or an API for a batch.

### Prompt template

```
New app icon, flat vector style, for a SignalK plugin called "<plugin name>", which <one
sentence on what it does>.

Subject: <one subject, one pose>. A golden squirrel <doing X>, wearing <accessory>.
Alternatively, for plugins without a mascot: <emblem built from 1 to 3 shapes>.

Style: 4 colors only: navy blue background (#002058), gold (#FDBB15)<, brown for X><, white for
Y>. Flat colors, no shading, no gradients, no shadows, no outlines of different colors, thick
simple shapes. No text, no letters, no numbers.

Composition: square canvas, background fills the whole square, no rounded corners, no border.
The subject is centered and fills about 70 percent of the frame. It must stay readable when
scaled down to 72 x 72 pixels, so leave out small details.

Output: 1024 x 1024 PNG.
```

### Example prompt

```
New app icon, flat vector style, for a SignalK plugin called "signalk-stowage-mgmt", which keeps
an inventory of what is stored where on the boat and helps find it again.

Subject: a golden squirrel in a white sailor hat, sitting on a storage shelf and holding a brown
package in both paws. A magnifying glass in the lower right shows the shelf section in front of
the squirrel.

Style: 4 colors only: navy blue background (#002058), gold (#FDBB15), brown for the package,
white for the hat. Flat colors, no shading, no gradients, no shadows, thick simple shapes. No
text, no letters, no numbers.

Composition: square canvas, background fills the whole square, no rounded corners, no border.
The squirrel is centered and fills about 70 percent of the frame. It must stay readable when
scaled down to 72 x 72 pixels, so leave out small details. Make sure it looks like a squirrel
(bushy tail, ear tufts).

Output: 1024 x 1024 PNG.
```

Follow-ups that fit this icon, one per message: "remove the light source in the top left
corner", "move the magnifying glass towards the viewer so it looks like we look through the glass
at the lower right of the shelf", "remove any letters or text".

### Icon checklist

- [ ] Four flat colours, navy background, gold subject, accents named in the prompt
- [ ] No shading, no gradients, no text, no rounded corners, square canvas
- [ ] One large subject, a real squirrel if a mascot is used
- [ ] Still readable at 72 px and 32 px
- [ ] 512 x 512 PNG saved as the `appIcon` file, smaller sizes scaled from it
- [ ] Prompt kept in the repo for reproducibility

## Checklist

- [ ] Buildless; one UI approach; runtime deps vendored under `public/vendor/` with licences
- [ ] Browser floor decided (Chromium 69 for MFDs), `clamp()` has a plain fallback if so
- [ ] Nothing loaded from the internet
- [ ] DOM built once, updated via `textContent` / attributes
- [ ] `data-mode` from `environment.mode`, default day; all colours from the variables
- [ ] Flat, brackets, `.theme-*` classes; stale or missing is neutral, never green
- [ ] One WebSocket, `period` / `minPeriod` set, backoff, offline banner; polls pause when hidden
- [ ] SI published, `meta.displayUnits` applied through a safe parser, ISO prefixes otherwise
- [ ] Times local without a zone or UTC with `Z`; dates `YYYY-MM-DD`
- [ ] 48 px targets, single column on phone, grid on desktop, `prefers-reduced-motion` honoured
- [ ] Favicon is the `appIcon` file; `theme-color` is `#080a0c`
- [ ] App icon follows section 13 (four flat colours, 512 px square PNG)
- [ ] Screenshots read in day, night, phone, desktop and offline
