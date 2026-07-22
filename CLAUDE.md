# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Interactive prerequisite map ("mapa de correlatividades") for a 5-year university degree program. The user clicks subjects to mark them as passed; the map recolors to show which subjects are now unlocked, and a progress bar tracks completion. UI text and identifiers are in Spanish — keep that convention.

## Running

There is no build step, no package manager, and no test suite. The app is two files: `index.html` (all markup, CSS, and JS inline) and `materias.json` (the data).

`index.html` loads the data with `fetch('materias.json')`, so opening the file directly via `file://` fails with a CORS error. Serve over HTTP instead:

```powershell
python -m http.server 8000   # then open http://localhost:8000
```

Deployment is automatic: pushing to `main` triggers `.github/workflows/azure-static-web-apps-*.yml`, which uploads the repo root as-is to Azure Static Web Apps.

## Data contract (`materias.json`)

A flat JSON array. Each entry:

```json
{ "codigo": "03621", "materia": "MATEMATICA DISCRETA", "anio": 1,
  "correlativas": [], "horas_semanales": 4, "horas_totales": 64 }
```

- `codigo` is the identity key — `correlativas` is an array of the `codigo` values that must be passed first. Referential integrity is not validated at runtime; a typo'd code silently makes a subject permanently unavailable.
- `anio` **must be 1–5**. Both the layout loop and the `coloresPorAnio` palette are hardcoded to that range; a subject with `anio: 6` is dropped from the layout, and `anio: 0` throws on color lookup. Extending the program means updating `coloresPorAnio`, the `for (let anio = 1; anio <= 5; ...)` loops in `calcularLayout`, and the `#legend` markup together.
- `horas_semanales` / `horas_totales` are present in the data but unused by the app.

## Architecture (`index.html`)

Everything hangs off module-level state and a single init chain. `cargarMaterias()` fetches and validates the JSON, populates `materiasMap` (código → materia), then calls `inicializarAplicacion()`, which runs in a fixed order: `calcularLayout` → `crearElementosMaterias` → `cargarProgreso` → `actualizarEstados` → `actualizarProgreso` → `dibujarFlechas` → `inicializarVista`.

**Hybrid DOM + SVG rendering.** Subjects are absolutely-positioned `div.subject` elements; the prerequisite arrows are `<path>` children of a single 5000×5000 `#flechas` SVG underneath them. Both live inside `#viewport`, which is pan/zoomed as a unit via a CSS `transform` on the parent — SVG user units are therefore fixed world space (0–5000) and never need to account for scale/offset, and the arrows stay vector-crisp at every zoom level (a canvas here would rasterize at 1× and blur when zoomed to 3×). `#flechas` is `pointer-events: none`, so clicks and drag-to-pan fall through to `#viewport`. `dibujarFlechas()` rebuilds every path from scratch on each state change via `svgFlechas.replaceChildren()`.

Each arrow is two paths: a stroked shaft and a filled head (`pathPunta()`, a notched triangle whose tip sits at the target). Endpoints are not box centers — `bordeDeCaja(item, angulo, margen)` intersects the ray between the two box centers with each box's border, so arrows run edge to edge and heads always land on the side of the target that faces the source. That intersection assumes every box is `ANCHO_MATERIA`×`ALTO_MATERIA`, which must stay in sync with `.subject`'s CSS width/height.

**Layout is computed, not stored.** `calcularLayout()` places subjects deterministically: one year per pair of columns (`baseX + (anio-1)*2*columnWidth`), splitting each year's subjects into two columns and vertically centering each year against the tallest year. Position is a pure function of array order within a year — reordering entries in `materias.json` moves boxes on screen. `layout` entries carry both the source `materia` and a back-reference to the created DOM node (`item.element`).

**Two pieces of state drive all color.** `materiasAprobadas` (passed) and the derived `estaDisponible(codigo)` — true when not yet passed and every correlativa is passed. `colorDeEstado(codigo)` resolves the precedence — passed (green) > available (orange) > year base color — and is the single source for both subject boxes (`actualizarEstados()`) and inspector list items. Arrow colors follow the same precedence via the `arrow` field of `coloresPorAnio`.

**Inspector panel.** `#inspector` is a fixed right-side panel that slides in via a `transform: translateX` transition toggled by the `.open` class. Every way in — the `.subject-props` (⋯) button (which must `stopPropagation()` so it doesn't also toggle passed), ctrl/cmd+click, and clicking a row in the panel's own lists — goes through `alternarInspector(codigo)`, which highlights the subject and opens the panel, or closes it when that subject is already open. Every way out — `#inspectorClose`, `Escape`, re-triggering an opener — goes through `cerrarInspector()`, the single undo: it hides the panel, clears `materiaSeleccionada` and the `.selected` class, and redraws. Keep both funnels intact when adding entry points. Códigos are stored 5-digit but always displayed through `formatearCodigo()`, which strips the leading zero. `refrescarInspector()` rebuilds both lists from current state: "Anteriores" from `materia.correlativas` (códigos with no matching entry are filtered out) and "Siguientes" from `correlativasSiguientes(codigo)`, an O(n) scan for entries listing this código. `actualizarEstados()` calls it, so any state change repaints the panel; `materiaSeleccionada` is not persisted.

**Selection is a single código, not a set.** `materiaSeleccionada` alone drives every emphasis: the `.subject.selected` class (bold + indigo ring) on the box, and on each arrow with that código at *either* endpoint — incoming or outgoing — a 4px line with a 15px head plus a blue halo. The halo is the same two paths re-stroked in `colorHalo` at `haloGrosor` px wider with round caps/joins — stroking rather than scaling is what makes it an even outset matching the box's ring. All halos are appended before all arrows, so they sit underneath. Selecting another subject therefore returns the previous one to its base look with no bookkeeping. Because the arrows are rebuilt wholesale, `abrirInspector`/`cerrarInspector` must call `dibujarFlechas()`.

**Interaction.** Plain click on a subject toggles passed; ctrl/cmd+click selects it and opens the inspector, exactly like the ⋯ button. Drag-to-pan only starts when the mousedown target is the container or the viewport itself, so dragging a subject box does not pan. Wheel zooms around the cursor position, clamped to 0.2–3×.

**Persistence.** Only `materiasAprobadas` is persisted, as a JSON array of códigos under the `mapaCorrelatividadesProgreso` localStorage key. Any mutation must call `guardarProgreso()` — `toggleAprobada` and `resetAprobadas` are the only current writers. The selection is intentionally not persisted.

After changing any state, the four-call sequence `guardarProgreso / actualizarEstados / actualizarProgreso / dibujarFlechas` must run together; skipping `dibujarFlechas` leaves arrow colors stale.
