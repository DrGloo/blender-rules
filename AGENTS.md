# Blender MCP Agent Guidelines

Rules and standards for AI agents working with Blender through the Model Context Protocol (MCP).

---

## Core Principles

1. **Inspect before you act.** Always call `get_scene_info` or `get_object_info` before modifying the scene.
2. **Incremental execution.** Break Python code into small, focused chunks — never send one massive script.
3. **Verify visually.** After any meaningful change, call `get_viewport_screenshot` to confirm the result.
4. **Name everything.** Every object, material, and collection must have a clear, descriptive name.
5. **Non-destructive first.** Prefer modifiers and node-based workflows over direct mesh edits when possible.

---

## Tool Usage Order

Follow this sequence when starting any Blender task:

```
1. get_scene_info          → Understand what exists
2. get_object_info         → Inspect specific targets
3. execute_blender_code    → Make changes (small steps)
4. get_viewport_screenshot → Verify the result
5. Repeat 2–4 as needed
```

Never skip step 1. Never batch unrelated operations into a single `execute_blender_code` call.

---

## Python Scripting Standards

### Script Structure

Every `execute_blender_code` call should:

- Import only what it needs at the top
- Target a single, well-defined operation
- Include error handling for object lookups
- Print confirmation of what was done

### Naming Conventions

| Element       | Convention                  | Example                    |
|---------------|-----------------------------|----------------------------|
| Objects       | PascalCase, descriptive     | `WoodenTable`, `FloorPlane`|
| Materials     | `MAT_` prefix + PascalCase  | `MAT_BrushedSteel`        |
| Collections   | PascalCase, grouped by role | `Lighting`, `Environment`  |
| Node Groups   | `NG_` prefix + PascalCase   | `NG_WoodGrain`            |
| Modifiers     | PascalCase, action-based    | `BevelEdges`, `SubdivSmooth`|

### Error Handling

Always guard object lookups:

```python
import bpy

obj = bpy.data.objects.get("TargetName")
if obj is None:
    raise ValueError("Object 'TargetName' not found in scene")
```

Never use bare `bpy.data.objects["Name"]` — it throws a `KeyError` with no context.

### Cleanup After Operations

```python
# Deselect everything before starting
bpy.ops.object.select_all(action='DESELECT')

# Always set the active object explicitly
bpy.context.view_layer.objects.active = obj
obj.select_set(True)
```

### Avoid These Patterns

- **No hardcoded file paths** — use `bpy.path` utilities or ask the user
- **No `bpy.ops` when a direct data API exists** — prefer `bpy.data` for creation and manipulation
- **No sleep/polling in Blender scripts** — Blender executes synchronously
- **No wildcard imports** — always `import bpy`, never `from bpy import *`

---

## Scene Management

### Before Modifying the Scene

1. Call `get_scene_info` to catalog existing objects, materials, and collections
2. Identify naming conflicts before creating new objects
3. Check the active camera and render settings if the task involves rendering

### Object Creation Workflow

```python
import bpy

# Create mesh and object with descriptive names
mesh = bpy.data.meshes.new("CoffeeTableMesh")
obj = bpy.data.objects.new("CoffeeTable", mesh)

# Link to appropriate collection (not just the scene)
collection = bpy.data.collections.get("Furniture")
if collection is None:
    collection = bpy.data.collections.new("Furniture")
    bpy.context.scene.collection.children.link(collection)
collection.objects.link(obj)

# Set as active and select
bpy.context.view_layer.objects.active = obj
obj.select_set(True)
```

### Organizing with Collections

Group related objects into collections:

| Collection Name | Contents                              |
|-----------------|---------------------------------------|
| `Environment`   | Ground planes, sky, background        |
| `Lighting`      | All lights and light rigs             |
| `Furniture`     | Tables, chairs, shelves               |
| `Props`         | Small decorative objects              |
| `Cameras`       | Camera objects and empties for tracks |

---

## Material and Texture Workflows

### Creating Materials

```python
import bpy

mat = bpy.data.materials.new(name="MAT_OakWood")
mat.use_nodes = True
nodes = mat.node_tree.nodes
links = mat.node_tree.links

# Clear defaults
for node in nodes:
    nodes.remove(node)

# Build node tree explicitly
output = nodes.new(type='ShaderNodeOutputMaterial')
bsdf = nodes.new(type='ShaderNodeBsdfPrincipled')
links.new(bsdf.outputs['BSDF'], output.inputs['Surface'])
```

