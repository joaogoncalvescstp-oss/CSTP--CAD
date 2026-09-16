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

## Verified

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
