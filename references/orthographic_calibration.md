# Orthographic Calibration

Establishing shared scale, axes, and ground plane across front/side/top reference views.

## Inputs

- One to three orthographic images (PNG/JPG/EXR), ideally 2K+ resolution.
- Optional: known real-world dimension for at least one feature (e.g., "total height = 180 cm").

## Calibration Procedure

### Step 1: View Classification

Classify each input as front, side, or top:

- Front: widest horizontal extent relative to vertical; shows primary face.
- Side: narrower horizontal extent, same vertical extent as front.
- Top: square-ish or horizontal aspect matching front width; shows plan outline.

If classification is ambiguous, run `engines/reference_analyzer.md` heuristics: compare aspect ratios and silhouette overlap after rotation candidates. Confidence below 0.7 → halt and request user labeling.

### Step 2: Foreground Segmentation

For each view, extract the object mask:

- If background is uniform: threshold via Otsu on grayscale, then largest connected component.
- If background is complex: use GrabCut seeded by border background pixels, or a provided alpha channel.
- Output: binary mask `M_v` per view at native resolution.

Store masks as `og_mask_front`, `og_mask_side`, `og_mask_top` image datablocks (non-color, single channel).

### Step 3: Bounding Box & Extents

For each mask compute:

```
bbox_v = (x_min, y_min, x_max, y_max)          # pixel coords
extent_u_v = x_max - x_min                      # horizontal pixels
extent_w_v = y_max - y_min                      # vertical pixels
```

### Step 4: Scale Resolution

Three cases:

**Case A — Known dimension provided.** Pick the view/axis matching the known dimension:

```
units_per_pixel_v = known_length / extent_u_v   # or extent_w_v for vertical dims
```

**Case B — Front + side views only.** Heights must agree:

```
units_per_pixel_front = extent_w_side_pixels_equiv …
```

Concretely: require `extent_w_front / units_pp_f == extent_w_side / units_pp_s`. Solve for the scale factor `k` between views by matching shared vertical extent; if user provides one absolute scale for either view, derive the other. If heights disagree beyond 3%, flag inconsistency.

**Case C — All three views.** Solve jointly: scale must satisfy front↔side height agreement AND front↔top width agreement. Set up two equations, solve least-squares for `(s_f, s_s, s_t)`.

### Step 5: Axis & Ground Plane Assignment

Define scene axes:

- Blender Z = world up = vertical axis of front and side views.
- Blender X = left-right of front view AND depth axis of side view.
- Blender Y = depth axis of front view AND left-right of side view (with sign flip: screen-right in side view maps to −Y if the side view shows the object's right side; determine via asymmetric features or default to +Y and document the choice).

Ground plane: bottom edge of front/side masks → `z = 0` unless a ground shadow/contact line is detected lower.

Top view: its image x-axis maps to scene X, image y-axis maps to scene Y (screen-down in image = −Y or +Y depending on convention; fix once and document).

### Step 6: Centering & Offset

Compute each mask's centroid and bbox center. Align all views to a common origin: typically the ground-contact centroid in X/Y. Record offsets `(ox_v, oy_v)` for reprojecting rendered previews back to image space during validation.

### Step 7: Calibration Artifact

Write calibration into the .blend:

```python
scene["og_calibration"] = json.dumps({
    "units_per_pixel": {"front": s_f, "side": s_s, "top": s_t},
    "image_size": {"front": (w_f, h_f), ...},
    "bbox_px": {...},
    "offsets_px": {...},
    "axis_map": {"front": {"img_x": "+X", "img_y": "-Z"}, ...},
    "ground_z": 0.0,
    "confidence": 0.0-1.0,
})
```

Attach reference images as camera-background or empty-image planes in collection `REF_VIEWS`, positioned at the correct orthographic depth but NEVER parented to the model.

## Inconsistency Handling

If computed extents disagree beyond tolerance (default 3% of the shared dimension):

1. Re-run segmentation with alternate thresholds.
2. If still disagreeing, report the discrepancy table and halt. Do not average silently.