### Polyhaven Assets

Before using Polyhaven tools, check integration status:

```
1. get_polyhaven_status       → Confirm availability
2. get_polyhaven_categories   → Browse what's available
3. search_polyhaven_assets    → Find specific assets
4. download_polyhaven_asset   → Import into Blender
5. set_texture                → Apply to target object
```

**Resolution guidance:**
- Previews / drafts: `1k`
- Standard work: `2k`
- Final renders: `4k`

Always download before applying. The `set_texture` tool requires the asset to be downloaded first.

---

## 3D Model Import Workflows

### Sketchfab Models

```
1. get_sketchfab_status           → Confirm availability
2. search_sketchfab_models        → Find candidates
3. get_sketchfab_model_preview    → Visually confirm before downloading
4. download_sketchfab_model       → Import with appropriate target_size
```

**Target size reference (in meters):**

| Object         | target_size |
|----------------|-------------|
| Coffee cup     | 0.12        |
| Chair          | 0.9         |
| Person         | 1.75        |
| Desk           | 0.75        |
| Car            | 4.5         |
| Building       | 10.0+       |

Always preview a model before downloading. Always specify a realistic `target_size`.

### Hyper3D Rodin (AI-Generated Models)

```
1. get_hyper3d_status                    → Confirm availability
2. generate_hyper3d_model_via_text       → Submit generation request
   OR generate_hyper3d_model_via_images
3. poll_rodin_job_status                 → Wait for completion
4. import_generated_asset                → Bring into scene
```

**Polling strategy:**
- Wait 10–15 seconds between polls
- Check at least 3 times before reporting a timeout
- Only proceed when all statuses return `"Done"`

**Text prompts:** Be specific and descriptive. "A wooden medieval shield with iron rivets" is better than "a shield."

**Bounding box hints:** Use `bbox_condition` when proportions matter:
- Tall narrow object: `[1, 1, 3]`
- Flat wide object: `[3, 3, 1]`
- Cube-like object: `[1, 1, 1]`

### Hunyuan3D (AI-Generated Models)

```
1. get_hunyuan3d_status              → Confirm availability
2. generate_hunyuan3d_model          → Submit with text, image, or both
3. poll_hunyuan_job_status           → Wait for "DONE"
4. import_generated_asset_hunyuan    → Import using the zip_file_url
```

Same polling strategy as Hyper3D. Hunyuan supports combined text + image input for better results.

---

## Roblox Export Standards

All models and maps built in Blender are destined for Roblox. Every modeling decision must respect these constraints.

### Hard Limits

| Constraint                  | Limit                     |
|-----------------------------|---------------------------|
| Meshes per import           | < 200                     |
| Triangles per MeshPart      | < 10,000                  |
| Texture resolution          | 1024x1024 max             |
| Materials per MeshPart      | 1 (one SurfaceAppearance) |
| Export format               | `.fbx`                    |

These are non-negotiable. The agent must check and enforce them before every export.

### Triangle Budget

- **Target 5,000–8,000 triangles** per mesh to leave headroom
- After every modeling step, print the triangle count:

```python
import bpy

obj = bpy.data.objects.get("ObjectName")
if obj is None:
    raise ValueError("Object 'ObjectName' not found")

depsgraph = bpy.context.evaluated_depsgraph_get()
evaluated = obj.evaluated_get(depsgraph)
mesh = evaluated.to_mesh()
mesh.calc_loop_triangles()
tri_count = len(mesh.loop_triangles)
evaluated.to_mesh_clear()
print(f"'{obj.name}' triangle count: {tri_count}")
```

- If a mesh exceeds 10,000 triangles, apply a Decimate modifier and report before/after count
- Always report total triangle count across all meshes when working on a map

### Decimate Strategy

Choose the right Decimate mode based on the geometry:

| Mode            | Best For                                          |
|-----------------|---------------------------------------------------|
| **Collapse**    | General-purpose poly reduction on any mesh        |
| **Un-Subdivide**| Meshes that were previously subdivided             |
| **Planar**      | Flat surfaces with unnecessary edges (common in AI-generated models) |

