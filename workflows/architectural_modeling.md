# Architectural Modeling Workflow

Specialized path for buildings, interiors, structural elements, monuments.

## Core Approach

Architecture reconstructs from **plan + elevations**: the top view is the floor plan; front/side views are elevations. Geometry is extruded masses with wall thickness, not carved volumes.

## Stage Adaptations

### Stage 3 — Classification Signals

- Top view contains rectilinear room-like structure (Hough lines both axes, grid-like).
- Front view shows horizontal floor bands (row-wise edge density peaks at regular intervals).
- Overall aspect: height may exceed width (towers) or be wide (slabs); plan view dominates.

### Stage 4 — Geometry (Architectural Variant)

**Step 1: Plan Extraction**

1. Top view mask → outer contour (building footprint) + inner contours (courtyards, atriums).
2. Simplify contour: `approxPolyDP` epsilon = 0.5% of plan diagonal → rectilinear-ish polygon. Snap near-orthogonal edges to exact 90° where deviation < 2°.
3. Detect interior wall lines: skeletonize mask (`skimage.morphology.skeletonize`), find long parallel line pairs = walls; wall centerlines become wall segments. Fallback: if skeleton unreliable, treat plan as solid footprint and add floor divisions from elevation hints only.

**Step 2: Floor Levels**

From front view edge map: horizontal projection of floor lines → peaks in row-gradient magnitude. Regular spacing → floor_height = median spacing; irregular → individual heights. Default if undetectable: `total_height / assumed_floors` with assumed_floors = round(height / 3.2m) for buildings, or ask user.

**Step 3: Mass Extrusion**

1. Extrude plan polygon to full height → massing solid.
2. For each floor level: slice mass at floor heights → floor plates (slabs) thickness = 0.3m default or detected.
3. Subtract wall volume: walls = plan walls thickened vertically between floor levels. Wall thickness default: 2% of smallest plan dimension, clamp [0.15m, 0.6m]; override if elevation shows thick walls.
4. Openings (windows/doors): detect as rectangular protrusions/recesses in elevation masks? Elevations show outlines only — openings appear as interior detail lines, not silhouette. ORTHO GENESIS rule: **openings are NOT reconstructed from silhouette alone** unless they break the silhouette (e.g., archways, recessed entries visible in top view). Otherwise report `openings_not_reconstructable` and leave walls solid; user adds openings manually or via separate detailing pass.

**Step 4: Setbacks & Stepped Forms**

If top view shows stepped footprint (multiple nested rectangles) or elevations show setbacks: detect footprint area changes implied by side-view width variations with height → construct per-height plan sections (loft plan polygons along Z instead of single extrusion).

**Step 5: Ornamental Elements**

Domes, spires, cornices: if silhouette shows curved top → cap extrusion with lofted section (arc approximated by 3–5 segment polygon or actual curve bevel). Columns/pilasters: only if they break silhouette; else skip (texture-level detail).

### Stage 5 — Topology (Architectural Rules)

- Prefer large planar n-gon-free quads; walls = box strips (each wall segment a rectangular tube).
- Wall corners: minimal geometry — miter or butt joints, 1 supporting loop only if curved corner.
- Slabs: single-face-thick planes OK if `solidify` planned; for watertight requirement, give slabs real thickness (0.2–0.3m).
- Quad ratio ≥ 0.80 acceptable (architectural meshes are naturally quad/planar).

### Stage 6 — Modifiers (Architectural)

- Bevel: 1 segment, width 0.02–0.05m for edge highlights — or skip entirely for pure massing models.
- No subdivision.
- UV: `cube_project` per wall face orientation — generate box-projected UVs with scale = 1 UV unit per meter for tiling materials.
- Materials: assign placeholder slots `M_Wall_Ext`, `M_Roof`, `M_Glass` by face normal direction (top faces → roof, vertical → wall) — speeds user shading.

### Stage 7 — Validation (Architectural Gates)

- Silhouette IoU ≥ 0.90 (slightly relaxed vs hard-surface: architectural silhouettes include roof elements underdetermined by plan).
- **Floor count check**: detected floor bands in elevation must match modeled floor count ±1.
- **Plan coverage**: mesh top-view silhouette must cover ≥ 95% of reference plan mask (buildings shouldn't have unexplained holes except courtyards that exist in ref).
- Wall thickness sanity: measured thickness within [0.1, 1.0]m.

### Stage 8 — Delivery

Architecture deliverables: `.blend` (layered collections per floor optional via spec), `.fbx`/`.obj` for BIM/CAD import, `.glb` for web viewers. `.stl` rarely needed (scale issues) — warn if requested on building-scale model (STL units are mm; a 50m building = 50000 units, most slicers choke).

## Scale Handling

Architecture is where `known_dimension` matters most. If user provides building height or footprint dimension, calibration is exact. Without it, assume floor_height = 3.0–3.5m and derive scale; report assumed scale prominently in manifest.

## Failure Modes

- Top view is actually a roof plan (parapets hide footprint) → footprint overestimated; check if side-view base width < plan width → use side-view base as footprint authority.
- Complex plans (L-shapes, courtyards) → skeletonization may merge walls; fallback to contour-offset method: outer contour minus inner contours, thicken boundary inward.
- Curved buildings → treat as hard-surface loft along plan curve; radial sections from elevation views.
