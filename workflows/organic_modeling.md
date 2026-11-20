# Organic Modeling Workflow

Specialized path for characters, creatures, anatomical objects, plants.

## Differences from Autonomous Pipeline

- Method forced to voxel carve → remesh → retopo (lofting rarely suits organic silhouettes).
- Symmetry handling: bilateral characters get mirror-built half; asymmetric creatures build whole.
- Additional stage: anatomical landmark placement (optional, user-provided or auto-detected from silhouette protrusions).

## Stage Adaptations

### Stage 3 — Classification Confirmation

Organic class requires: smooth silhouette curvature, no dominant planar regions, symmetry_score typically < 0.7 unless humanoid.

### Stage 4 — Geometry (Organic Variant)

1. Voxel carve at `size/128` (final tier) or `size/96` (standard).
   - Use all available views; missing side view → carve with front+top only, note reduced depth accuracy.
2. Marching cubes → dense tri mesh (~2–5M tris at 128³ for human-scale).
3. **Remesh**: Blender Remesh modifier voxel mode, `voxel_size = target_edge × 0.8` where `target_edge = smallest_anatomical_feature / 4` (e.g., finger width / 4 for hands; for full body use shoulder-width / 20 as coarse target then refine).
4. **Quadriflow/quad remesh** to ~50k–200k quads depending on tier.
5. **Landmark detection** (optional): protrusion analysis on silhouette — local maxima of boundary curvature distance; place empty markers `LM_*` for user reference. Never auto-add geometry at landmarks without user confirmation.

### Stage 5 — Topology (Organic Rules)

- Face poles ONLY at: top of head, fingertips, toes, tail tip, wing tips — valence ≤ 5.
- Edge loops follow major muscle groups / spine curve.
- After retopo, run `bmesh.ops.smooth_vert` (lambda 0.5, 10 iterations, use_topology_dist) to even valence — do NOT smooth positions enough to break silhouette (verify IoU doesn't drop > 0.01).

### Stage 6 — Modifiers (Organic)

- NO bevel modifier (organic surfaces don't bevel).
- Smooth shading 100%; optional Subsurf level 1 if base < 30k quads and tier = final.
- Multiresolution modifier optional for user sculpting: add `MULTIRES` with `subdivision_type='SIMPLE'`, levels = 2, so user can sculpt detail without destroying base.

### Stage 7 — Validation (Organic Gates)

Same gates, but:

- Quad ratio threshold relaxed to 0.85 pre-subdiv.
- Silhouette IoU threshold same (0.92) — organic shapes demand accuracy too.
- Additional check: slice-volume profile — compare mesh cross-section area at 10%,30%,...,90% height vs silhouette-implied areas; deviation > 20% flags underdetermined region.

### Stage 8 — Delivery

Organic models commonly need `.blend` (for sculpt continuation) + `.fbx`/`.glb` (for engines). Include Multires if present. Warn if mesh > 500k faces for game delivery — suggest decimation plan in report.

## Symmetry Workflow (humanoid/bilateral)

1. Build left half only (voxel carve clipped to x ≤ 0 plane, or loft half-sections).
2. Mirror via `Mirror` modifier (merge threshold 0.001) OR apply scale -1 on duplicated half then `remove_doubles`.
3. Verify mirrored silhouette still matches BOTH front and side views (asymmetry in refs → disable mirror, rebuild whole).

## Failure Modes Specific to Organic

- Voxel carve produces disconnected parts (thin connecting regions vanish) → reduce voxel size 25%, recarve; or add "bridge" voxels manually along detected connection axis.
- Retopo destroys thin features (fingers, ears) → locally increase target edge density before quadriflow; quadriflow respects no guides, so pre-decimate thin regions less aggressively.
- Self-intersection at armpit/crotch → voxel remesh smooths these; if persists, local smooth + shrinkwrap to voxel surface.