AI-generated models (Hyper3D, Hunyuan3D) almost always arrive over budget. Run Planar dissolve first to clean flat regions, then Collapse to hit the target count. Always print the count before and after:

```python
import bpy

obj = bpy.data.objects.get("GeneratedModel")
if obj is None:
    raise ValueError("Object 'GeneratedModel' not found")

# Print before count
depsgraph = bpy.context.evaluated_depsgraph_get()
evaluated = obj.evaluated_get(depsgraph)
mesh_eval = evaluated.to_mesh()
mesh_eval.calc_loop_triangles()
before = len(mesh_eval.loop_triangles)
evaluated.to_mesh_clear()

# Add Decimate modifier
mod = obj.modifiers.new(name="DecimatePlanar", type='DECIMATE')
mod.decimate_type = 'DISSOLVE'
mod.angle_limit = 0.087  # ~5 degrees

# Print after count
depsgraph = bpy.context.evaluated_depsgraph_get()
evaluated = obj.evaluated_get(depsgraph)
mesh_eval = evaluated.to_mesh()
mesh_eval.calc_loop_triangles()
after = len(mesh_eval.loop_triangles)
evaluated.to_mesh_clear()

print(f"'{obj.name}' triangles: {before} → {after}")
```

### Geometry Cleanup (Required Before Export)

Every mesh must pass these checks before export:

1. **Triangulate** — apply Triangulate modifier or `bpy.ops.mesh.quads_convert_to_tris()`. Roblox does not support quads or n-gons.
2. **Apply all transforms** — `bpy.ops.object.transform_apply(location=True, rotation=True, scale=True)`.
3. **Recalculate normals outward** — `bpy.ops.mesh.normals_make_consistent(inside=False)` in Edit Mode.
4. **Remove doubles** — merge by distance to eliminate overlapping vertices.
5. **Delete loose geometry** — remove floating vertices and edges.
6. **Remove interior faces** — delete hidden geometry (insides of walls, bottoms of placed furniture).

### Invisible Collision Space (Critical for Roblox)

Roblox MeshParts use their bounding box or collision mesh for physics. If a mesh has hidden/interior faces, offset origins, loose vertices extending beyond the visible surface, or was merged from distant objects with gaps between them, players will collide with "invisible" space — they'll walk on air or hit walls that aren't there.

**Root causes and fixes:**

| Cause | Fix |
|---|---|
| **Merged distant objects** | Never merge objects that aren't physically touching. Each formation/structure must be its own mesh so its bounding box tightly wraps only that geometry. |
| **Interior/hidden faces** | Delete faces trapped inside overlapping geometry (e.g., the bottom of a cylinder sitting on a cube). Use `Select Interior Faces` or manual selection. |
| **Loose vertices/edges** | Run `bpy.ops.mesh.delete_loose()` — stray vertices far from the surface silently expand the bounding box. |
| **Origin offset** | Reset origin to geometry center with `bpy.ops.object.origin_set(type='ORIGIN_GEOMETRY', center='BOUNDS')`. An origin far from the mesh shifts the collision box. |
| **Overlapping merged layers** | When merging stacked layers (e.g., mesa bands), use `Remove Doubles` / merge-by-distance aggressively to fuse shared vertices. |

**Aggressive cleanup script (run on every mesh before export):**

```python
import bpy

obj = bpy.data.objects.get("ObjectName")
if obj is None:
    raise ValueError("Object 'ObjectName' not found")

bpy.ops.object.select_all(action='DESELECT')
obj.select_set(True)
bpy.context.view_layer.objects.active = obj

# Apply all transforms so bounds are accurate
bpy.ops.object.transform_apply(location=True, rotation=True, scale=True)

bpy.ops.object.mode_set(mode='EDIT')
bpy.ops.mesh.select_all(action='SELECT')

# Remove doubles aggressively
bpy.ops.mesh.remove_doubles(threshold=0.05)

# Delete loose geometry (expands bounding box invisibly)
bpy.ops.mesh.delete_loose(use_verts=True, use_edges=True, use_faces=True)

# Delete interior faces (hidden geometry that inflates collision)
bpy.ops.mesh.select_all(action='DESELECT')
bpy.ops.mesh.select_interior_faces()
bpy.ops.mesh.delete(type='FACE')

# Recalculate normals
bpy.ops.mesh.select_all(action='SELECT')
bpy.ops.mesh.normals_make_consistent(inside=False)

bpy.ops.object.mode_set(mode='OBJECT')

# Reset origin to tight geometry center
bpy.ops.object.origin_set(type='ORIGIN_GEOMETRY', center='BOUNDS')

print(f"Aggressive cleanup done on '{obj.name}'")
```

