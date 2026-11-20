# Production Delivery Standard

Final packaging, export, and handoff procedures for ORTHO GENESIS runs.

## Deliverable Package Structure

```
<run_name>/
├── <object_name>.blend          # master scene, all collections
├── exports/
│   ├── <object_name>.obj
│   ├── <object_name>.fbx
│   ├── <object_name>.stl         # only if requested & watertight
│   └── <object_name>.glb
├── previews/
│   ├── preview_front.png
│   ├── preview_side.png
│   ├── preview_top.png
│   ├── turntable_*.png           # if requested
│   └── diff_*.png                # reference vs render comparisons
├── reports/
│   ├── validation_report.json
│   ├── run_log.json
│   ├── feng_shui_report.md       # only if requested
│   └── reconstruction_notes.md   # human-readable summary
└── manifest.json
```

## Export Specifications

### .blend

- Save with `compress=True` (blend file compression).
- Collections intact: `REF_VIEWS` (hidden in render), `BLOCKOUT` (kept for iteration), `MESH` (final), `PREVIEW`, `EXPORT`.
- `scene["og_run_log"]` and `scene["og_calibration"]` custom properties preserved.
- Relative paths off; textures packed if any (`bpy.ops.file.pack_all()`).

### .obj

```python
bpy.ops.export_scene.obj(
    filepath=..., use_selection=True,
    use_mesh_modifiers=True, use_materials=True,
    path_mode='COPY'  # embed textures if any
)
```

Select only final mesh objects (exclude cameras, empties, reference planes).

### .fbx

```python
bpy.ops.export_scene.fbx(
    filepath=..., use_selection=True,
    object_types={'MESH'},
    apply_scale_options='FBX_SCALE_NONE',
    mesh_smooth_type='FACE',
    use_mesh_modifiers=True,
    add_leaf_bones=False,
    bake_anim=False
)
```

### .stl

```python
bpy.ops.export_mesh.stl(filepath=..., export_selected_objects=True,
                        apply_modifiers=True, ascii_format=False)
```

Prerequisites: Gate-3 watertight must pass. STL has no units — if scene unit is meter and object is building-scale (> 100m), warn user that slicers expect mm; auto-scale option `scale_to_mm: true` multiplies coordinates by 1000.

### .glb (glTF binary)

```python
bpy.ops.export_scene.gltf(filepath=..., use_selection=True,
                          export_format='GLB', export_apply=True,
                          export_yup=True)
```

Y-up conversion automatic. Materials export with base color factors.

## Manifest Format

```json
{
  "package": "orthogenesis_export",
  "version": "1.0",
  "run_id": "...",
  "created": "ISO-8601",
  "blender_version": "x.y.z",
  "object_class": "hard_surface",
  "units": "meters",
  "dimensions_world": [x, y, z],
  "quality_tier": "standard",
  "validation_overall": "pass|fail|degraded",
  "files": [{"path": "...", "bytes": 0, "sha256": "..."}],
  "limitations": ["..."],
  "feng_shui_review": false
}
```

`limitations` must list every known compromise: missing views, relaxed gates taken, assumed dimensions, skipped UV detail, etc.

## Export Validation

After writing each file:

1. **.obj/.fbx/.glb**: reload via `trimesh.load()` (or `pygltflib` for glb); verify face count within 1% of scene evaluated mesh and bbox dims match within 0.1%.
2. **.stl**: reload, verify watertight via trimesh `is_watertight`.
3. **.blend**: reopen check optional (same-session save is reliable); verify file size > 10KB.

Any reload failure → mark that file `status: "export_failed"` in manifest, do not silently ship corrupt files.

## Naming & Hygiene

- Object names: `{project}_{object}_final` (e.g., `acme_vase_final`). No spaces, no special chars.
- File names lowercase snake_case.
- Remove temp files: voxel intermediates, debug renders not in `previews/`.
- If `quality_tier == draft`, manifest sets `"draft": true` — recipients know it's blockout-grade.

## Reconstruction Notes (reconstruction_notes.md)

Human-readable summary, ≤ 1 page:

- What was reconstructed, from how many views, which class/method.
- Key assumptions (scale source, symmetry, missing detail).
- Validation headline numbers (IoU per view, face count).
- Known limitations & suggested next steps (manual detailing, UV work, texture baking).

Written in English. No marketing language.

## Delivery Checklist

- [ ] All requested formats exported and reload-verified.
- [ ] `validation_report.json` present with `overall` field.
- [ ] `manifest.json` complete with hashes.
- [ ] `limitations` array non-empty if any compromise existed.
- [ ] `.blend` opens with collections and run log intact.
- [ ] No placeholder objects, no `Cube.001`-style orphan meshes in final collection.
- [ ] Scale applied; units metric; dimensions match calibration.

A package missing any checklist item is `degraded`, not `pass`.
