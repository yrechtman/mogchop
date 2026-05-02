# MOG/CHOP — context for future Claude sessions

A static PWA at https://mogchop.vercel.app that runs face-mesh morphometry in
the browser and overlays a forensic-looking dossier on the captured photo.
The bit is that we treat trivial proportions with absolute clinical seriousness.

If you're starting cold, read this whole file before changing anything — the
gotchas section will save you a debugging round.

## Stack

- Static HTML, no build step. React via CDN, Babel-standalone in-browser.
- MediaPipe FaceLandmarker (`@mediapipe/tasks-vision`), runs on-device.
- Single Vercel serverless function: `api/analyze.js` — SSE-streams a
  Claude Sonnet 4.6 vision-grade qualitative layer.
- Supabase (Postgres) for the global corpus + `/map.html` heatmap.
- localStorage for per-device dossier history; coordinates never go there.
- PWA service worker (`sw.js`).

## File layout

| Path | What |
| --- | --- |
| `index.html` | The whole app — React tree, helpers, MediaPipe loader, dossier renderer, Supabase submit, telemetry status, phrenology FacultyPanel. ~1880 lines, single Babel script tag. |
| `map.html` | Standalone heatmap page. Leaflet 1.9 + CARTO dark tiles + circle markers. **Not** leaflet.heat — that was replaced in v1.7 because the radial-fade gradient lied (made every cell read red on its outer ring). Hits the Supabase RPCs directly. Linked as `/map`. |
| `api/analyze.js` | Vercel serverless function. Streams Claude Sonnet 4.6 with the morphometric report context. Uses prompt caching on the system block. |
| `sw.js` | Service worker, network-first for HTML, cache-first for static. Force-reloads claimed tabs on activate so SW upgrades are one-shot. |
| `vercel.json` | Functions config, `/map → /map.html` rewrite, SW cache headers, manifest content-type. |
| `manifest.webmanifest` | PWA manifest. |
| `icons/` | App icons. |
| `.claude/launch.json` | Preview server config: `npx serve -l 5173 .` |

## How to run / deploy

- **Local dev**: `npx serve -l 5173 .` (the preview server in `.claude/launch.json`).
  Or `vercel dev` if you need to exercise `api/analyze.js` (needs `ANTHROPIC_API_KEY` in env).
- **Deploy**: `git push origin main`. Vercel auto-deploys.
- **No PRs, no branches.** Per the user's global rule: commit and push directly to main.

## Supabase

- **Project**: `mogchop` (`kqesemrxxymodmydgapf`) in org `onrbxitpfajzsduvghoq`, region `us-east-1`.
- **URL**: `https://kqesemrxxymodmydgapf.supabase.co`
- **Anon key (publishable)**: `sb_publishable_lJLA1HvzC5vGktLNW4cb2Q_1lLTKng7` — embedded in
  `index.html` and `map.html`. Not a secret; RLS is the actual control plane.

### Schema (`scans` table)

```
id            uuid pk
scan_id       text                 -- the CAE-XXX-XXX code shown in the dossier
ts            timestamptz default now()
verdict       text check in ('MOG','MID','CHOP')
tier          text                 -- e.g. 'CERTIFIED MOGGER'
overall       numeric(5,2)         -- weighted morphometric score
percentile    numeric(5,2)
sigma_drift   numeric(6,3)
lat, lng      numeric(6,3) range-checked  -- ~110m precision
user_agent    text
```

Indexes on `(lat, lng)`, `verdict`, `ts desc`. RLS enabled.

### Policies + RPCs

- `anon_insert` allows anonymous INSERTs only. No SELECT policy → anon reads
  return `[]` (200, empty body). Don't add a SELECT policy unless you mean to
  expose individual records.
- `public.heatmap(bucket_prec int default 2)` — SECURITY DEFINER, returns
  bucketed `(lat, lng, total, mog/mid/chop_count, avg_overall)`. Granted to anon.
- `public.summary()` — SECURITY DEFINER, returns global counts + avg_overall.
  Granted to anon.

The map page calls these RPCs at `bucket_prec: 1` (~11km) so even the public
read view is coarsened beyond the 110m source precision.

### Privacy invariants (don't break these)

1. Coordinates round to 3 decimals on the client BEFORE submission. The
   `numeric(6,3)` column also enforces this server-side.
