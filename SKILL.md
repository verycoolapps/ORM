---
name: orthogenesis
description: Autonomous orthographic three-view (front/side/top) image-to-3D reconstruction for Blender. Calibrates orthographic reference images, extracts silhouettes, reconstructs geometry via cross-section/voxel methods, enforces topology standards, applies modifiers, validates results, and delivers .blend/.obj/.fbx/.stl/.glb exports. Use when converting orthographic designs (product renders, concept sheets, blueprint views) into accurate 3D models.
---

# ORTHO GENESIS

Autonomous orthographic image-to-3D reconstruction engine for Blender.

## Mission

Given one or more orthographic images — front, side, top — reconstruct a parametric, topology-clean 3D model inside Blender, validate it against the source views, and export production assets.

## When to Use

- Reconstructing 3D models from orthographic concept art, turntable renders, or technical drawings.
- Converting product silhouette sheets into editable Blender meshes.
- Rebuilding hard-surface, organic, or architectural objects where three-view orthographic references exist.

Do NOT use for:
- Arbitrary perspective photo reconstruction (use photogrammetry or NeRF workflows instead).
- Single-view 2.5D generation without depth cues.

## Pipeline Overview

1. **Reference intake** — classify inputs, verify orthographic validity, detect inconsistencies across views.
2. **Calibration** — establish shared scale, ground plane, and axes between views.
3. **Silhouette extraction** — segment foreground, derive 2D masks per view.
4. **Geometry reconstruction** — cross-section lofting, voxel carving, or hybrid methods depending on object class.
- `references/` — modeling principles, calibration math, reconstruction algorithms, topology standards, object classification, quality standards, known limitations.
- `engines/` — implementation modules: reference analyzer, alignment, silhouette extraction, geometry, topology, modifiers, validation, rendering.
- `workflows/` — end-to-end execution paths for autonomous, organic, hard-surface, and architectural modeling.
- `standards/` — feng shui geometry advisory module and production delivery checklist.

## Execution Contract

Every reconstruction run MUST produce:
- A calibrated scene with named collections (`REF_VIEWS`, `BLOCKOUT`, `MESH`, `PREVIEW`).
- A validated mesh passing the quality gate in `references/quality_standards.md`.
- A comparison report: silhouette IoU per view, Hausdorff distance, volume sanity check.
- Exported deliverables per `standards/production_delivery.md`.

## Hard Rules

- Never fabricate metrics. Every quality number must be computed from actual geometry and images.
- Orthographic consistency is mandatory: if views disagree, stop and report ambiguity rather than guessing.
- The feng shui module (`standards/feng_shui_geometry.md`) is optional and advisory only; it never overrides engineering requirements.

## Originality & Provenance

ORTHO GENESIS is an original work authored for VeryCoolApps. All documentation in this skill package is original prose written for this system; it does not copy or paraphrase third-party products, documentation, or datasets.
