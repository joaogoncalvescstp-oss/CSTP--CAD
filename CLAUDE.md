# CSTP CAD Viewer: notes for Claude

A single-file web app (`index.html`: HTML, CSS and JS inline) with feature notes in `README.md`.

## Version badge: change it on EVERY push (owner's standing rule)

The badge next to the title (`VERSION` at the bottom of `index.html`) is how the
owner confirms the browser is running the newest push, not a cached copy.
**Every commit pushed to `main` must change it:**

1. Pick a **new animal emoji + NAME** that has not been used before (see the log
   below). Never reuse the current one.
2. Bump `build` by 1.
3. Add a line to the log below in the same commit.
4. Tell the owner the new badge (e.g. "badge is now 🦉 OWL · b21") in the reply.

The badge renders as `<emoji> <NAME> · b<build>`.

### Badge log (newest last)

| Build | Badge | Change |
|---|---|---|
| 1 | 🦊 FOX | first badge, 3D orbit / map / OSNAP ported |
| 2–20 | 🐨 KOALA | (badge animal was not changed on these pushes; only `build` moved) |
| 21 | 🦉 OWL | badge now changes every push and shows the build number |
| 22 | 🐢 TURTLE | 📍 Label Points: check-all boxes for all alignments / lines / polylines |
| 23 | 🐙 OCTOPUS | 📍 Label Points: reference alignment "None" keeps each alignment's own XML stationing |
| 24 | 🦩 FLAMINGO | 📍 Label Points: thin dashed tangent lines PI→PC/PT and radius lines CC→PC/PT |
| 25 | 🦦 OTTER | 🌊 water: color drops by speed (red/yellow/blue/purple) + 🕳 catch basins that capture drops |
| 26 | 🦫 BEAVER | 🚰 pipe flow: water caught by structures travels down the pipes to the outfall |
| 27 | 🦔 HEDGEHOG | UI cleanup: grouped toolbar + Import menu; every panel draggable / collapsible with remembered layout |
| 28 | 🦜 PARROT | ☀/☾ light & dark theme toggle (UI vars + auto-adapted canvas colors) |
| 29 | 🐧 PENGUIN | 🧊 3D view cube (faces / corners / home) + 🎥 perspective view with field of view |
| 30 | 🦒 GIRAFFE | 📈 profile: existing-ground lines from surfaces, % grades + L/K, vertical exaggeration, measure & adjust-PVI tools |
| 31 | 🦓 ZEBRA | 👁 deutan-safe color mode (Okabe-Ito palette, blue→orange/yellow ramps), on by default |
| 32 | 🐌 SNAIL | 🌀 click a labeled PI / CC / MID to open a Curve Properties panel (all curve formulas, stations, coords, bearings) |

## Workflow

- The owner wants finished work **pushed straight to `main`**. Keep the working
  branch in sync with `main` too.
- Before pushing: `node --check` the main inline script (extract the LARGEST `<script>` block — a one-line theme script sits in `<head>`), and exercise the
  change in headless Chromium (Playwright) with a small LandXML fixture.