**Merge rules to prevent invisible collision:**

- **Same formation, touching geometry** → safe to merge (e.g., stacked mesa layers)
- **Nearby but separated objects** → do NOT merge (e.g., two mesas 50 studs apart)
- **Rule of thumb:** if there's walkable space between two objects, they must be separate meshes

**Post-cleanup bounding box audit:**

```python
import bpy

for obj in bpy.data.objects:
    if obj.type != 'MESH':
        continue
    bb = [obj.matrix_world @ mathutils.Vector(corner) for corner in obj.bound_box]
    dims = obj.dimensions
    print(f"{obj.name}: bounds {dims.x:.1f} x {dims.y:.1f} x {dims.z:.1f}")
```

If a mesh's bounding box dimensions are significantly larger than its visible geometry, it has invisible collision space that needs fixing.

### Double-Sided Face Handling

Roblox renders meshes **single-sided** by default. Thin objects (leaves, paper, banners, cloth) will be invisible from one side unless handled:

- **Preferred:** Add a Solidify modifier with minimal thickness before export
- **Alternative:** Duplicate the face, flip its normal, and merge — creates a two-sided surface
- The agent must identify thin/planar objects and warn the user or apply Solidify automatically

### Scale and Units

Roblox 1 stud ~ 0.28 meters (roughly 1 foot). Choose one approach and stick with it:

- **Option A:** Model at real-world scale, apply factor `0.28` on FBX export
- **Option B:** Set Blender unit scale to `0.28` and model in stud-scale

All models in a project must use the same approach. Mismatched scales cause sizing issues in Studio.

### UV Unwrapping Requirements

Every textured mesh **must** have a UV map before export. The agent should:

1. **Check for UVs** after creating or importing any mesh:

```python
import bpy

obj = bpy.data.objects.get("ObjectName")
if obj is None:
    raise ValueError("Object 'ObjectName' not found")

if not obj.data.uv_layers:
    print(f"WARNING: '{obj.name}' has no UV map — textures will not display in Roblox")
```

2. **Auto-unwrap** if no UV exists — use Smart UV Project as a safe default:

```python
import bpy

obj = bpy.data.objects.get("ObjectName")
bpy.context.view_layer.objects.active = obj
bpy.ops.object.mode_set(mode='EDIT')
bpy.ops.mesh.select_all(action='SELECT')
bpy.ops.uv.smart_project(angle_limit=1.15, island_margin=0.02)
bpy.ops.object.mode_set(mode='OBJECT')
print(f"Smart UV Project applied to '{obj.name}'")
```

3. **Warn about UV issues** — stretching, overlapping islands, and missing UVs all cause broken textures in Roblox

### Texture Baking (Critical for Roblox)

Blender procedural materials (Noise, Voronoi, Wave, Musgrave, etc.) **do not transfer to Roblox**. Any procedural material must be baked to image textures before export.

**When to bake:**
- Any material using procedural texture nodes
- Any material with complex node mixing
- Materials from Polyhaven that use multiple texture maps through nodes

**Bake targets** (match Roblox SurfaceAppearance channels):

| Bake Type       | Roblox Channel   | Resolution  |
|-----------------|------------------|-------------|
| Diffuse/Color   | `ColorMap`       | 1024x1024   |
| Normal          | `NormalMap`      | 1024x1024   |
| Roughness       | `RoughnessMap`   | 1024x1024   |
| Metallic        | `MetalnessMap`   | 1024x1024   |

**Baking workflow:**

