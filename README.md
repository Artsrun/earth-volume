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

## Accuracy pass (v2.1 → v2.2)

Three tiers, measured against the DEM rather than asserted.

### 1 · The Ararat constant was 8.33 km off the summit

`39.702, 44.396` samples **2,363 m** at native resolution — it is on the eastern
flank, not Masis. Searching the raster for the massif's own local maximum
(z10 → z12 → z14) recovers **5,111 m at 39.7019, 44.2986**, 27 m from the
declared 5,137 m. The §05 caption blamed the raster for an error in the
coordinate.

| | was | now |
|---|---|---|
| summit, native zoom | 2,363 m | **5,111 m** |
| ground distance | 53.384 km (frozen) | **55.217 km** (derived) |
| elevation angle, declared h | 4.34° | **4.18°** |
| elevation angle, DEM heights | 0.63° | **4.04°** |

`BASELINE_M` and `NET_DEPRESSION` are no longer frozen — both derive from the
coordinates, through the exact spherical form on an effective-radius sphere.
A refraction band (k = 0.07 … 0.25) is shown, because that spread is 2.6′ while
geodesic-vs-haversine, Euler-vs-mean radius and exact-vs-linearised angle come
to ~11″ **combined**: refraction dominates every geometric term by ~14×.

### 2 · The shell raster cannot report point elevations

Two independent reasons, both measured:

- **Scale.** Detail 5 vertex spacing is 209 km; a z3 cell is 19.6 km. Each vertex
  point-samples 1 of ~114 cells. Against the true cell mean over 40 random land
  points: **RMS 467 m, max 1,806 m**. Raising tile zoom makes it worse (z4 → 456
  cells per vertex); a mesh that resolves z3 needs detail 8.4 ≈ 2.3M faces.
- **Aggregation.** The Terrarium pyramid is mean-aggregated, so summits are
  destroyed going up it: Masis reads 2,657 m at z3, 4,316 at z6, 5,023 at z8,
  5,110 at z14. `MEASURED` at z3 means *area mean*, not *elevation*.

So the two jobs are now split. The shell keeps the coarse global raster; the
named field points get `probeH()` — a 4-tile fetch at z13 (14.6 m/px at 40°N)
bilinear-sampled off a scratch canvas. **Eight tiles, not 65,536.** The panel
prints both numbers side by side.

The observer residual survives the fix at **+103 m**, so it is terrain, not
aliasing: at z14 the 50 m neighbourhood of the pin spans 987–1012 m and the
lowest cell within 4 km is 876 m, ~3 km SSW. Either the pin sits up the bank or
the declared 895 m is the water surface elsewhere — reported, not explained away.

### 3 · Metres

- `Int16Array` assignment **truncates toward zero** — a signed bias (−1 m on land,
  +1 m in bathymetry) with a discontinuity at the datum. Now `Math.round`.
- Ground sample distance was quoted at the equator only; the observer's latitude
  figure (×cos φ) is shown alongside.
- The header claimed **EGM2008** while GMTED2010 is EGM96-referenced and no
  undulation model is applied anywhere in the code. Label corrected and stated.

Verified non-issue: the canvas decode path is byte-exact — the tiles carry no
`gAMA`/`iCCP`/`sRGB` chunk and `getImageData` matches a reference zlib decode.

---

## Line of sight (v2.3)

The question the project is actually about — *is Masis above the horizon from
that shore?* — was never computed. Endpoint arithmetic cannot answer it; the
terrain between can stand in the way.

On the effective-radius sphere the ray is straight and refraction lives in R′,
so the ray height at central angle θ follows from the sine rule:

```
r = (R′ + h₁) · cos α / cos(θ + α)        α = elevation angle to the summit
```

Walk the great circle at z12 (29 m/px), 512 points, and take the deepest
intrusion of terrain into that ray. Only the tiles the line crosses are
fetched — **10 tiles** for a 55 km baseline.

```
verdict                CLEAR
minimum clearance      +6 m at 0.11 km  (terrain 1,000 m)
clearance, open window  +122 m at 1.1 km
```

The binding constraint is **the observer's own bank, 110 m away** — which is the
same +103 m of terrain that the §04 residual reports, showing up a second time
as a grazing limit. Beyond it nothing intervenes. The result holds across the
whole plausible refraction range (k = 0.07 … 0.25 moves the open-window figure
by 1 m), so it is not a refraction artefact. The ray meets the ground at both
ends by construction, so the intervening terrain is judged over an open window
1 km clear of each.

Also in this pass:

- The baseline polyline **slerps the unit vectors** instead of lerping lat/lon,
  and the straight sight ray is drawn beside the terrain profile.
- The shell is the **WGS84 spheroid**. An oblate spheroid is a sphere scaled
  along its polar axis, so one scale gives the exact figure — and it matters:
  a − b is 21,385 m, **2.4× Everest**, so at ×1 relief the flattening is the
  dominant shape signal and it was missing.
- The point probe and the profile now share one tile-level sampler, so each
  tile is fetched and decoded once whoever asks for it.

Cross-checked against an independent implementation (pure-Python PNG decode,
same formulas): identical to the metre.

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