2. The image is never sent to Supabase. Only the verdict + score + coords.
3. RLS prevents reading individual rows. Heatmap is via aggregated RPC only.
4. **Submission is automatic, geolocation is opt-in.** Every scan auto-archives
   to the corpus when analysis finishes. Geolocation is requested on the BEGIN
   ANALYSIS / UPLOAD click (must be a user gesture for iOS to surface the
   prompt). If the user denies or the prompt times out, the row is still
   inserted with `lat`/`lng` null. The intake page discloses this behavior in
   the procedure list.

## App architecture

### Stages (App component, `index.html`)

`intake → camera | upload → confirm → scan → result → history`

State is held in App. Headers across all screens dispatch `cae:home` when the
logo is clicked; App listens and resets transient state to land on intake.

### Scan pipeline (`handleAnalyze`)

1. `getLandmarker()` lazy-loads MediaPipe (cached after first call).
2. Downsize the captured image to 1024px max dimension.
3. `landmarker.detect(canvas)` → 468 normalized landmarks.
4. `computeMetrics(lm, w, h)` → 12 indices, scored against μ/σ, weighted overall.
5. `verdictFor(overall)` → tier (MYTHICAL MOGGER → PERMACHOPPED).
6. `renderAnnotatedDossier(...)` → 1100px JPEG with the full overlay burned in.
7. POST to `/api/analyze` — SSE stream, sectioned plaintext (`[+] FACULTY READINGS` etc).
8. `awaitGeo(4000)` then `submitScan(...)` — auto-archive to Supabase (see Auto-archive flow below).
9. Save to localStorage history (annotated dossier + telemetry state stripped).

### Score calibration (current state — v1.11)

The instrument's punchline is the verdict tier. Most specimens should land in
MID; MOG and CHOP are reserved for genuine outliers in either direction.

```
score(v, μ, σ) = clamp(100 - 28 * |z|, 5, 98)        where z = (v − μ) / σ
weighted overall = Σ(score_i * weight_i) / Σ(weight_i)
```

Verdict bands (v1.11):

| Range  | Verdict           | side   |
| ------ | ----------------- | ------ |
| ≥ 92   | MYTHICAL MOGGER   | mog    |
| 84-91  | CERTIFIED MOGGER  | mog    |
| 76-83  | MARGINAL MOGGER   | mog    |
| **50-75** | **MORPHOLOGICAL MID** | **mid** |
| 36-49  | MARGINAL CHOP     | chop   |
| 24-35  | HARD CHOP         | chop   |
| < 24   | PERMACHOPPED      | chop   |

Three σ values are intentionally widened relative to the strictest possible
canonical envelope so harshly-tight indices don't dominate the overall:
`bgBz` σ=0.10 (was 0.07), `mandible` σ=10° (was 8°), `brow` σ=0.018 (was
0.012). Keep these. The slope (28) is original; the σ widening + the wide
MID band are what land typical faces in MID without burying chad-filtered
specimens.

Reference data point: a Pexels stock photo of a young man with somewhat-sharp
features scores 67 → MID. Recompute against this if you change the curve.

### Auto-archive flow (geolocation + corpus submit)

Every scan auto-submits to Supabase when analysis finishes. The user does not
click anything to opt in — opt-out is a denied geolocation prompt (the row is
still inserted, just with `lat`/`lng` null).

State on App:
- `geoRef.current` — `null` (pending), `{ lat, lng }` (granted), or `'denied'`.
- `geoRequestedRef.current` — bool, ensures `getCurrentPosition` fires once.

Trigger order:
1. On app mount: `navigator.permissions.query({ name: 'geolocation' })`. If
   `granted`, fetch silently — no prompt. If `denied`, set ref to `'denied'`.
2. On BEGIN ANALYSIS / UPLOAD button click: `ensureGeo()` fires the prompt.
   Must be in a user-gesture handler so iOS surfaces it.
3. End of `handleAnalyze`: `awaitGeo(4000)` waits up to 4s for an in-flight
   request, then `submitScan(...)` regardless of outcome.

Result panel renders `<TelemetryStatus>` showing one of:
`QUEUED → UPLINKING → ARCHIVED · GEO | ARCHIVED · ANON | FAULT`. FAULT exposes
a RETRY link wired to `retryTelemetry`. History records have `telemetry`
stripped, so re-opened scans don't show the panel at all.

### Phrenology layer (`FacultyPanel`)

`[+] FACULTY READINGS` is the second-funniest part after the verdict, so it
gets a dedicated dossier panel positioned right after the specimen image —
ahead of the qualitative observations panel.

Each line from the model is parsed by `parseFaculty(line)`:
```
"- CAUSALITY: 64 — frontal occupies 0.34 of facial height."
        ↓
{ name: "CAUSALITY", score: 64, note: "frontal occupies …",
  category: "NOMINAL", color: "#FFEC2A" }
```

