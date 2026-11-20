# Feng Shui Geometry Module (Optional, Advisory Only)

**Status: OPTIONAL.** This module provides advisory analysis only. It never modifies geometry, never blocks delivery, and never overrides engineering requirements in `references/` or `workflows/`. A model that fails feng shui checks is still a valid deliverable; a model that fails engineering gates is never excused by feng shui compliance.

## Purpose

For users who want spatial-harmony considerations alongside reconstruction (e.g., architectural or product placements), ORTHO GENESIS can report advisory metrics about the reconstructed geometry relative to traditional feng shui concepts. Output is a report section, not a gate.

## Scope

Applies only when:

- Object class is `architectural` or `hard_surface` (large objects with spatial presence).
- User sets `"feng_shui_review": true` in run parameters.
- A site context is provided (plot orientation or adjacent structure positions) — without context, only intrinsic shape metrics are reported.

## Advisory Metrics

### 1. Form Classification (八卦 Bagua shape tendencies)

Classify dominant silhouette geometry into traditional shape categories:

- **Tall-narrow** (height > 1.5× min(width,depth)): "tower/needle" tendency — traditionally associated with pressure/instability in some schools; other schools value upward aspiration. Report neutrally: `form: "vertical"`.
- **Wide-low** (width > 2× height): "lake/grounding" tendency — `form: "horizontal_spread"`.
- **Balanced cube-ish** (all dims within 30%): `form: "balanced"`.
- **L-shaped / irregular**: `form: "irregular"` — flag that irregular forms are interpreted divergently across schools.

Report the classification; do NOT score it good/bad.

### 2. Proportion Check (San He / harmonic ratios)

Compute ratio `r = height / sqrt(footprint_area)`. Traditional references often cite ~0.5–1.0 as "harmonious" for buildings, but this is culturally variable. Report the number and the ratio band; no pass/fail.

### 3. Facing & Entry (orientation)

If site context gives a front direction (e.g., street side from reference placement), report:

- Which compass direction the object's front faces (needs site north — ask user or skip).
- Whether entry side is unobstructed in the plan (check plan mask for protrusions blocking the entry axis).

Without site north, report only relative orientation ("front faces −Y in scene").

### 4. Sharp Cornents & Sha Qi (煞气) Tendencies

Detect acute convex corners on plan silhouette (< 60° interior angle). Traditional practice associates sharp protruding corners aimed at entries with "sha qi" (sha = killing energy). Advisory output:

```
sha_qi_candidates: [{corner_xy, angle_deg, faces_entry: bool}]
```

Mitigation suggestions (rounded corner, vegetation buffer) are advisory text only — ORTHO GENESIS does not modify geometry for this.

### 5. Five Elements Material Hints (optional)

If materials are assigned, map material classes to elements for users interested in the framework:

| Material class | Element |
|---|---|
| Glass/metal | Metal |
| Wood/timber | Wood |
| Stone/concrete | Earth |
| Fabric/upholstery | Fire-adjacent (textile) |
| Water features | Water |

Report element presence counts. Missing element → advisory note only ("no Water-element material present"), not a defect.

## Output Format

Append to `validation_report.json`:

```json
"feng_shui_advisory": {
  "enabled": true,
  "form": "balanced",
  "proportion_ratio": 0.73,
  "sha_qi_candidates": [],
  "element_counts": {"metal": 2, "earth": 1},
  "notes": ["advisory only — no engineering impact"]
}
```

Or standalone `feng_shui_report.md` if requested.

## Hard Boundaries

- NEVER changes mesh geometry, proportions, or placement to satisfy feng shui.
- NEVER appears in `gates` or affects `overall` status.
- NEVER uses absolute terms ("auspicious", "inauspicious", "must", "forbidden") in output — descriptive terms only.
- Schools of feng shui disagree; the module reports measurements and traditional associations, leaving interpretation to the user and their consultant.

## Implementation Note

All metrics are pure geometry reads (bbox, plan polygon angles, material slot names) — no simulation, no scoring model, no AI judgment. ~50 lines of Python at most.
