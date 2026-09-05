# EARTH.VOLUME / geodesic shell · v2

Interactive icosahedral shell of Earth, displaced by **measured** elevation (Terrarium RGB tiles = GMTED2010 land + ETOPO1 bathymetry, AWS Open Data). Hypsometric colour, nested atmosphere shells at declared scale, device-tilt principal motion, and a field-report baseline from **Lake Yerevan → Ararat (Masis)**.

**Real device + web.** Open on a phone, grant motion permission, tilt the device to turn the shell. Drag / pinch as fallback.

Live: https://artsrun.github.io/earth-volume/ (or raw `index.html`)

---

## Code review (v1 → v2)

### Strengths kept
- Strict provenance tags: `measured` / `derived` / `constant` / `declared` / `absent`
- Concurrent tile pool + progress bar
- Correct Web-Mercator bilinear sampling (polar clamp ±85.051°)
- iOS `DeviceOrientationEvent.requestPermission` + screen-angle compensation
- Yerevan observer pin + Ararat cone + dashed baseline
- Flat-shaded Phong + additive atmosphere + cage

### Issues fixed in v2
| Issue | Fix |
|-------|-----|
| Subdivision params 32/48/72 produced impractically dense meshes for mobile GPUs; face count was approximate (`n/3`) | Standard detail levels **3 / 4 / 5** (1.3k / 5.1k / 20k faces). Accurate `faces = 20 × 4^detail` |
| No guard against concurrent DEM loads when zoom buttons hammered | `loading` mutex in `boot()` |
| No easy return to the Yerevan–Ararat home view after free drag | **Recenter · Yerevan view** button |
| Missing notched-device insets | `env(safe-area-inset-*)` padding |
| Mobile performance risk | Default detail = 4, mobile-first meta strip |

### Tests run
- Visual load of z3 tiles (64) — progress, hypsometric ramp, residual readouts (sampled − declared)
- Marker placement & baseline
- Orientation path (permission gate, αβγ readout)
- ResizeObserver + pointer drag + wheel zoom
- Mobile viewport stack (single column)
- Partial-tile failure path (cells read 0 m / ABSENT)

---

## Code review (v2 → v2.1)

Interaction and lifecycle pass over the v2 shell. No change to the physics, the
sampling, or the frozen constants — the numbers on screen are the same numbers.

| Issue | Fix |
|-------|-----|
| The header promised “drag / pinch as fallback”, but only `wheel` was bound — a phone with motion permission denied had **no way to zoom at all** | Two-finger pinch dolly on the pointer-event path; two pointers dolly and never rotate |
| Shell unreachable without a pointer | Canvas is focusable, labelled, and driven by arrow keys (shift = coarse) with `+` / `−` to dolly |
| `placeMarkers()` built a fresh `Line` + `LineDashedMaterial` per call and disposed only the geometry — the exaggeration slider runs it ~60×/s | Baseline geometry is written in place; one material for the life of the page. One 200-step sweep: **200 → 0** leaked materials, **200 → 0** orphan `Line` objects |
| Every slider tick recomputed 20k-face normals synchronously | One `applyExag` per animation frame (`computeVertexNormals` **200 → 1** over the same sweep) |
| A throw inside `loadDEM` (tainted canvas, or the z4 4096² raster failing to allocate on a small device) left `loading = true` and the loader overlay up **forever** | `boot()` wrapped in try/finally; the failure is reported in the loader line and the shell falls back to datum radius |
| A second zoom click during a load mutated `ZOOM` under the in-flight `boot()` — readouts then labelled a z2 raster as “terrarium z4 · 9.8 km/px” | Zoom/detail buttons ignore clicks and disable while `loading` |
| `deviceorientation` with a partial sensor (`beta`/`gamma` null) threw inside the handler | All three angles guarded |
| `prefers-reduced-motion` only stopped CSS transitions; the ring kept pulsing and the atmosphere kept spinning | Honoured in the render loop too — steady ring, still atmosphere, instant slerp |
| Dead `DETAIL = isMobile ? 4 : 4` | Removed; `isMobile` now caps device pixel ratio at 1.75 (3× DPR + additive shells is what melts phone GPUs) |
| z3 → z4 held both rasters live during decode | Previous `Int16Array` released before the next allocation |

Verified headless (Chromium/Playwright) against cached z2/z3 Terrarium tiles: boot,
sampling readouts, all three detail levels, layer toggles, keyboard and pinch paths,
the total-tile-failure path (z4 aborted → `ABSENT`, loader recovers), and recovery
back to z3.

---

## Field constants (frozen)
- Observer: 40.177°N 44.487°E · declared 895 m (SW shore Erebuni / Lake Yerevan)
- Ararat summit: 39.702°N 44.396°E · declared 5 137 m
- Ground distance (haversine): 53.38 km
- Net curvature − refraction depression: 194.6 m → elevation angle 4.34°

Sampled heights and the derived angle come from the DEM raster; any residual is the cell size, not the mountain.

---

## Controls
- **Use device tilt** — live orientation (recalibrate by stopping/starting)
- **Recenter** — snap back to the Yerevan home quaternion
- **Drag** to rotate · **pinch** or **scroll** to dolly
- **Keyboard** — focus the shell, arrows rotate (shift = coarse), `+` / `−` dolly
- Relief exaggeration slider (declared display scale)
- Shell subdivision (coarse / medium / fine)
- Layer toggles: sea datum / atmosphere / cage
- Tile zoom z2–z4 (higher = finer GSD, more tiles)

---

## Provenance legend
- **measured** — sensor / dataset value  
- **derived** — computed from measured inputs  
- **constant** — frozen field-report figure  
- **declared** — display choice, not physical  
- **absent** — no data in source (Mercator poles, failed tiles)

Built for real-device exploration of the measured Earth shell.