Categorical ramp:

| Score   | Category    | Color (hex)  |
| ------- | ----------- | ------------ |
| ≥ 85    | PRONOUNCED  | `#2ECF6B`    |
| 70-84   | DEVELOPED   | `#9DD53A`    |
| 50-69   | NOMINAL     | `#FFEC2A`    |
| 30-49   | DEPRESSED   | `#FF8C2A`    |
| < 30    | VESTIGIAL   | `#FF2A2A`    |

UI structure: 2-column grid of categorical cards on top (faculty name +
big DEVELOPED/NOMINAL/etc. label + score), a `// JUSTIFICATIONS` section
with per-line clinical notes below.

Faculty name list is fixed in the system prompt (CAUSALITY, COMBATIVENESS,
ACQUISITIVENESS, IDEALITY, VENERATION, SELF-ESTEEM, AMATIVENESS, MIRTHFULNESS,
CONCENTRATIVENESS, FIRMNESS, CAUTIOUSNESS, ORDER, LANGUAGE, COMPARISON). The
prompt says NEVER fabricate one outside the list. If the model invents
something, the parser will still pick it up (regex matches any all-caps name)
— but that's a model failure to flag, not a feature.

### Dossier renderer (`renderAnnotatedDossier`)

Single big function in `index.html`. Burns onto the photo:

- 468-pt mesh as faint red dots
- Subject midline, vertical fifths grid, horizontal thirds rule
- Bizygomatic / bigonial / inter-canthal / mouth / nose calipers (ticked ends)
- Canthal tilt: line + horizontal baseline + arc + signed degree label
- Mandibular angle arcs at each gonion
- Eye boxes (frontal aperture zones), brow segments, naso-facial axis
- Glyph crosshairs: α forehead, β chin, γ nose tip, δ subnasale,
  ε canthi, ζ zygomatic, η gonion
- Top + bottom telemetry strips with all 12 metric values + verdict stamp
- Diagonal `CAE / CFM-12 / DO NOT REPRODUCE` watermark
- Corner registration brackets

Two helpers matter:
- `line(a, b)` — haloed (dark outline + colored stroke). Use for primary
  construction.
- `lineFaint(a, b)` — plain stroke. Use for the background grid only.

Same haloing applies to `arc()` and `circle()`.

The values are NOT shown in inline chips (they collide with each other on
real faces). Only canthal tilt and mandibular angle have inline labels because
they label the arcs. Everything else lives in the bottom telemetry strip.

### Qualitative layer (`api/analyze.js`)

Claude Sonnet 4.6 with prompt caching on the system block. Output format is
strict sectioned plaintext:

```
[+] FACULTY READINGS    ← Victorian phrenology, deadpan; streams FIRST
[+] OBSERVATIONS
[+] PROFILE INFERENCE
[+] STRENGTHS
[+] WEAKNESSES
[+] FINAL REMARK
```

Section order matches the visual layout on the result screen, top-down by
importance: phrenology cards sit immediately under the specimen image, so
they need to stream in first or the page loads "wrong" (text below populates
before the hero panel).

Streamed to the client via SSE deltas; client parses incrementally and renders
each section as it fills in. The system prompt explicitly bars age/gender/race
commentary, slurs, and dating-market language. Tone is even-handed clinical
observation — the verdict tier carries the punchline.

The phrenology section maps facial morphometrics to Victorian "faculties"
(CAUSALITY, COMBATIVENESS, ACQUISITIVENESS, IDEALITY, VENERATION, SELF-ESTEEM,
AMATIVENESS, MIRTHFULNESS, CONCENTRATIVENESS, FIRMNESS, CAUTIOUSNESS, ORDER,
LANGUAGE, COMPARISON). The bit is the absurd specificity of treating bunk with
the same rigor as the actual metrics. Faculties are derived from morphometric
values ONLY — never from race, gender, age, expression, or anything outside
the geometric mesh. The model is instructed not to fabricate faculties outside
the mapping list.

If you add or rename a section, update BOTH parsers (server-side
`parseSections` in `api/analyze.js` and client-side `parseLiveSections` in
`index.html`) AND the commentary state initializer in `handleAnalyze`'s
`baseRecord`.

## Important gotchas (read before debugging)

### 1. Babel preset

The Babel script tag in `index.html` MUST be `data-presets="react"` only,
NOT `"env,react"`. The `env` preset targets ES5 and rewrites `await import(...)`
into a `require()`-based dynamic loader, which throws "require is not defined"
in the browser. Modern browsers handle every ES feature this app uses natively.

