# Reference Analyzer Engine

First-stage engine: ingest, classify, and normalize orthographic reference images.

## Responsibilities

- Accept image paths or Blender image datablocks.
- Classify each image as front / side / top view.
- Detect missing views and ordering ambiguities.
- Compute per-image quality metrics (resolution, aspect, background uniformity).
- Produce normalized inputs for the calibration engine.

## Inputs

- `images`: list of `{path | bpy.types.Image, label_hint?}`.
- `expected_views`: optional list, e.g., `["front","side","top"]`.

## Outputs

```python
{
  "views": {
    "front": {"image": ..., "mask_hint": "uniform_bg|complex_bg|alpha", "resolution": (w,h)},
    "side":  {...},
    "top":   {...},
  },
  "missing": ["top"],
  "classification_confidence": 0.0-1.0,
  "warnings": [...],
}
```

## Execution Logic

### 1. Ingest

Load each image; if EXR/HDR, tonemap a copy for analysis but keep original for masking. Record SHA-256 hash into run log.

### 2. View Classification Heuristic

For each image compute:

- `aspect = w / h`
- `edge_density` = Canny edge pixels / total pixels.
- `symmetry_lr` = IoU(mask_or_luminance, horizontal flip) — needs rough mask; use luminance threshold fallback.

Rules:

- Front & side candidates: `aspect > 0.8` typically (portrait-ish or square). Distinguish by comparing the two: the view whose horizontal silhouette extent is larger when both normalized to same height = front (primary face is widest). If only one non-top candidate exists and a second exists, compute width ratio; ratio > 1.15 → wider is front.
- Top candidate: image whose aspect best matches the front view's aspect when both represent the same horizontal plane — top view typically has `aspect ≈ front_aspect` and lowest edge density (plan views have less visual detail) OR contains room-like rectangular structure.
- If a view is labeled by user (`label_hint`), trust it but verify consistency.

### 3. Background Detection

Sample border pixels (outer 2% frame):

- Std-dev < 8/255 → uniform background → thresholding viable.
- Else → complex background → flag `complex_bg`, plan GrabCut or alpha extraction.
- If image has alpha channel with meaningful coverage → `alpha`.

### 4. Quality Gate

Reject or warn when:

- Resolution < 512 px on longest side → warn (accuracy degrades).
- Aspect ratio extreme (> 3:1) → likely cropped; warn.
- All views same image → error.

## Error Handling

- Unreadable image → skip with warning; if < 2 views remain, halt.
- Classification confidence < 0.6 → return `needs_user_label: true` with per-image scores; pipeline pauses for labels.

## Dependencies

Runs before `engines/alignment_engine.md`. Provides masks-ready images to `engines/silhouette_engine.md`.
