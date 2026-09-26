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
- **⛰ Elevation** — click a visible, unlocked TIN surface to save its
  interpolated elevation at that exact point (barycentric interpolation
  across whichever triangle you clicked) — the actual payoff of importing a
  surface model: getting a spot elevation off it without CAD. The tool
  stays armed after each click, so a whole series of spots can be picked
  in one go instead of re-toggling the button every time; **📐 Elevation
  Spots** lists every spot saved this session (with CSV export), the most
  recently-picked one drawn full-strength on the canvas and earlier ones
  dimmed so the running history stays visible without crowding out the
  current pick. Picking now also **snaps first** — to the nearest TIN
  wireframe vertex or edge, DXF line/point, or pipe network structure, via
  the same engine 🧲 Snap already uses — so a spot lands exactly on a real
  break-line vertex instead of an approximate nearby guess, falling back to
  the raw click position when nothing snappable is in range. (🧲 Snap
  itself also gained TIN vertices/wireframe edges as snap targets from this
  same change, so it isn't only an elevation-picking benefit.)
- **🌊 Water Flow** animates droplets across every visible TIN surface,
  following real plane geometry rather than a stylistic guess: each droplet
  repeatedly looks up which triangle it's currently inside and moves along
  *that triangle's own* steepest-descent direction — computed from the
  triangle's own upward-facing plane normal `(nx,ny,nz)`, where the descent
  direction is exactly proportional to `(nx,ny)` (derivable directly from
  the plane equation, not approximated). When more than one surface
  overlaps the same plan footprint (a real Civil3D export routinely has an
  existing-ground surface AND a proposed/graded surface sitting on top of
  it in the same area — a pad, a driveway, a building pad), the lookup
  collides with whichever surface is actually **highest** at that exact
  point, the same way real water would run off a raised pad instead of
  silently draining straight through it into the ground surface
  underneath. A droplet respawns at a fresh
  random point on the surface once it flows off the modeled edge or lands
  on a flat/vertical facet with no defined downhill direction, and faster
  on steeper triangles than gentle ones. This is a **stylized visual
  approximation of surface runoff direction, not a hydrology/watershed
  simulation** — there's no infiltration, ponding capacity, channel
  concentration, or flow accumulation, just "which way does this exact spot
  drain." Renders in **both 2D plan and 3D orbit** — each droplet's true
  terrain-interpolated elevation is looked up from its own cached triangle,
  so in 3D the flow visibly sits on and follows the tilted surface, not a
  flat plane. Only *starting a dump by clicking a point* (below) stays 2D
  only, same reasoning as ⛰ Elevation/🧲 Snap — once water is flowing, both
  it and any dumped bursts keep animating and rendering correctly if you
  switch to 3D orbit mid-flow, and the 🌊 Water Flow toggle itself now works
  from either view.
- **💧 Dump Water** — click a point on a visible TIN surface to pour a
  one-shot burst of 40 droplets there, spread in a small ~1.2-unit disk
  around the click (not stacked on one pixel, and not scattered across
  whichever triangle happens to contain the click — a coarse TIN's own
  triangles can span tens of units, so a naive "random point in this
  triangle" jitter can badly overshoot "at a point"). Clicking Dump Water
  on an idle surface starts the ambient 🌊 Water Flow too, with the burst
  added on top of it, not instead of it.
  - **The burst is a real (if small/local) fluid, not 40 independent
    droplets.** Unlike the ambient droplets — which each just teleport at a
    slope-dependent constant speed along their own current triangle,
    completely unaware of each other — a dumped burst is simulated with the
    same core technique as Matthias Müller's "Ten Minute Physics" FLIP
    tutorials (a staggered MAC-grid PIC/FLIP fluid solver), re-derived
    independently for this heightfield rather than copied line-for-line:
    each frame, every dumped particle (1) picks up the LOCAL terrain's own
    downhill pull (the same `triFlowDir` plane-normal math the base flow
    already uses, standing in for the tutorial's one constant box-gravity
    vector) and advects; (2) gets pushed apart from anything it's now
    overlapping; (3) splats its velocity onto a small grid, freshly rebuilt
    every frame to snugly bound wherever the live burst currently is; (4)
    that grid is solved to zero divergence (Gauss-Seidel with
    overrelaxation — the actual incompressibility projection, not a
    cosmetic effect) so the water can't compress into itself; (5) the
    corrected grid velocity is blended back onto each particle via a
    90% FLIP / 10% PIC mix, the tutorial's own default ratio. The net
    effect: a dump visibly spreads into a puddle and slides downhill as one
    connected body, with real momentum (it has inertia — releasing on a
    slope keeps it accelerating, unlike the ambient droplets' instant
    constant speed), instead of 40 particles passing through each other.
    **The per-slope gravity scaling has the same floor + amplification as
    the ambient droplets' own speed formula** (`Math.max(0.3, Math.min(1.5,
    slope*3))`) — a real civil site's typical grade is only a percent or
    two, and the FIRST version of this shipped with no such floor, so on
    any realistically gentle real-world slope the resulting acceleration
    was nearly zero: the burst would spread from the pressure solve but
    barely translate, reading as "floating in place" rather than flowing.
    The floor guarantees a clearly visible minimum acceleration on any
    real slope (a truly flat facet still correctly gets zero gravity and
    rests as a puddle, unaffected — the floor only applies once there IS a
    defined downhill direction at all).
  - What still makes a dumped particle behave like a one-time pour rather
    than a permanent fixture: each one carries its own **decay closure** —
    `makeDumpParticle(E,N,tri,layer)` captures the exact timestamp the
    particle was born and returns a `decayed(now)` function closed over
    that birth time — so **15 seconds after being dumped, it's removed
    outright, wherever it's since flowed to**, rather than respawning
    elsewhere. A dumped particle sitting on flat ground is left alone as a
    valid resting puddle (unlike an ambient droplet, which respawns the
    instant it hits a flat facet with no defined downhill direction) — it's
    only removed early if it genuinely flows off the modeled TIN surface
    entirely, or once its own 15s is up.
  - The tool stays armed for repeat dumps at different points (each new
    burst joins the same shared local grid, so overlapping dumps interact
    with each other too); Esc cancels it. *Starting* a dump by clicking a
    point on the canvas is still 2D plan view only (there's no valid
    world-space inverse for the oblique 3D orbit projection to resolve a
    click against — the same limitation ⛰ Elevation/🧲 Snap/COGO picking
    already have) — but once a burst exists, it renders and keeps flowing
    correctly in 3D orbit too, same as 🌊 Water Flow itself.
- **🗻 Surface Display** offers alternate/additional ways to read a TIN,
  each independently toggleable and drawn in both 2D plan and 3D orbit:
  - **Wireframe** (on by default) — the same triangle-edge mesh always
    drawn before; turn it off to see contours/shading/arrows on their own,
    e.g. for a cleaner printout.
  - **Contour lines** — exact per-triangle "marching triangles" extraction
    (not a resampled grid): each triangle's true planar crossing of a given
    elevation is found directly from its own 3 vertices. The interval
    auto-computes a sensible round step from the surface's own elevation
    range, or type one in to override it. Every Nth level (5 by default,
    also adjustable) draws **major** — solid and a touch bolder — and the
    rest draw **minor** — dashed and thinner, the standard cartographic
    index-contour convention. Rather than drawing each triangle's raw
    straight-line crossing in isolation, the true crossing points are
    **chained into continuous polylines** first (two triangles sharing an
    edge share that edge's exact crossing point, so the real, connected
    contour line — or closed loop, for a hill/depression — was always
    implicit in the triangulation) and then rendered as a **smooth spline**
    (Catmull-Rom, converted to cubic Béziers) through those same true
    points, instead of a jagged polyline.
  - **Slope shading** — every triangle filled by its own steepest-descent
    steepness (the same exact plane-normal math 🌊 Water Flow uses),
    classified into **5 shades** from gentle (green) to steep (red).
    Classification is relative to *that surface's own* slope range
    (equal-interval, the same auto-scaling idea the contour interval
    already uses), so it stays informative whether the surface loaded is a
    nearly-flat lot or a steep hillside, rather than assuming a fixed,
    possibly-meaningless absolute % scale.
  - **Slope arrows** — one steepest-descent arrow per triangle, with a
    **density slider** capping how many draw per surface (20–1000, default
    300) so a dense TIN stays legible and fast — the fixed on-screen arrow
    length keeps them visible at any zoom regardless of that setting.
  - A **droplet-count slider** (5–5,000, default 50) for 🌊 Water Flow
    lives in this same panel — dragging it while the animation is already
    running re-seeds it immediately at the new count, not just on the next
    toggle. Thousands of droplets stay smooth because each one caches which
    triangle it's currently in and only re-scans the whole surface on the
    rare frame it actually crosses into a new one (or that triangle's
    layer gets hidden) — an O(1) fast path instead of an O(triangle count)
    scan on every droplet, every frame.
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
- **Touch gestures** (phone/tablet) — the common navigation set works the
  same way in 2D plan, 3D orbit, and profile view: pinch with two fingers to
  zoom (about the pinch midpoint, matching the mouse wheel's zoom-toward-cursor
  behavior), drag with two fingers to pan without zooming, and double-tap to
  ⊡ Fit (zoom to extents). In 3D orbit specifically, a single finger rotates
  (same as a mouse drag) while two fingers pan, mirroring the desktop's
  drag-to-rotate / shift-drag-to-pan split without needing a modifier key.
  Lifting one finger mid-pinch smoothly hands off to single-finger
  panning/rotating with the finger that's still down, rather than stopping
  the gesture dead. `touch-action:none` on the canvas hands all of this to
  the app instead of the browser's own native pinch-zoom/scroll.

## 🌧 Rain (Water Flow) and 🖌 painted rain areas

- **🌊 Water Flow** now rains. Each drop lands at a random spot spread
  evenly by area over the visible surface(s), so large triangles get
  proportionally more drops. Drops land after random delays and live for
  random lengths of time, so the rain is continuous rather than arriving
  in synchronized waves. A landing drop shows a falling streak in 3D or a
  splash ring in 2D, then flows downhill by steepest descent as before.
- **🖌 Rain Area** paints where the rain falls: drag on the drawing in 2D
  plan (Esc stops). Painting switches the rain to *Painted areas only*.
  The 🌧 Rain box (bottom-left, shown while Water Flow or painting is on)
  has Whole site / Painted areas only, 🖌 Paint, ⌫ Erase, Clear, brush
  size, rain intensity (drop count), and the painted area. Painted cells
  sit on a ~250-across grid over the site and show as a blue tint in 2D
  and draped on the surface in 3D.

## Slope between two points

- **📏 Slope 2-Pt** measures the slope between two picked points on a
  chosen **target surface** (or "Auto", the topmost visible surface at each
  point, so a pad over existing ground reads the pad). Click A, then B, in
  2D plan. Both points snap to TIN vertices/edges, CAD lines and points. A
  dashed rubber band shows the live grade before you click B. Each result
  lists A/B northing, easting and elevation, ΔZ, horizontal and slope
  distance, signed grade % (A → B, negative = downhill), H:V ratio and
  bearing. It's drawn in 2D and 3D with a downhill arrow, and the table
  exports to CSV. A point off the target surface is rejected. It only falls
  back to a snapped CAD point/line's own Z, never another TIN's elevation.

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
  - **🎯 Pivot snap** — when a drag starts, the orbit pivot snaps to the
    nearest CgPoint / DXF POINT / manhole, else the nearest DXF or TIN
    vertex within 12px, else the front-most TIN surface point under the
    cursor. Hovering previews the snap target (green square = vertex,
    circle = surface) and a pink crosshair marks the live pivot while
    dragging. Toggle it off in the 3D view box.
  - **Vertical exaggeration slider** (0.5×–25×, 1× button for true scale)
    in the 3D view box, top-right. Rescales Z about the pivot, so the pivot
    stays put on screen.
  - **🎨 Surface shaders** (3D view box or 🗻 Surface Display, drawn in both
    2D and 3D): Solid (layer color), Hillshade, Elevation (hypsometric tint),
    Slope ramp, Aspect (hue by facing direction), and Normal map. Lit shaders
    use a ☀ sun with adjustable azimuth/altitude, with optional **cast
    shadows** ray-marched over a height grid of the visible TINs. Lighting
    and shadows use the current vertical exaggeration. Triangles are
    flat-shaded and painter-sorted back to front in 3D.
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

## Touch / tablet navigation

The canvas already used Pointer Events (not separate mouse/touch handlers),
so a single finger dragging on a tablet always panned in 2D, orbited in 3D,
and scrubbed the profile view exactly like a mouse — that part needed no
new code. What a mouse genuinely can't do — put down two contact points at
once — is what was missing, since a real tablet workflow leans on
two-finger gestures for the things a mouse uses a wheel or a modifier key
for.

- **Pinch to zoom** — spread two fingers apart to zoom in, pinch them
  together to zoom out, in **2D plan, 3D orbit, and profile view alike**.
  The zoom is centered on the pinch's own midpoint, the same "whatever's
  under your fingers stays under your fingers" feel as a wheel zoom under
  the cursor.
- **Two-finger drag to pan** — sliding two fingers together (without
  changing how far apart they are) pans the view. This is the one gesture
  that's genuinely new capability, not just a touch equivalent of something
  a mouse could already do: **3D orbit previously had no way to pan on a
  touchscreen at all** — panning there needs a Shift key or a middle mouse
  button, neither of which exists on a tablet, so before this a finger could
  only ever rotate the 3D view, never move the pivot sideways. A real pinch
  almost always drifts a little even when the user "only" meant to zoom, so
  both gestures are computed together as one combined transform every
  frame, re-baselined each frame from the immediately preceding one rather
  than the gesture's start — the same increment-per-frame approach the
  mouse wheel handler already used, generalized (`pinchZoomPan`) to a
  moving anchor instead of a stationary cursor, so the wheel handler now
  calls the very same function with old-anchor=new-anchor as its own
  zoom-in-place special case.
- **Double-tap to fit** — a quick double-tap anywhere on the canvas resets
  the view to ⊡ Fit (whichever view mode is active), the same shortcut the
  `F` key already provides — a fast way back if a pinch/pan/orbit gesture
  leaves the view somewhere confusing while getting used to the controls. A
  single tap alone does nothing; only two taps close together in both time
  and position count.
- A third or later finger is tracked (so lifting it later doesn't confuse
  the gesture) but otherwise ignored — only the first two fingers drive
  anything. Lifting one finger out of an active two-finger gesture resumes
  a plain single-finger drag with whichever finger is still down, seeded
  from its own current position so there's no jump.

Verified end-to-end in a real headless browser using synthetic touch
`PointerEvent`s (`pointerType:'touch'`) dispatched directly at the canvas,
the same event shape a real touchscreen delivers: a lone finger still pans
in 2D and orbits in 3D exactly like a mouse; landing a 2nd finger correctly
starts a pinch and cancels any single-finger drag in progress; spreading
two fingers apart zooms in (`view.s`/`orbit.s` both confirmed increasing by
more than 50% for a 2× spread) while pinching together zooms out; a
two-finger drag at a **constant** distance apart pans without changing
scale at all (`view.s`/`orbit.s` unchanged to float precision) while
clearly moving `view.x/y` or `orbit.ox/oy`; lifting one finger out of a
pinch resumes an ordinary single-finger pan with the remaining finger, no
jump; a 3rd finger joining or moving does nothing to the gesture; a genuine
double-tap resets a deliberately-mangled view back to a proper fit, while a
single tap alone (checked after the double-tap window expires) does
nothing; and — since the wheel handler was refactored to share
`pinchZoomPan` with the new touch code — a real mouse wheel zoom still
produces the exact same `view.s` (to 1e-6) as before the refactor. Also
fixed a related edge case surfaced by this same testing: `setPointerCapture`
can throw if the browser doesn't consider a given pointer ID currently
active (confirmed harmless and expected for synthetic test events, but
cheap to guard defensively regardless of cause) — every `setPointerCapture`
call in the pointerdown handler is now wrapped in a try/catch so a capture
failure can never stop a finger from being tracked. Re-ran the complete
existing regression battery (FLIP/water-flow, surface-collision, DXF/
LandXML/pipe-network import, elevation query, snap, aerial map) — all pass
unchanged. `index.html`'s inline script still parses clean (`node --check`).

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

A seventh pass verified 🗻 Surface Display against that same known ramp:
the wireframe checkbox correctly toggled `SURFACE_DISPLAY.wireframe` both
ways; the auto contour interval produced a sensible, in-range level set
from the surface's own true 0–10 elevation range; a hand-verified exact
case — a custom interval of 5 on this exact ramp — produced precisely the
2 triangle segments expected, both lying exactly on the true `E=5` vertical
line and together spanning the full `N=0..10` range with no gaps, while a
level entirely outside the surface's range correctly produced zero
segments; clearing the typed interval correctly reverted to auto; and
turning on every overlay together (wireframe, contours, slope arrows) and
rendering in both 2D plan and 3D orbit produced zero console errors.

An eighth pass verified the three new sliders/controls end-to-end. The
droplet-count slider updated its backing variable and its live label even
with the animation off; turning the animation on then spawned exactly the
slider-set count; and dragging the slider again *while already running*
re-seeded the particle array immediately to the new count, not merely on
the next toggle. The slope-arrow density slider updated its backing
variable, its label, and was confirmed to feed directly into the actual
per-surface decimation formula. For major/minor contour styling, canvas
`setLineDash` calls were spied on directly: on a known 6-level contour set
(interval 2 over the ramp's true 0–10 range, majorEvery set to 3), the
recorded dash pattern was solid exactly at levels 0 and 6 — the two
positions `idx % 3 === 0` predicts — and dashed at every other level, with
the line dash correctly reset to solid at the end of the draw so no other
layer inherits it.

A ninth pass verified 5-class slope shading, contour spline smoothing, and
the droplet scale-up together against two new synthetic surfaces: a set of
5 independent triangles built with exact, hand-derived slopes (0.1, 0.325,
0.55, 0.775, 1.0) landed in exactly the 5 distinct classes 0–4 the
equal-interval formula predicts, one triangle per class. Contour chaining
was checked on two shapes: the known ramp's level-5 crossing (2 raw,
disconnected triangle segments) was correctly reassembled into 1 continuous
3-point chain lying exactly on its true `E=5` line, and spying on
`bezierCurveTo` confirmed the resulting "spline" through 3 exactly-colinear
points stayed perfectly straight, as it mathematically must; a synthetic
8-facet cone's level-5 crossing (8 raw segments, one per facet) was
correctly reassembled into a single **closed loop** of 8 unique vertices,
each sitting at exactly half the cone's true rim radius, rendered as a
smooth curve rather than a jagged octagon (screenshot-verified — the raw
octagonal data visibly rounds into a near-circle). For the droplet slider,
raising it to its new 5,000 maximum spawned exactly 5,000 particles, all
finite and on the surface, and 120 simulated steps of all 5,000 completed
in ~126ms while the population still net-drifted in the true downhill
direction — confirming the new per-particle "last known triangle" cache
keeps this responsive. The cache's two invalidation paths were each forced
directly: a deliberately wrong cached triangle self-corrected to the real
one containing the particle on the very next step, and hiding a droplet's
only surface layer mid-animation correctly respawned it to nothing rather
than silently continuing to use the now-hidden surface's stale cache.

A tenth pass verified 💧 Dump Water against the same 5-triangle synthetic
pyramid (a real, non-degenerate slope everywhere, so nothing is "pooled"):
clicking Dump Water on an idle surface (Water Flow not yet running)
correctly auto-started the ambient animation with its own default droplet
count AND added the 40-particle burst on top, both confirmed by counting
`waterParticles` split by its new `dumped` flag; the tool stayed armed and
a second click at a different point correctly stacked a second, independent
40-particle burst (80 dumped total) without touching the ambient count;
Esc correctly disarmed the tool. For the decay itself, `stepWaterParticles`
was called directly with a fabricated timestamp ~15.1 simulated seconds
past a burst's own real birth time (`performance.now()` at dump time,
captured inside each particle's own closure) — confirmed a freshly-dumped
particle's `decayed(now)` correctly reads `false` immediately after being
dumped, and that after the simulated 15.1s every one of the 80 dumped
particles was spliced out of the array entirely (down to exactly the
50 ambient particles, unaffected) rather than respawned the way an ambient
droplet would be. Turning off 🌊 Water Flow afterward correctly cleared
the array back to empty, dumps included.

An eleventh pass verified the FLIP-fluid upgrade to 💧 Dump Water and, along
the way, caught and fixed a real pre-existing bug in the original dump
jitter. **The bug:** the burst's "spread around the click" sampled a
uniformly random point across the ENTIRE clicked triangle via barycentric
coordinates — fine on the tenth pass's small synthetic pyramid, but on a
real/coarser TIN a single triangle can span tens of world units, so the
"burst" could land scattered across a large chunk of the model instead of
near the actual click. Caught directly: a two-triangle 40×40-unit ramp
produced an initial burst with a 33+ unit spread on the very first test
run. Fixed by sampling a small disk (`WATER_DUMP_SPREAD_RADIUS`, 1.2 world
units, `sqrt(rand)` for uniform density) around the clicked point itself
and only falling back to the clicked triangle if a jittered point lands
off every triangle; re-verified the same ramp scenario now spreads within
~2.2 units of the click, as intended. **The FLIP physics itself:** (1) a
direct unit test built a synthetic splat with deliberately divergent
velocities, confirmed real nonzero divergence existed pre-solve, then
confirmed `flipSolveIncompressibility` drove it down by >95%; (2) a direct
separation-only test confirmed 3 overlapping synthetic particles end up
farther apart after `pushParticlesApart`'s formula, not closer/unchanged;
(3) end-to-end on the (correctly re-authored, LandXML N-E-Z order double
-checked) ramp, `triFlowDir` was queried directly at the dump point FIRST
to get the real ground-truth downhill vector (rather than assuming a
direction), then 40 real animation-frame steps of the actual dumped burst
were run and its net displacement projected onto that real downhill vector
came back clearly positive (+4.07 units) — confirming actual accelerating
motion in the physically correct direction, not just "moved somewhere."
The burst's bounding spread also grew over those same 40 steps (confirming
it visibly spreads/sloshes rather than staying in its initial tight
cluster), and every particle's E/N/vE/vN stayed finite throughout (no
NaN/Infinity blow-up). Ambient droplets were re-confirmed completely
unaffected by any of this (still exactly 50, untouched, after the FLIP
burst ran its 40 steps), and the pre-existing 15-second decay-closure
behavior was re-verified unchanged on top of the new physics. Re-ran the
complete existing regression battery (DXF/LandXML/pipe-network import,
elevation query, 3D orbit, snap, aerial map) — all pass unchanged.
`index.html`'s inline script still parses clean (`node --check`).

A twelfth pass fixed a real reported bug in that same FLIP gravity: the
owner tried it and reported the dump "floating," not flowing. Root cause,
confirmed numerically before touching any code: the gravity magnitude was
`WATER_DUMP_GRAVITY * Math.min(1.5, dir.slope)` — no floor — so on a
realistic civil-site grade (a synthetic 2% slope, `dir.slope≈0.02`, built
specifically to reproduce a real site rather than the earlier test's much
steeper 25% ramp), the resulting acceleration was `16*0.02=0.32` units/s²;
after 1 simulated second (about what someone actually watches before
judging "is this flowing"), that's a mean speed of only ~0.32 units/sec and
~0.16 units of displacement — genuinely imperceptible, exactly matching
"floating." Fixed by giving the gravity scaling the SAME floor +
amplification the ambient droplets' own (already-tuned, already-visible)
speed formula uses, `Math.max(0.3, Math.min(1.5, slope*3))`. Re-ran the
exact same 2%-slope scenario after the fix: mean speed after 1 simulated
second came back at ~4.86 units/sec (15× higher) and displacement at ~2.80
units along the real, independently-queried downhill direction — clearly
visible motion. Re-ran the eleventh pass's own steeper-ramp scenario
unchanged (it never depended on the floor, since its slope was already
well above where the floor kicks in) — still passes with the same
qualitative behavior. Re-ran the complete existing regression battery
(FLIP unit tests, dump/decay lifecycle, DXF/LandXML/pipe-network import,
elevation query, 3D orbit, snap, aerial map) — all pass unchanged.
`index.html`'s inline script still parses clean (`node --check`).

A thirteenth pass fixed water flow going 2D-only, read by the owner as
"still floating in the air" after the twelfth pass's gravity fix — a
report that only makes sense in a 3D context, since a flat top-down 2D
view has no vertical axis to perceive floating in. Root cause: the whole
feature was 2D-only in a way that went beyond the documented "starting a
dump needs a click" limitation — entering 3D orbit unconditionally called
`stopWaterAnimation()`, killing any running flow outright, and even if it
had kept running, `draw3D()` never called a water-drawing function at all,
so nothing would have appeared regardless. Fixed by: (1) generalizing
`drawWaterParticles` to take a `proj(E,N,Z)` callback and a `need3DZ` flag,
matching the same shared-projection pattern already used by
`drawPipeNetworks`/`drawSlopeShading`/`drawContours`/`drawGeomLabels` — in
3D it looks up each particle's real terrain Z via its own cached `_tri`
and the existing `pointInTri` barycentric helper, rather than assuming
Z=0; (2) wiring a `drawWaterParticles((E,N,Z)=>P3(E,N,Z),true)` call into
`draw3D()`, alongside the pre-existing `drawWaterParticles((E,N)=>W2S(E,N),
false)` call in `draw2D()`; (3) removing the forced `stopWaterAnimation()`
from entering 3D orbit, so a running flow (ambient or a dumped burst)
keeps simulating and rendering across the view switch instead of being
killed; (4) removing 🌊 Water Flow's own "2D only" refusal, since the
toggle itself has no click-to-place step and works identically in either
view now. 💧 Dump Water's click-to-place trigger deliberately stays 2D-only
and untouched — there's still no valid world-space inverse for the oblique
3D orbit projection to resolve a canvas click against, the same reasoning
⛰ Elevation/🧲 Snap/COGO picking already document. Verified end-to-end in a
real headless-browser session on the same realistic 2%-slope fixture the
twelfth pass used: a burst dumped in 2D (90 total particles: 40 dumped +
50 ambient) survived switching to 3D orbit completely unstopped (still 90,
`waterOn` still true); a live `P3` projection of a real dumped particle
using its true interpolated terrain Z differed from a naive Z=0 projection
by ~14px on screen — proof the 3D draw path is actually using real
elevation, not floating at a flat Z=0 plane; a real `draw3D()` frame ran
with the water drawn and threw no error; clicking 💧 Dump Water while in 3D
still correctly refused to arm (`waterDumpOn` stayed `false`); and 🌊 Water
Flow's own toggle button correctly stopped AND restarted the ambient
animation from within 3D orbit, something it previously refused outright.
Re-ran the complete existing regression battery (FLIP unit tests, dump/
decay lifecycle, gentle-slope gravity, DXF/LandXML/pipe-network import,
elevation query, 3D orbit, snap, aerial map) — all pass unchanged, zero
console errors beyond the pre-existing, already-documented aerial-map
tunnel-connection failures this sandbox always produces. `index.html`'s
inline script still parses clean (`node --check`).

A fourteenth pass fixed water silently ignoring/passing through whichever
surface should have stopped it — the owner reported "the particles are not
colliding with the surfaces." Root cause: `findWaterTriangleAt`, the one
lookup every water function (ambient flow, dumping, the FLIP solver's
per-frame terrain pull) uses to find which triangle a point sits on,
scanned `SURFACES` in plain file order and returned the FIRST surface
whose triangle contained the point — with no regard for elevation at all.
A real Civil3D LandXML export routinely has more than one surface
overlapping the same plan footprint (an existing-ground surface plus a
raised/graded proposed surface — a pad, a driveway) — wherever they
overlap, water always locked onto whichever surface happened to be listed
first in the file, even where a different, physically higher surface was
the one actually there. From the user's point of view this looks exactly
like water failing to collide with (i.e. passing straight through) the
surface that should have been in the way. Reproduced directly with a
synthetic two-surface fixture — a broad, gently-sloped "EG" surface
(elevations 0–2) fully underlying a smaller, steeper "PAD" surface
(elevations 10–12) covering the middle of it: dumping water at a point
inside PAD's footprint resolved to `TIN-EG` at `Z=1`, confirming the old
code fell straight through the higher PAD surface onto the lower one
listed first. Fixed by having `findWaterTriangleAt` scan every visible
surface's matching triangle and keep the one with the **highest**
interpolated Z, not just the first found — physically the correct
"collision" semantics for a set of heightfield surfaces, since real water
would rest on / run off whichever surface is actually uppermost at that
point rather than the ground surface underneath it (a valid TIN never has
two overlapping triangles within itself, so it's still only ever one
triangle test per surface, just no longer short-circuiting after the
first surface). Verified: the same synthetic fixture now resolves the
overlap point to `TIN-PAD` at `Z=11`, and dumping a full 40-particle burst
there confirms zero particles land on the lower EG surface instead of
colliding with PAD; a point outside PAD's footprint but still on EG
(unaffected region) still correctly resolves to `TIN-EG`, confirming the
fix only changes behavior where surfaces genuinely overlap. Re-ran the
complete existing regression battery (FLIP unit tests, dump/decay
lifecycle, gentle-slope gravity, 3D rendering, DXF/LandXML/pipe-network
import, elevation query, 3D orbit, snap, aerial map) — all pass unchanged,
zero new console errors. `index.html`'s inline script still parses clean
(`node --check`).

A fifteenth pass (merged in from a parallel branch) verified saved elevation spots and TIN-aware snapping end-to-end
against the known ramp surface. `collectSegments()` was confirmed to emit
exactly 6 TIN wireframe edge segments (2 triangles × 3 edges each), and
`snapPoint()` correctly resolved a click a few pixels off a true TIN vertex
to its exact coordinates `(E0, N0, Z10)`, and a click near an edge's midpoint
to the exact midpoint `(E5, N0)`. Arming ⛰ Elevation and clicking twice
confirmed the tool now stays armed — the spot count grew from 0 to 1 to 2
rather than overwriting a single slot — with the first, snapped pick landing
exactly on the true vertex `(0, 0, 10)` and recording `"TIN vertex"` as what
it snapped to; the snap-options box was also confirmed visible while
elevation picking is armed even though 🧲 Snap itself is off, since both
tools now share the same underlying snap engine. Toggling the tool off
correctly stopped picking. **📐 Elevation Spots** opened to a 2-row table
matching the 2 saved picks, and its CSV export produced a header plus
exactly 2 data rows, with a "Snapped to" column present; Clear all correctly
emptied the saved list back to 0. The 500-spot cap was exercised directly by
pushing 501 entries and confirming exactly 500 remained, with the oldest
entry dropped and the newest kept. Finally, the general 🧲 Snap tool (with
elevation picking off) was confirmed to independently find the same TIN
vertex, proving the wireframe-snap addition benefits both tools rather than
being elevation-specific. A companion screenshot test drove 4 sequential
real picks at deliberately offset click positions and confirmed each landed
exactly on its intended target — `(E0,N0,Z10)`, `(E10,N0,Z0)`, the edge
midpoint `(E5,N5,Z5)`, and `(E10,N10,Z0)` — with the table listing all 4 in
order, each with the correct snap label, and the canvas showing the most
recent pick at full strength with earlier ones dimmed.

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