```python
import bpy

obj = bpy.data.objects.get("ObjectName")
if obj is None:
    raise ValueError("Object 'ObjectName' not found")

# Ensure UV map exists
if not obj.data.uv_layers:
    raise ValueError(f"'{obj.name}' needs a UV map before baking")

# Create bake target image
bake_image = bpy.data.images.new("BakedColor", width=1024, height=1024)

# Add Image Texture node to material and set as active for baking
mat = obj.data.materials[0]
nodes = mat.node_tree.nodes
bake_node = nodes.new(type='ShaderNodeTexImage')
bake_node.image = bake_image
nodes.active = bake_node

# Set render engine to Cycles (required for baking)
bpy.context.scene.render.engine = 'CYCLES'

# Select object and bake
bpy.context.view_layer.objects.active = obj
obj.select_set(True)
bpy.ops.object.bake(type='DIFFUSE')

print(f"Baked diffuse texture for '{obj.name}'")
```

Repeat for each PBR channel. Save baked images to disk as PNG at 1024x1024.

### Materials for Roblox

Roblox `SurfaceAppearance` supports four PBR maps:

| Blender Channel      | Roblox SurfaceAppearance |
|----------------------|--------------------------|
| Base Color           | `ColorMap`               |
| Normal Map           | `NormalMap`              |
| Metallic             | `MetalnessMap`           |
| Roughness            | `RoughnessMap`           |

Rules:
- **One material per object** — each Blender object = one MeshPart with one SurfaceAppearance
- **Texture atlas for maps** — consolidate textures into a single 1024x1024 atlas to reduce draw calls
- **No textures larger than 1024x1024** — Roblox downscales anything bigger

### Vertex Colors as a Texture Alternative

For stylized or low-poly models, **vertex colors** are cheaper than image textures:

- No UV map required
- No texture memory overhead
- Works well for flat-shaded, colorful props

```python
import bpy

obj = bpy.data.objects.get("StylizedProp")
if obj is None:
    raise ValueError("Object 'StylizedProp' not found")

# Create vertex color layer
if not obj.data.vertex_colors:
    obj.data.vertex_colors.new(name="Col")

print(f"Vertex color layer created on '{obj.name}'")
```

Use vertex colors when the model is simple/stylized and doesn't need detailed textures. Use image textures for realistic or detailed surfaces.

### Emissive and Special Materials

**Emissive/Glow:** Blender Emission shader maps to Roblox **Neon** material. The agent should:
- Note when a material uses emission and flag it for Neon material assignment in Studio
- Bake emission to a separate texture if needed for the ColorMap

**Transparency:** Roblox handles transparency through MeshPart `Transparency` property (0–1), not through material nodes. The agent should:
- Warn that Blender alpha transparency does not export to Roblox
- Flag transparent objects for manual `Transparency` setup in Studio

**Glass/Reflections:** Not supported via mesh import. Use Roblox's built-in Glass material in Studio instead.

### Object Structure

- **One object = one MeshPart** — every piece that will be a separate MeshPart must be a separate Blender object
- **Set origins intentionally** — Blender origin becomes the Roblox pivot point. Base for props, center for symmetric objects.
- **Roblox-safe names** — PascalCase, no special characters: `WallSegment_2m`, `RoofTile_Corner`, `DoorFrame_Single`

### Collision Meshes

- Create simplified collision meshes for complex models — `_Collision` suffix (e.g., `TreeTrunk_Collision`)
- Roblox collision works best with **convex shapes** — decompose concave models into multiple convex parts
- Collision meshes do not need textures or materials

### Roblox Avatar and Accessory Rules

When building UGC accessories or avatar items:

- **Follow R15 rig structure** — use Roblox-provided templates for attachment points
- **Attachment naming** — must match Roblox attachment names exactly: `HatAttachment`, `HairAttachment`, `FaceAttachment`, etc.
- **Cage meshes** — layered clothing requires inner and outer cage meshes for proper deformation
- **Triangle budget for accessories** — keep well under 10,000; hats and simple accessories should target 2,000–4,000
- **Size limits** — accessories must fit within Roblox's bounding box limits for their category

### Modular Map Design

When building maps and environments:

- **Reusable tiles** — wall segments, floor tiles, roof pieces that snap together
- **Stud-aligned grid** — dimensions in multiples of 0.28m or whole studs
- **Hollow interiors** — thin shells, not solid walls — saves triangles
- **Zone collections** — group by area: `Zone_Spawn`, `Zone_Arena`, `Zone_Lobby`

