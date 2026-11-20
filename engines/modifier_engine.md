# Modifier Engine

Applies and configures Blender modifiers and shading preparation for final output.

## Responsibilities

- Bevels, weighted normals / smooth-by-angle, subdivision surface (where class-appropriate).
- UV unwrap preparation (smart UV project or lightmap pack).
- Optional material slots: clay material for validation renders.
- Apply-or-keep decision per delivery spec.

## Inputs

- Clean mesh from `topology_engine`.
- Object class parameters (bevel width, segments, subdiv level).
- Delivery spec from `standards/production_delivery.md` (applied vs. live modifiers).

## Outputs

- Final mesh object(s) in `MESH` collection.
- UV layer(s) named `UVMap` (and `UVMap_Lightmap` if requested).
- Material slot `M_Clay_Validation` (neutral gray, roughness 0.9) for preview renders.

## Execution Logic

### 1. Bevel (hard-surface / architectural)

```python
bev = obj.modifiers.new("Bevel", 'BEVEL')
bev.width = bevel_width          # default 1% of min bbox dim, clamp [0.001, 0.05] * min_dim
bev.segments = 2                 # 2 for hard-surface; 3 only if class demands highlight roll
bev.limit_method = 'ANGLE'
bev.angle_limit = math.radians(30)
bev.harden_normals = True
```

Bevel must come AFTER confirming supporting edge loops exist (topology engine's job). If bevel produces artifacts (overlap on thin geometry < 2×bevel_width), reduce width to 0.5× and flag `thin_geometry_bevel`.

### 2. Normals

```python
# Blender 4.1+: use Smooth by Angle modifier or set shade_smooth_by_angle
mod = obj.modifiers.new("SmoothByAngle", 'NODES')  # or use obj.data.set_sharp_from_angle if available
# Fallback: for poly in obj.data.polygons: poly.use_smooth = True; then set sharp edges where dihedral > 40°
```

Organic meshes: full smooth shading. Hard-surface: smooth-by-angle ~35–45°. Architectural: flat shading except explicitly curved elements.

### 3. Subdivision (optional)

Only when class preset requests it AND base mesh density already meets minimum (subdiv multiplies faces ~4× per level; check against max faces budget).

```python
sub = obj.modifiers.new("Subsurf", 'SUBSURF')
sub.levels = 1; sub.render_levels = 2
```

### 4. Weighted Normal alternative

If Bevel+SmoothByAngle insufficient, add WeightedNormal modifier after bevel with `keep_sharp=True, weight=50`.

### 5. UV Preparation

```python
bpy.ops.object.mode_set(mode='EDIT')
bpy.ops.uv.smart_project(angle_limit=math.radians(66), island_margin=0.02)
bpy.ops.object.mode_set(mode='OBJECT')
```

For architectural: prefer per-face planar mapping (`cube_project`) since walls are axis-aligned. For organic: smart_project acceptable at blockout stage; note in report that final UVs need manual/texture-space work — ORTHO GENESIS guarantees *unwrap-ready* topology, not production UVs.

### 6. Clay Material

```python
mat = bpy.data.materials.new("M_Clay_Validation")
mat.use_nodes = True
bsdf = mat.node_tree.nodes["Principled BSDF"]
bsdf.inputs["Base Color"].default_value = (0.5, 0.5, 0.5, 1)
bsdf.inputs["Roughness"].default_value = 0.9
bsdf.inputs["Metallic"].default_value = 0.0
```

Assign to a slot; validation renders override world to neutral gray.

### 7. Apply vs. Keep

Read delivery spec:

- `apply_modifiers: true` → apply bevel/subdiv/weighted-normal before export (except keep Subsurf render levels if spec says `subdiv_render_only`).
- `apply_modifiers: false` → leave live; exporters must use `use_mesh_modifiers=True` (FBX/OBJ options) so exported mesh includes them.

Record decision in run log.

## Error Handling

- Modifier stack conflicts (e.g., bevel before remesh) → enforce canonical order: Remesh → Bevel → WeightedNormal/SmoothByAngle → Subsurf.
- UV unwrap failure on non-manifold → shouldn't happen post-topology; if it does, log and continue (UV is Gate-4-adjacent, not blocking).

## Dependencies

After `topology_engine`; before `validation_engine` and export.
