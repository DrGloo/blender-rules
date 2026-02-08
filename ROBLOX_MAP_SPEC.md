# Roblox Map Specification (Medium, Stylized Low-Poly)

This document implements the approved map plan for a medium Roblox map using a stylized low-poly, vertex-color workflow. It follows the constraints and export rules in `AGENTS.md`.

---

## 1) Zone Layout and Stud Dimensions

**Global footprint:** 360 x 280 studs (approx. 100.8m x 78.4m at 0.28m per stud).  
**Player target:** 20–40 players.  
**Grid alignment:** 4-stud increments for all major dimensions.

### Zones (3–5 total)

| Zone | Stud Footprint | Purpose | Notes |
|------|----------------|---------|-------|
| `Zone_Spawn` | 80 x 80 | Safe spawn + tutorial | Clear line of sight, low cover |
| `Zone_Lobby` | 80 x 60 | Social hub | Benches, signage props |
| `Zone_Arena` | 160 x 140 | Main combat/interaction area | Verticality via platforms |
| `Zone_SidePath` | 60 x 140 | Flanking route | Narrow corridors, limited sight |
| `Zone_Overlook` | 60 x 80 | Sniper/advantage area | Elevated, limited access |

### Primary Paths

- **Main route:** `Zone_Spawn` → `Zone_Lobby` → `Zone_Arena`
- **Secondary route:** `Zone_Spawn` → `Zone_SidePath` → `Zone_Arena`
- **Vertical route:** `Zone_Arena` → `Zone_Overlook` (stairs + ladder)

---

## 2) Modular Kit (Snap-Aligned)

All modular pieces are designed to snap on a 4-stud grid. Each object = one MeshPart.

### Structural Modules

| Module | Size (studs) | Name Example | Origin |
|--------|-------------|--------------|--------|
| Floor Tile | 8 x 8 x 1 | `FloorTile_8x8` | Bottom-center |
| Wall Segment | 8 x 1 x 8 | `WallSegment_8x8` | Bottom-center |
| Corner Wall | 8 x 8 x 8 | `WallCorner_8x8` | Bottom-center |
| Door Frame | 8 x 1 x 8 | `DoorFrame_8x8` | Bottom-center |
| Window Wall | 8 x 1 x 8 | `WallWindow_8x8` | Bottom-center |
| Stair Unit | 8 x 16 x 8 | `Stairs_8x16` | Bottom-center |
| Ramp | 8 x 16 x 4 | `Ramp_8x16` | Bottom-center |
| Platform | 8 x 8 x 2 | `Platform_8x8` | Bottom-center |

### Prop Modules (Low-Poly)

| Prop | Size (studs) | Name Example | Notes |
|------|-------------|--------------|-------|
| Crate | 4 x 4 x 4 | `Crate_4x4` | Cover element |
| Barrel | 4 x 4 x 6 | `Barrel_4x6` | Rounded low-poly |
| Lamp Post | 2 x 2 x 12 | `LampPost_2x12` | Light marker |
| Bench | 8 x 2 x 2 | `Bench_8x2` | Lobby decoration |
| Sign | 6 x 1 x 4 | `Sign_6x4` | Zone labeling |

### Snap Rules

- Every module dimension is a multiple of **4 studs**.
- Origins are set to **bottom-center** for easy floor snapping.
- Rotation increments are **90°** for alignment.

---

## 3) Vertex Color Palette (Stylized)

Vertex colors replace textures for primary surfaces. Use a consistent palette to unify the map.

| Palette Role | Hex | RGB |
|--------------|-----|-----|
| Primary Stone | `#6B7A8F` | (107, 122, 143) |
| Accent Stone | `#8899AA` | (136, 153, 170) |
| Metal Dark | `#3A3F4B` | (58, 63, 75) |
| Metal Light | `#5D6775` | (93, 103, 117) |
| Wood Dark | `#5B3E2B` | (91, 62, 43) |
| Wood Light | `#8B6A4E` | (139, 106, 78) |
| Trim Accent | `#C2A46E` | (194, 164, 110) |
| Danger Accent | `#C94C4C` | (201, 76, 76) |
| Foliage Green | `#4C7A4C` | (76, 122, 76) |

**Usage rules:**  
- Use **Primary Stone** for floors/walls, **Accent Stone** for edges.  
- Use **Metal** colors for railings, ladders, lamps.  
- Use **Trim Accent** sparingly to guide navigation.  
- **Danger Accent** only for hazard zones or critical markers.

---

## 4) Optimization and Export Checklist

Before export, each object must pass:

- < 10,000 triangles (target 5,000–8,000)
- One material (vertex colors only)
- Triangulated
- Transforms applied
- Normals outward
- No interior faces or loose geometry
- Roblox-safe names

### Batch Export Naming

- Per-object exports: `WallSegment_8x8.fbx`, `FloorTile_8x8.fbx`
- Versioned exports: `WallSegment_8x8_v02.fbx`

---

## 5) Roblox Studio Import Verification

After import:

- Check scale against a default Part (1 x 1 x 1 stud cube)
- Confirm vertex colors are visible
- Verify collisions and walkable surfaces
- Check performance in MicroProfiler for the full map