### Batch Export for Maps

When a map has many separate pieces, export each object or collection as its own `.fbx`:

```python
import bpy
import os

export_dir = "/path/to/exports"
os.makedirs(export_dir, exist_ok=True)

# Deselect all
bpy.ops.object.select_all(action='DESELECT')

for obj in bpy.data.collections["Zone_Spawn"].objects:
    if obj.type != 'MESH':
        continue

    bpy.ops.object.select_all(action='DESELECT')
    obj.select_set(True)
    bpy.context.view_layer.objects.active = obj

    filepath = os.path.join(export_dir, f"{obj.name}.fbx")
    bpy.ops.export_scene.fbx(
        filepath=filepath,
        use_selection=True,
        apply_scale_options='FBX_SCALE_ALL',
        axis_forward='-Z',
        axis_up='Y',
        use_mesh_modifiers=True,
        mesh_smooth_type='FACE',
        use_mesh_edges=False,
        bake_anim=False
    )
    print(f"Exported: {filepath}")
```

Never export an entire map as one monolithic `.fbx`. Keep pieces modular for Roblox Studio.

### Pre-Export Validation Checklist

The agent must run this checklist before every export:

| Check                        | Requirement                             |
|------------------------------|-----------------------------------------|
| Triangle count per mesh      | < 10,000                                |
| Total mesh count             | < 200                                   |
| All transforms applied       | Location, Rotation, Scale               |
| All meshes triangulated      | No quads or n-gons                      |
| Normals facing outward       | No flipped normals                      |
| No loose geometry            | No floating verts/edges                 |
| No interior faces            | Hidden geometry removed                 |
| No invisible collision space | Bounding box tightly wraps visible geo  |
| No distant-object merges     | Only merge physically touching geometry |
| Origins at geometry center   | `ORIGIN_GEOMETRY` with `BOUNDS` center  |
| One material per object      | Clean SurfaceAppearance mapping         |
| UV maps present              | On every textured mesh                  |
| Procedurals baked            | No unbaked procedural materials         |
| Texture resolution           | <= 1024x1024                            |
| Descriptive object names     | PascalCase, no special characters       |
| Origins set correctly        | Intentional pivot placement             |
| Scale consistent             | Same unit convention across all models  |
| Thin objects solidified      | No single-sided faces on visible meshes |
| Collision meshes created     | For complex or concave models           |

### FBX Export Settings

```python
import bpy

bpy.ops.export_scene.fbx(
    filepath="/path/to/export.fbx",
    use_selection=True,
    apply_scale_options='FBX_SCALE_ALL',
    axis_forward='-Z',
    axis_up='Y',
    use_mesh_modifiers=True,
    mesh_smooth_type='FACE',
    use_mesh_edges=False,
    bake_anim=False
)
```

Always export with `use_mesh_modifiers=True` so Triangulate and Decimate modifiers are baked in.

### Roblox Studio Import Verification

After importing into Roblox Studio, verify:

- [ ] Model scale matches expectations (compare to a default Roblox Part)
- [ ] SurfaceAppearance textures loaded on all MeshParts
- [ ] No missing or flipped faces when rotating the camera around the model
- [ ] Collision behaves correctly (walk into it, stand on it)
- [ ] Transparency and special materials flagged during export are configured in Studio
- [ ] Performance is acceptable (check MicroProfiler if the map is large)

---

## File Versioning

### Blend File Saves

- Save `.blend` files with version suffixes: `MapArena_v01.blend`, `MapArena_v02.blend`
- Increment the version before major changes so you can roll back
- Keep a `_latest` symlink or copy if preferred: `MapArena_latest.blend`

### Exported FBX Naming

- Match the Blender object name: `WallSegment_2m.fbx`, `RoofTile_Corner.fbx`
- For versioned exports: `WallSegment_2m_v02.fbx`
- Store exports in a dedicated folder: `exports/` or `exports/v02/`

---

## Lighting Best Practices

### Three-Point Lighting Setup

