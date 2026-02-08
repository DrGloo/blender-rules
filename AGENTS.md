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
