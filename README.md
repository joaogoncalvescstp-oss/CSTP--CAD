# CSTP CAD Viewer

A single-file, no-build, browser-based viewer for CAD/survey drawings. Open
`index.html` directly (or serve it statically) — there's no server, no
dependencies, and nothing to install.

## What it does

- **↥ Import DXF** — reads `LINE`, `LWPOLYLINE`, `POLYLINE`/`VERTEX`, `POINT`,
  `CIRCLE`, and `3DFACE` entities out of a plain ASCII DXF file. Each
  entity's own DXF layer becomes a layer in the viewer.
- **↥ Import LandXML (TIN)** — reads a LandXML `<Surface>` (TIN: `Pnts` +
  `Faces`) as a triangulated surface layer, and any `<CgPoint>` records as
  plotted survey/COGO points. Multiple imports layer onto whatever's already
  loaded — you can bring in a DXF and a TIN together.
- **⛰ Elevation** — click a visible, unlocked TIN surface to read its
  interpolated elevation at that exact point (barycentric interpolation
  across whichever triangle you clicked). This is the actual payoff of
  importing a surface model: getting a spot elevation off it without CAD.
- **🌊 Water Flow** animates droplets across every visible TIN surface,
  following real plane geometry rather than a stylistic guess: each droplet
  repeatedly looks up which triangle it's currently inside and moves along
  *that triangle's own* steepest-descent direction — computed from the
  triangle's own upward-facing plane normal `(nx,ny,nz)`, where the descent
  direction is exactly proportional to `(nx,ny)` (derivable directly from
  the plane equation, not approximated). A droplet respawns at a fresh
  random point on the surface once it flows off the modeled edge or lands
  on a flat/vertical facet with no defined downhill direction, and faster
  on steeper triangles than gentle ones. This is a **stylized visual
  approximation of surface runoff direction, not a hydrology/watershed
  simulation** — there's no infiltration, ponding capacity, channel
  concentration, or flow accumulation, just "which way does this exact spot
  drain." 2D plan view only, same reasoning as ⛰ Elevation/🧲 Snap.
