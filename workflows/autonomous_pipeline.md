# Autonomous Pipeline Workflow

Default end-to-end execution path for ORTHO GENESIS.

## Trigger

User supplies orthographic image(s) + optional parameters:

```
{ "images": [...], "object_class_hint": "auto|hard_surface|organic|architectural",
  "known_dimension": {...}?, "deliverables": ["blend","obj","fbx","stl","glb"],
  "quality_tier": "draft|standard|final" }
```

## Stage Sequence

### Stage 0 — Intake & Setup

1. Create collections: `REF_VIEWS`, `BLOCKOUT`, `MESH`, `PREVIEW`, `EXPORT`.
2. Record Blender version, timestamp, input hashes → `scene["og_run_log"]`.
3. Determine `quality_tier` presets:

| Tier | Voxel res | Subdiv | Render samples | Iteration budget |
|---|---|---|---|---|
| draft | size/64 | none | 16 EEVEE | 1 |
| standard | size/96 | lvl1 opt | 64 EEVEE | 2 |
| final | size/128 | per class | 64 EEVEE + 1 Cycles64 | 3 |

### Stage 1 — Reference Analysis

Run `engines/reference_analyzer.md`. Outputs classified views. If `needs_user_label` → pause, ask user to label front/side/top, resume.

### Stage 2 — Segmentation & Calibration

1. `silhouette_engine.extract_mask` per view.
2. `alignment_engine` solves scales/axes → calibration JSON + cameras.
3. If `status == "inconsistent_views"` → halt with discrepancy report.

### Stage 3 — Classification

`object_classification.md` decision tree → class + confidence. Confidence < 0.6 → ask user; else proceed with flagged hybrid if needed.

### Stage 4 — Geometry Reconstruction

`geometry_engine` dispatches method, iterates up to budget on silhouette IoU. Produces `BLOCKOUT` mesh(es).

### Stage 5 — Topology Enforcement

`topology_engine` → clean `MESH` object.

### Stage 6 — Modifiers & Shading Prep

`modifier_engine` per class preset + delivery spec.

### Stage 7 — Validation

`validation_engine` computes all gates → `validation_report.json`.

- Pass → Stage 8.
- Fail + attempts remaining → return to Stage 4 with adjusted params (method switch allowed once: loft→voxel or vice versa).
- Fail + exhausted → Stage 8 still runs exports but marks package `status: "degraded"` with report attached; user decides.

### Stage 8 — Rendering Previews

`rendering_engine` → preview PNGs + optional turntable.

### Stage 9 — Delivery

`standards/production_delivery.md` → exports + manifest.

## Decision Points Summary

| Point | Condition | Action |
|---|---|---|
| After Stage 1 | confidence < 0.6 | ask user for view labels |
| After Stage 2 | extent disagreement > 3% | halt, report inconsistency |
| After Stage 3 | class confidence < 0.6 | ask user or default hard-surface |
| After Stage 7 | gates fail, attempts < 3 | loop to Stage 4 |
| After Stage 7 | gates fail, attempts = 3 | export degraded + report |

## Run Log Requirements

Every stage appends `{stage, timestamp, status, metrics}` to `scene["og_run_log"]`. Final log exported as `run_log.json` alongside deliverables.

## Time Budget Guidance

- Draft: target < 2 min on ≤ 1M-voxel problems.
- Standard: < 10 min.
- Final: < 30 min (voxel 128³ + remesh + 3 iterations can be heavy; warn user if estimated voxels > 4M).
