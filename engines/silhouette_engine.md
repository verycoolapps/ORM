# Silhouette Engine

Extracts and processes 2D object masks from reference images; provides projection queries used by geometry and validation engines.

## Responsibilities

- Foreground segmentation → binary mask per view.
- Mask cleanup: largest component, hole filling, boundary smoothing.
- Distance-transform utilities for chamfer metrics.
- Projection functions `project(point_3d) -> (view, u, v)` and `inside_mask(view, u, v)`.

## Inputs

- Image + background hint from `reference_analyzer`.
- Calibration (pixel↔world mapping) from `alignment_engine`.

## Outputs

- `mask_v`: binary numpy array per view, stored as Blender image (`colorspace Non-Color`, single channel packed into R).
- `boundary_v`: ordered boundary pixel list (subsampled, ~256 points) for chamfer computation.
- Projection callables used by `geometry_engine` (voxel carving) and `validation_engine`.

## Execution Logic

### 1. Segmentation

```python
if background == "uniform":
    gray = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)
    _, mask = cv2.threshold(gray, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)
    # invert if object darker than bg: compare mean intensity inside border vs center
elif background == "alpha":
    mask = (alpha_channel > 128).astype(np.uint8) * 255
else:  # complex
    bgd = border_seed_points; fgd = center_seed
    mask = cv2.grabCut(img, None, rect_or_mask, bgd, fgd, 5, cv2.GC_INIT_WITH_MASK)
```

Post-process:

```python
mask = keep_largest_component(mask)
mask = cv2.morphologyEx(mask, cv2.MORPH_CLOSE, kernel_3x3, iterations=2)
mask = fill_holes(mask)
```

### 2. Boundary Extraction

```python
contours, _ = cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_NONE)
boundary = max(contours, key=cv2.contourArea).reshape(-1, 2)
boundary = resample_polyline(boundary, 256)  # arc-length uniform
```

### 3. World ↔ Pixel Projection

Given calibration `s_v` (world units per pixel), bbox offset, and axis map:

```python
def world_to_px(view, P):
    # P in world units relative to calibrated origin
    if view == "front": u, v = P.x, -P.z   # per axis map
    elif view == "side":  u, v = -P.y, -P.z
    else:                 u, v = P.x, -P.y
    px = (u / s_v + cx_v, v / s_v + cy_v)   # cx, cy = calibrated center in px
    return px

def px_to_world(view, px):  # inverse
    ...
```

`inside_mask(view, px)` returns True if pixel coordinate falls inside mask (bilinear sample of signed distance or direct lookup with 0.5px tolerance).

### 4. Silhouette of Mesh (for validation)

```python
def mesh_silhouette(obj, cam):
    # render depth+object-index pass, or project all verts & rasterize
    # preferred: bpy render with film_transparent, object index pass
    verts_2d = [world_to_camera_view(cam, v.co) for v in mesh.vertices]
    hull_or_boundary = alpha_shape_or_concave_hull(verts_2d)
    rasterize to mask-sized array
```

For concave silhouettes, vertex projection alone fails — use rendered object-index pass instead. Implement `render_silhouette(cam, resolution)` via offscreen render to `Render Result` → pixels → threshold by object index == target.

### 5. Metrics Utilities

```python
def iou(a, b): return np.logical_and(a,b).sum() / np.logical_or(a,b).sum()
def chamfer(b1, b2):  # boundary point sets in px
    d1 = cKDTree(b2).query(b1)[0]; d2 = cKDTree(b1).query(b2)[0]
    return (d1.mean() + d2.mean()) / 2
def hausdorff95(b1, b2): ...  # 95th percentile of directed distances
```

## Error Handling

- Segmentation yields < 100 px object → likely failed; try alternate threshold/inversion; if still tiny → error.
- Multiple large components → keep largest, warn about discarded area %; if discarded > 20% → flag `segmentation_ambiguous`.
- Mask touches image border → warn `possibly_cropped`.

## Dependencies

Used by `alignment_engine`, `geometry_engine`, `validation_engine`.