- **↥ Import LandXML (TIN / Pipes)** also reads Civil3D **pipe networks**
  (`<PipeNetworks><PipeNetwork><Structs>`/`<Pipes>`) — manholes, catch
  basins, and cleanouts as structures; storm/sanitary pipes as lines between
  them. Each network becomes its own layer. Labels (🏷 Pipe Labels, each
  independently toggleable) show:
  - a structure's **rim elevation** and **depth** (Rim − Sump, or Rim minus
    its lowest invert if no sump elevation is given — the closest
    computable proxy this data has for a manhole's build/barrel height),
  - a pipe's **diameter** and **% slope** (straight from the file's own
    `slope` attribute),
  - a pipe's **invert elevations** at both ends — these live on the
    *structure* side of the connection in real LandXML (`<Invert elev
    flowDir refPipe>`, matched back to the pipe by name), not on the pipe
    itself, so the viewer resolves that lookup for you.
  - Labels are zoom-gated (manholes/pipes always draw; text only past a
    zoom threshold) and use a small collision-avoidance pass so two
    structures sitting close together (a sanitary and storm manhole at the
    same intersection is a completely normal real case) don't merge into
    unreadable overlapping text.
  - A real Civil3D export can reference a structure name that's never
    actually defined anywhere in the file (confirmed against a real
    project file — structures outside the export's own scope). A pipe
    like that has nowhere to draw to, so it's skipped and counted rather
    than silently vanishing or inventing a location — check the ⚠ badge
    next to a pipe-network layer's entity count in the Layers panel for
    the count and the exact missing structure names.
- **📑 Layers** — every imported DXF layer, TIN surface, and pipe network
  gets its own row: a **👁 visibility** checkbox (show/hide), a **🔒 lock**
  checkbox (dims the layer and excludes it from the elevation query,
  without hiding it — the same distinction a real CAD lock makes), and a
  **🔍 zoom-to-layer** button. The last one matters for a real street/
  alignment corridor project — the whole site can be extremely elongated
  (e.g. 178ft wide by 5000+ft long isn't unusual), so fitting the WHOLE
  drawing at a uniform scale (the only geometrically honest way to fit
  anything) can leave every individual manhole far too small to read;
  zooming to one network's own extent is the practical way to actually
  work it.
- **↥ Import LandXML (TIN / Pipes / Alignments)** also reads horizontal
  **Alignments** (`<Alignments><Alignment><CoordGeom>` — `Line`, `Curve`,
  `Spiral`), a sibling **`<Profile><ProfAlign>`** (vertical curves), and
  **`<StaEquation>`** (station equations) — the one import format that
  actually carries real curve/chainage/profile data (a plain DXF
  `LINE`/`LWPOLYLINE` has no curvature or station of its own). Each
  alignment becomes its own layer (tangents as lines, curves sampled as a
  smooth arc).
- **📍 Label Points** labels an alignment's key geometry points across four
  categories, each with its station and elevation (a vertical/profile
  point's own design elevation, or the point's plain geometry Z otherwise —
  every category shows one on-canvas whenever the source data actually
  carries it, matching the 📋 Points Table's Elevation column):
  - **Horizontal curve** — PC, PT, PI (only when the file's own `<PI>` is
    given — never derived by trig), MID (arc midpoint), CC (the circle's own
    center — off the physical curve, at radius distance), PCC/PRC (compound/
    reverse curve, where two curves meet with no tangent between).
  - **Spiral transition** — TS/SC/CS/ST (the exact junction coordinates
    between a line/curve and a spiral — these are file-given boundary
    points even though a spiral's own interior shape is drawn as a straight
    chord, see below), SPI (a spiral's own `<PI>`, when given), SS (spiral-
    to-spiral).
  - **Vertical / profile** — PVC/PVT/PVI (from each `<Profile>` grade break
    and its optional vertical curve length), PVCC/PVRC (two vertical curves
    meeting with no tangent between), and Hi/Lo (the curve's own turning
    point, only reported when it actually falls inside the curve). These are
    plotted **on this same plan-view canvas** — not a separate elevation/
    station graph — by looking up each one's station on the alignment's own
    horizontal geometry, so you see exactly where on the ground each
    vertical event sits, alongside its station and design elevation.
  - **Alignment reference** — POB/POE (the chain's own start/end) and EQN
    (a station equation's back/ahead numbers, plotted at its physical
    location).
  - **Custom** — type station(s) (e.g. `12+50, 18+00`) to drop a POT/POC/POS
    at an arbitrary point along the target alignment.

  The workflow matches how this is actually done in Civil3D: first pick the
  **reference alignment**, which defines station 0 and direction; then check
  one or more **targets** to label from a checklist — each can independently
  be an alignment (its own geometry gives every real point type above) or a
  plain DXF line/layer (which has no curve/profile data at all, so only its
  segment midpoints can honestly be labeled as MID — nothing else is ever
  invented without real geometry behind it). Checking several targets at
  once labels them all together against the same reference in a single
  pass — e.g. a mainline plus every side street off it. Most point types'
  displayed station is the closest station/offset projection onto the
  reference alignment as a whole (not just its nearest single segment), so
  checking the reference as its own target recovers native stationing; EQN,
  POT/POC/POS, and every vertical/profile point instead always keep the
  station they're actually defined at, since re-stationing a profile point
  or an authored station equation against an unrelated alignment wouldn't
  mean anything. CC never shows a station at all. Display is toggleable by
  category (plus an "all categories" convenience toggle) — 25 individual
  colors would be unreadable, so color/marker-shape denote the category at
  a glance while the printed 2-4 letter code always gives the exact point
  type.
- **📋 Points Table** lists every currently labeled point — type, category,
  which target it came from, station, northing, easting, and elevation
  (a vertical/profile point's own design elevation, or the point's plain
  geometry Z otherwise) — sorted by station, as a field stakeout reference
  sheet. It shows everything computed regardless of the on-canvas category
  toggles, and **⬇ Export CSV** saves the same table to a file for printing
  or loading into a data collector.
- **📈 Profile** (P) is a third view mode, alongside 2D plan and 3D orbit,
  for one alignment at a time: station on the X axis, elevation on Y, each
  with its **own independent scale** rather than the plan view's single
  shared one — a real profile is a strip a few tens of feet tall and
  thousands of feet long, so forcing equal axes would draw every grade as a
  nearly flat line. That deliberate mismatch is exactly what "vertical
  exaggeration" means on a real plan-and-profile sheet, and the current
  ratio is always shown on-screen so it's never silently misread as true
  slope. An alignment with an imported `<Profile>` shows its real design
  grade line plus PVC/PVT/PVI/PVCC/PVRC/Hi/Lo; one without falls back to a
  dashed line tracing its horizontal geometry's own elevation, clearly
  marked as not a real design profile. A light vertical tick at each
  horizontal PC/PT station cross-references the plan-view curves, matching
  how a real plan-and-profile sheet lines the two up. Drag to pan, wheel to
  zoom (both axes scale together, so zooming never changes the chosen
  exaggeration) — 2D plan-view-only tools (⛰ Elevation, 🧲 Snap, 🗺 Map) are
  unavailable while in profile view, same as in 3D orbit.
- Pan by dragging, zoom with the mouse wheel (zooms toward the cursor).

## 3D orbit view, aerial MAP background, and CAD object snap

Three subsystems ported over from this project's sibling app, FBK-Checker,
adapted to this viewer's DXF/TIN/pipe-network data model:

- **⟲ 3D** toggles an oblique orbit view of everything currently loaded (DXF
  entities, TIN wireframes, CgPoints, pipe networks — a structure's Z comes
  from its rim elevation, the only Z this data carries for it). Drag to
  rotate — the pivot re-centers on whatever's under the cursor at the start
  of each drag, so orbiting a large, elongated corridor project doesn't sweep
  a far corner across the screen for a tiny mouse move. Shift-drag or
  middle-mouse-drag pans, the wheel zooms about the cursor. ⊡ Fit and 🔍
  zoom-to-layer both work in 3D too. ⛰ Elevation and 🧲 Snap are 2D-only —
  there's no world-space inverse for this oblique projection to resolve a
  screen click back through, the same limitation the source app's own
  object-snap-driven tools have in its 3D view.
- **🗺 Map** overlays aerial imagery, tiled from Ramsey County MN's own
  ArcGIS ImageServer (falls back to Esri World Imagery, then USGS NAIP) over
  a calibrated Lambert-Conformal-Conic ↔ lat/lon ↔ Web Mercator transform.
  Like its source in FBK-Checker, **this is only geometrically correct for a
  job whose E/N are already Ramsey County's own survey coordinate system** —
  it's ported as-is, not generalized to an arbitrary CRS, since resolving
  "which projection is this file even in" for an arbitrary DXF/LandXML is a
  materially bigger, separate problem. 2D view only.
- **🧲 Snap** is a CAD object-snap readout: hover the canvas and it reports
  (in the hud, and with an on-screen marker) whichever of an endpoint, a
  genuine crossing of two real lines, a midpoint, a circle's center/quadrant,
  or the nearest point on a line is closest to the cursor, in that priority
  order — each toggleable from the floating checkbox panel. It reads
  E/N/Z off DXF LINE/LWPOLYLINE/POLYLINE vertices and CIRCLE
  centers/quadrants, LandXML CgPoints, and pipe network structures/pipes.
  Unlike FBK-Checker (which uses this to place a new point), this viewer has
  no point-insertion tool — Snap is a precision read-only aid for eyeballing
  exact coordinates off the drawing. 2D only, same reasoning as 3D above.

Verified end-to-end in a real headless browser: importing a synthetic DXF
with two crossing lines and a circle, then hovering the crossing correctly
resolves to `type:'intersection'` at the true crossing point and hovering
the circle's own center correctly resolves to `type:'center'` (not a plain
endpoint — DXF circle entities are excluded from the generic endpoint scan
specifically so their center isn't shadowed by itself); entering 3D and
dragging genuinely rotates (`orbit.az`/`orbit.el` change), middle-dragging
genuinely pans (`orbit.ox`/`orbit.oy` change), and wheel-zooming genuinely
zooms (`orbit.s` increases) — all confirmed against the live app state, not
a reimplementation; toggling Map correctly arms `showMap` and fires the
tiled fetch (the fetch itself can't be verified end-to-end from this
sandbox, whose own network policy blocks `maps.co.ramsey.mn.us` — the exact
same standing limitation FBK-Checker's own MAP section documents). The
pre-existing DXF/LandXML/pipe-network/elevation-query/layer-manager
regression suite was re-run afterward and passes unchanged.

## DWG — why it isn't supported directly

DWG is Autodesk's proprietary binary format. Reading it in a browser with no
bundled parser isn't practical: the open-source option (LibreDWG) is GPL-3.0
and still has real entity-coverage gaps, and a Civil3D **surface** or **COGO
point** object specifically is not a plain DWG entity at all — it's a
proprietary "object enabler" payload that even most third-party DWG readers
can't decode without Autodesk's own SDK for it.

Picking a `.dwg` file in the **↥ Import DWG** button doesn't silently fail or
pretend to work — it just tells you, in the status bar, to do one of these
instead (both are a couple of clicks in Civil3D/AutoCAD):

- **SAVEAS → DXF** for ordinary drawing linework, then **↥ Import DXF**.
- **Export → LandXML** for a Civil3D surface (and/or its COGO points), then
  **↥ Import LandXML (TIN)** — this is the actual bridge format Civil3D
  itself uses for exactly this kind of interop, and it's already a plain,
  regular XML structure — nothing proprietary to decode.

## Notes on the LandXML parser

- Surface points (`<P>`) are read in LandXML's own order, **Northing Easting
  Elevation** — not Easting/Northing.
- A face's point-id references (`<F>`) can carry a leading `-` marking a
  suppressed breakline edge; that's stripped before looking up the id, it's
  not a sign on the point number.
- A `<Surface>` with no faces that actually resolve to 3 real points is
  skipped rather than added as an empty layer.
- Same Northing/Easting/Elevation text order applies to an `<Alignment>`
  geometry element's `<Start>`/`<End>`/`<Center>` points.
- A `<Curve>`'s rotation direction comes from its own `rot="cw"|"ccw"`
  attribute when present, otherwise it's inferred from the cross product of
  the start/end vectors off the center — and a missing `length` is
  recomputed from the swept angle and radius rather than left blank, since
  both are always derivable from the point/radius data every `<Curve>`
  already carries.
- A `<Spiral>` is drawn as a straight chord between its `Start`/`End` (its
  true Euler-spiral interior shape isn't modeled) — but those two endpoints
  are exact, file-given coordinates regardless, so its TS/SC/CS/ST are
  still labeled correctly; only a point *inside* the spiral (a typed POS
  station) uses the same chord approximation for its position.
- A `<Curve>`/`<Spiral>`'s PI is only labeled when the file gives an
  explicit `<PI>` point — it's never derived by trigonometry, since that
  would mean guessing a tangent intersection the source data didn't
  actually provide.
- `<Profile><ProfAlign>` is read as a sequence of `<PVI>sta elev</PVI>`
  grade breaks, where any `*Curve length="...">` element (`ParaCurve`,
  `CircCurve`, `UnsymParaCurve`, …) immediately following a `<PVI>` in the
  file applies a vertical curve of that length to the PVI just before it —
  the ordering every real Civil3D export uses. The curve's length is always
  split symmetrically about its PVI (by station), and its shape is treated
  as a parabola regardless of the curve tag's own name (a true circular
  vertical curve is a near-identical-in-practice approximation of a
  parabola over these lengths, so this is a documented simplification, not
  an error).
- `<StaEquation staBack="..." staAhead="...">` is labeled at its physical
  break point (using the *back* station, the side that's still physically
  continuous). Stations elsewhere in the alignment are **not** renumbered
  past the equation — a station equation's whole point is to let numbering
  jump, and correctly re-deriving every downstream station through one (or
  several, compounding) equations is out of this feature's scope; the
  equation itself is always labeled correctly, just not propagated.

## Verified

Alignment import and 📍 Label Points were exercised end-to-end in a real
headless-browser session, in two passes:

- A simple tangent → 90°/200'-radius curve → tangent alignment: the curve's
  PC/PT/MID landed at their exact expected coordinates and stations
  (`11+00.00` / `14+14.16` / `12+57.08`), the curve itself rendered as a
  smooth sampled arc (not a straight chord), and a plain DXF line target
  correctly produced only a MID (no fabricated PC/PT) stationed by
  projecting onto the *nearest* element of the reference alignment as a
  whole — including correctly preferring a nearby curve segment over a
  closer-looking straight one once actual perpendicular/radial offset was
  compared.
- A second, deliberately elaborate alignment — line → curve → curve (same
  rotation) → curve (opposite rotation) → line → spiral → curve → spiral →
  line, plus a `<StaEquation>` and a `<Profile>` with a crest curve directly
  adjacent to a sag curve (zero tangent between them) — checked all 27
  computed values against independently hand- and trig-derived expected
  results: every junction type (PC/PT/PCC/PRC/TS/SC/CS/ST) landed at the
  exact shared coordinate between its two elements; PI/SPI passed through
  the file's own `<PI>` points unchanged; CC reported all 4 curve centers
  with no station; the station equation's back/ahead numbers and physical
  location matched; typed POT/POC/POS stations (including one requiring
  independent arc trigonometry to verify) matched exactly; and the profile
  math correctly produced PVI/PVC/PVT at their exact parabola-derived
  stations/elevations, found the crest's Hi point and the sag's Lo point
  only where the turning point actually fell inside each curve, and
  correctly detected the directly-adjacent crest/sag pair as a PVRC (not a
  PVCC, since a crest-then-sag is a reversal) rather than firing two
  separate, disconnected curves. The category display toggles (each of the
  5 individually, and the "all categories" master, in both directions) were
  also driven programmatically with zero console errors.

Multi-target selection and the 📋 Points Table were verified together in a
third headless-browser pass: importing two separate alignments ("Test2" and
"Main St") and checking both as targets against one reference in a single
📍 Label Points pass correctly labeled all of both alignments' points in
one combined result, each point correctly tagged with which target it came
from; the table opened with every expected column (including Northing/
Easting/Elevation), rendered exactly one row per computed point, and the
rows came back sorted by station ascending; and the CSV export's header and
row count matched the on-screen table exactly.

A fourth pass confirmed on-canvas labels show elevation for every category,
not just vertical/profile points: against a curve with distinct, non-zero
elevations at each end, PC/PT/MID/CC/POB/POE all printed the exact expected
`EL` value (the curve's own Start/End/Center Z, or the correct interpolated
average for MID), matching what the table's Elevation column already shows.

A fifth pass verified the 📈 Profile view end-to-end: toggling it on defaults
to the first imported alignment and shows the alignment picker; entering 3D
orbit correctly exits profile view (and vice versa); an alignment with a
`<Profile>` reproduced the exact same PVI/PVC/PVT/Hi/Lo/PVRC set already
verified for the plan-view labeling feature, now drawn as a real sampled
parabola on independent station/elevation axes; an alignment with no
`<Profile>` fell back to its horizontal geometry's own elevation, sampled
across 200 points and matching that geometry's true start/end elevations
exactly; a real mouse drag panned the view by exactly the pointer's pixel
delta; a real mouse-wheel zoom kept the cursor's underlying station and
elevation exactly fixed while growing both axis scales together (preserving
vertical exaggeration); and the close button, plus the `P` keyboard
shortcut, correctly toggled the view off and back on, restoring the legend
each time.

A sixth pass verified 🌊 Water Flow's math and lifecycle end-to-end against
a synthetic 10×10 unit ramp surface (two triangles, elevation running
exactly 10→0 across the E axis and constant across N — so the true downhill
direction is exactly `+E`, no other component): the steepest-descent
formula returned `(dirE:1, dirN:0)` on both triangles to within floating-
point precision; triangle containment correctly found the surface's center
and correctly rejected a point far outside it; toggling on spawned the
expected particle count all placed on the surface, and running the
simulation for a simulated 2 seconds produced a clear net population drift
in the `+E` direction with every particle remaining finite and carrying a
trail; entering 3D orbit or 📈 Profile view each correctly force-stopped the
animation and cleared its particles, attempting to toggle it on while
already in 3D correctly refused with an explanatory hud message instead of
silently starting an animation nothing would render, and manual toggling
on/off cleanly repeated with the animation frame handle and particle array
always left in the expected state.

Both parsers, the layer visibility/lock toggles, the elevation-query tool
(including its lock-exclusion behavior), and zoom-to-layer were exercised
end-to-end in a real headless-browser session against synthetic DXF/LandXML
files covering every supported entity type — zero console errors. The pipe
network parser, invert lookup, depth calculation, dummy-structure handling,
and undefined-structure-reference skipping were additionally verified
against a real Civil3D LandXML export (4 networks, 102 structures, 85
pipes) with independently-checked ground truth (exact rim/sump/invert
values, and the 19 structure names — confirmed by grepping the raw file —
that are referenced but never defined anywhere in it).