```python
import bpy
import mathutils

def create_area_light(name, location, energy, size, rotation):
    light_data = bpy.data.lights.new(name=name, type='AREA')
    light_data.energy = energy
    light_data.size = size
    light_obj = bpy.data.objects.new(name, light_data)
    light_obj.location = location
    light_obj.rotation_euler = rotation

    collection = bpy.data.collections.get("Lighting")
    if collection is None:
        collection = bpy.data.collections.new("Lighting")
        bpy.context.scene.collection.children.link(collection)
    collection.objects.link(light_obj)
    return light_obj
```

### HDRI Environment Lighting

Use Polyhaven HDRIs for realistic environment lighting:

```
1. search_polyhaven_assets (asset_type="hdris", categories="outdoor")
2. download_polyhaven_asset (resolution="2k" for preview, "4k" for final)
```

The HDRI is automatically set as the world environment after download.

---

## Camera and Rendering

### Camera Setup

```python
import bpy

cam_data = bpy.data.cameras.new("MainCamera")
cam_obj = bpy.data.objects.new("MainCamera", cam_data)

collection = bpy.data.collections.get("Cameras")
if collection is None:
    collection = bpy.data.collections.new("Cameras")
    bpy.context.scene.collection.children.link(collection)
collection.objects.link(cam_obj)

bpy.context.scene.camera = cam_obj
```

### Render Settings Checklist

Before rendering, always verify:

- [ ] Active camera is set (`bpy.context.scene.camera`)
- [ ] Resolution is appropriate (`scene.render.resolution_x`, `resolution_y`)
- [ ] Render engine matches the task (`EEVEE`, `CYCLES`, `BLENDER_WORKBENCH`)
- [ ] Sample count is set (higher for final, lower for preview)
- [ ] Output path is configured if saving to file

---

## Geometry Nodes

### Creating a Geometry Nodes Modifier

```python
import bpy

obj = bpy.data.objects.get("TargetObject")
if obj is None:
    raise ValueError("Object 'TargetObject' not found")

modifier = obj.modifiers.new(name="GeoNodesScatter", type='NODES')
group = bpy.data.node_groups.new("NG_ScatterPoints", 'GeometryNodeTree')
modifier.node_group = group
```

### Node Tree Best Practices

- Name every node group with the `NG_` prefix
- Use Group Input/Output nodes for reusable setups
- Add Frame nodes with labels to organize complex trees
- Keep node trees modular — nest groups instead of building monoliths

---

## Animation Guidelines

### Keyframe Insertion

```python
import bpy

obj = bpy.data.objects.get("AnimatedCube")
if obj is None:
    raise ValueError("Object 'AnimatedCube' not found")

# Frame 1: start position
obj.location = (0, 0, 0)
obj.keyframe_insert(data_path="location", frame=1)

# Frame 60: end position
obj.location = (5, 0, 2)
obj.keyframe_insert(data_path="location", frame=60)
```

### Animation Checklist

- Set frame range before animating (`scene.frame_start`, `scene.frame_end`)
- Use meaningful frame numbers (24fps or 30fps standard)
- Apply easing curves via the Graph Editor for polished motion
- Name actions descriptively: `Action_CubeSlide`, `Action_CameraOrbit`

---

## Troubleshooting

### Common Issues

| Problem                         | Solution                                              |
|---------------------------------|-------------------------------------------------------|
| Object not visible              | Check collection visibility and object hide state     |
| Material appears black          | Ensure nodes are connected to Material Output          |
| Script fails with context error | Set `view_layer.objects.active` before `bpy.ops` calls|
| Modifier not applying           | Check object mode — most modifiers need Object Mode   |
| Texture not appearing           | Verify UV map exists and material uses correct image   |
| Invisible collision in Roblox   | Remove interior faces, delete loose geo, reset origin, never merge distant objects — see "Invisible Collision Space" section |

### Debugging Workflow

```
1. get_scene_info            → Check overall state
2. get_object_info("Name")   → Inspect the problem object
3. get_viewport_screenshot   → See what Blender actually shows
4. execute_blender_code      → Print diagnostic info
5. Fix and verify
```

---

## Workflow Summary

For any Blender MCP task, follow this general pattern:

1. **Understand** — Inspect the scene and gather context
2. **Plan** — Break the task into discrete, small steps
3. **Execute** — Run each step as a separate code block
4. **Verify** — Screenshot and inspect after each meaningful change
5. **Refine** — Adjust based on visual confirmation

Never assume the scene state. Always verify. Always name things clearly.