### 2. MediaPipe ESM bundle

Import path is `vision_bundle.mjs`, NOT `+esm`:

```js
await import('https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.22-rc.20250304/vision_bundle.mjs')
```

The jsdelivr `+esm` auto-converted bundle has stray CommonJS internals that
fail in the browser.

### 3. Service worker upgrade strategy

`sw.js` is currently `mogchop-v4`. HTML requests are network-first (so deploys
land immediately). On `activate`, the new SW calls `skipWaiting`, `clients.claim`,
THEN `client.navigate(client.url)` for every controlled window — this forces
already-loaded tabs to reload through the new SW and pick up fresh HTML in one
shot. If you change SW behavior, bump the cache key (`mogchop-vN`) so old
installs evict.

### 4. Canthal tilt math

The right-eye direction is `outer.x < inner.x` in viewer-space, so naive
`atan2(outer.y - inner.y, outer.x - inner.x)` returns ~±180° for that side
and the average comes out as -88° on real faces. Use `tiltOf(inner, outer)`
which always measures rise over `|run|`:

```js
const tiltOf = (inner, outer) =>
  Math.atan2(inner.y - outer.y, Math.abs(outer.x - inner.x)) * 180 / Math.PI;
```

Positive = outer canthus above inner ("hunter eyes"), regardless of side.

### 5. Annotated dossier is not persisted

The 1100px overlay JPEG (~150KB) is computed at scan time and lives on the
React state, but it's stripped before saving to localStorage history. 50 scans
× 150KB = blown localStorage quota. If you want to show the full dossier for
historical scans, regenerate it from the stored landmarks (currently not stored;
you'd need to add `landmarks` to `historyRecord`).

### 6. `vercel.json` rewrite

`/map → /map.html` is configured in `vercel.json`. `npx serve` does extension
stripping by default so it works locally too. If you add another HTML page,
either link to `.html` directly or add another rewrite.

### 7. Supabase costs

The `mogchop` project is on the free tier. The user already has another inactive
project (`fit-check`) — free tier allows 2 active projects per org. Don't create
new projects without checking with the user.

## Visual / typography conventions

- Display font: Space Grotesk (700 weight)
- Mono: JetBrains Mono
- Colors: `--bg #050505`, `--fg #F5F5F5`, `--red #FF2A2A`, `--green #2ECF6B`,
  `--amber #FFEC2A`. The map page uses the same palette.
- Section markers: `// SECTION NAME` in dim caps
- Codes / IDs: `CAE-XXX-XXX` format
- Live indicator: small red dot + "LIVE" badge in the header

## Things explicitly left out

- No analytics, no tracking pixels, no third-party JS beyond CDN libs (Babel,
  React, MediaPipe, Leaflet, fonts).
- No auth. The app is anonymous-first by design.
- No /api/submit serverless function. Inserts go directly from the client to
  Supabase REST. Adding a function would only add latency and a code path to
  maintain; RLS is the right control point.

## Recent commits (for context)

- `v1.12` — stream phrenology first so the page populates top-down by importance
- `v1.11` — extend MID band upward (50-75); phrenology promoted to a hero panel with categorical cards
- `v1.10` — phrenology layer (Victorian faculty readings) added to the dossier
- `v1.9` — recalibrate so most specimens land in MID; revert v1.8 prose sharpening
- `v1.8` — sharper qualitative prose (later mostly reverted in v1.9)
- `v1.7` — heatmap shows hotness (per-cell avg) not density; switched leaflet.heat → circle markers
- `v1.6` — soften score curve (slope 28→24, σ widened on bgBz/mandible/brow) so chad-filtered specimens clear MOG; superseded by v1.9 calibration but the σ widening stays
- `v1.5` — auto-archive every scan to the corpus; geolocation prompt on capture click
- `v1.4` — Supabase corpus + /map heatmap; CLAUDE.md created
- `v1.3` — high-contrast overlay strokes (haloed lines) + clickable home logo
- `v1.2` — phrenology-grade overlay (lines/arcs/glyphs burned into the photo)
- `v1.1` — live camera, SSE-streamed verdict, full dossier renderer
- `v1.0` — initial PWA, MediaPipe + 12 indices + Claude vision layer
- earlier fixes: `vision_bundle.mjs` import, drop Babel `env` preset, network-first SW with force-reload-on-activate

If something looks wrong on the live site but the source looks right, suspect
the SW first (force a hard reload) before assuming the deploy didn't propagate.
