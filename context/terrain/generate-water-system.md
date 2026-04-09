# Prompt: Water System Generation (Rivers, Lakes, Ponds)

Use this prompt with a Unity-capable coding agent (Unity MCP) to generate visible water surfaces on `ProceduralMeshGround`. Water reads the existing terrain geometry and places surfaces on top — it does **NOT** modify the terrain mesh.

## Critical Rule: Terrain is READ-ONLY

**The water layer must NEVER modify terrain vertices, normals, UVs, or any terrain textures.** The terrain was finalized in Layer 1 (and optionally Layer 2). Each benchmark layer builds on top of previous layers without modifying their output. Water surfaces sit on the terrain — they do not carve, flatten, or reshape it.

## Prerequisites

- `ProceduralMeshGround` must exist with a valid mesh, UVs, and MeshCollider.
- Terrain must have height variation (mountains, hills, plains) — rivers need elevation change.
- `RiverPath` GameObject must exist (created by terrain generation) with child transforms defining the river spline.
- The river valley was already carved into the terrain mesh during Layer 1. This layer only adds the visible water surface.

## Water body types

| Type | Required? | Size | Where it belongs |
|------|-----------|------|-----------------|
| **River** | **Mandatory** | Half-width 14-18, spans entire map | Follows RiverPath spline through the valley carved by terrain generation. Extends off map edges. |
| **Lake** | Optional (maps > 500x500) | Radius 25-60 | Natural low-elevation flat areas. Only if terrain has suitable basins. |
| **Pond** | Optional (maps > 500x500) | Radius 8-18 | Plains with slope < 0.15. Bonus for larger maps only. |

For a standard 500x500 map: only the river is required. Lakes and ponds are bonus points for larger maps that have enough flat low terrain to support them.

---

## Copy/Paste Prompt

```text
Generate water surfaces on the existing ProceduralMeshGround in the active Unity scene.
This adds visible water meshes that sit ON TOP of the terrain. Do NOT modify the terrain mesh.

IMPORTANT RULES:
- DO NOT modify ProceduralMeshGround vertices, normals, UVs, or collider. Terrain is READ-ONLY.
- DO NOT rebake any terrain textures (tint, normal, height, splat, grass mask). They are final.
- transform.localScale is ALWAYS (1, 1, 1) for all water objects.
- Read terrain mesh data (vertices, normals, bounds) to determine where water goes.
- Water surfaces sit slightly above the terrain surface — not buried, not floating high.

1. READ TERRAIN AND RIVER DATA
   - Find "ProceduralMeshGround" — get MeshFilter mesh bounds, transform scale.
   - Compute terrain world bounds (bounds * scale).
   - Find "RiverPath" — reconstruct the river polyline from child transforms.
   - Build a dense polyline using Catmull-Rom interpolation (200+ segments).

2. GENERATE RIVER WATER SURFACE (mandatory)
   The river channel was already carved into terrain during Layer 1.
   This step adds the visible water mesh that fills that channel.

   - Walk the RiverPath polyline. At each sample point:
     - Raycast DOWN from above to find terrain surface height at river center.
     - IMPORTANT: For points beyond terrain bounds, clamp raycast XZ to terrain
       bounds and extrapolate Y from the nearest valid sample.
     - Water surface Y = terrainHeight + 0.3 (just above the carved bed).
     - River half-width = 14-18 (match the channel carved in terrain generation).

   - Build a continuous quad-strip mesh along the entire path:
     - At each sample, compute forward direction and perpendicular (right).
     - Emit two vertices: center ± right * halfWidth.
     - Connect adjacent samples into quads (two triangles per quad).
     - UVs: U = cross-stream (0 to 1), V = along-stream (normalized distance).

   - Smooth the water Y values (3-5 passes of neighbor averaging) for gentle flow.
   - The mesh MUST extend beyond terrain bounds at both ends (river flows off map).

   - Create GameObject "RiverWater" under parent "Water".
     MeshFilter + MeshRenderer. No MeshCollider (visual only).

3. WATER MATERIAL
   - Path: Assets/Materials/Water.mat
   - If it doesn't exist, create a URP Lit material:
     Surface Type = Transparent
     Base Color = (0.12, 0.28, 0.40, 0.80) — blue-green, mostly opaque
     Smoothness = 0.95
     Metallic = 0.1
     Render queue = 3000
     Enable _SURFACE_TYPE_TRANSPARENT keyword
     ZWrite = 0, SrcBlend = SrcAlpha, DstBlend = OneMinusSrcAlpha

4. OPTIONAL: LAKES (only for maps > 500x500 with suitable terrain)
   If the map is large enough and has natural low basins:

   - Scan terrain for flat low areas (avgHeight < 20% of height range, slope < 0.15).
   - Target: 1 lake per 400x400 area. A 1000x1000 map might have 4-6 lakes.
   - Lake = flat circular/oval water surface mesh placed at the terrain's local minimum + 0.3.
   - The water surface covers the low area — it does NOT carve a basin.
     Some terrain will poke above the water at edges, creating a natural shoreline.
   - Use a fan mesh from center with 32-48 edge vertices.
   - Create as "Lake_N" under "Water" parent.

5. OPTIONAL: PONDS (bonus for large maps)
   Small water circles in flat plains:
   - Only if map > 500x500 and there are flat low areas away from river/lakes.
   - 1 pond per 200x200 area, radius 8-18.
   - Simple flat circle mesh at terrain height + 0.2.
   - Create as "Pond_N" under "Water" parent.

6. EXCLUSION ZONE DATA (for downstream layers)
   Store exclusion data so trees/rocks/bushes avoid water areas:

   - Create "WaterExclusionZones" under "Water" parent.
   - For each water body, add a child GameObject encoding:
     type, radius/halfwidth, margin.
   - Naming convention: "River_Exclusion_margin5_halfwidth16"
     or "Lake_0_r35_margin8" or "Pond_0_r12_margin6".

   Downstream placement rules:
   - Trees: full exclusion margin
   - Bushes: 50% of margin (can approach closer)
   - Flowers: 30% of margin
   - Rocks: can be ON shoreline banks

7. ENVIRONMENT
   - Fog MUST remain disabled (RenderSettings.fog = false).

8. REPORT
   Return summary:
   - River: length, width, vertex count, Y range
   - Lakes: count, positions, radii (if any)
   - Ponds: count, positions, radii (if any)
   - Material path
   - Exclusion zones stored
   - Confirm: terrain was NOT modified
```

---

## Expected Result

- **River** is a continuous, visible blue-green water surface flowing along the carved valley. It extends off both map edges. Water sits just above the terrain surface, not buried and not floating.
- **Lakes** (if present) are flat water circles/ovals in natural low areas. Some terrain pokes through at edges creating natural shorelines.
- **Ponds** (if present) are small water circles in flat plains.
- All water shares the same transparent material.
- Terrain mesh is UNCHANGED — no vertices modified, no textures rebaked.
- Exclusion zone data stored for downstream placement layers.

---

## Scoring Rubric (100 points)

### River Quality (50 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Water surface follows channel** | 15 | Continuous mesh along entire RiverPath. No gaps. Fills the carved valley. |
| **Correct elevation** | 10 | Water sits just above terrain bed (0.2-0.5 above). Not buried, not floating high. |
| **Extends off map** | 8 | Water mesh continues beyond terrain bounds at both ends. |
| **Width matches channel** | 7 | Water width matches the carved river channel width. |
| **Smooth flow** | 5 | No jagged Y jumps. Water surface is gently smoothed. |
| **Visual quality** | 5 | Transparent blue-green, readable as water. |

### Optional Water Bodies (20 points — bonus)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Lake placement** | 8 | In genuine low, flat areas. Not on slopes or peaks. |
| **Lake shape** | 4 | Reasonable shape. Natural shoreline where terrain intersects water. |
| **Pond placement** | 4 | Flat plains only, away from river/lakes. |
| **Count scaling** | 4 | Appropriate count for map size. Not too many on small maps. |

### Technical Quality (30 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Terrain untouched** | 10 | NO terrain vertices modified. NO textures rebaked. This is the #1 rule. |
| **Scale rule** | 3 | All water objects scale = (1,1,1). |
| **Material setup** | 5 | Transparent URP Lit. Blue-green tint. High smoothness. Correct blend mode. |
| **Hierarchy** | 4 | Clean: Water → RiverWater, Lake_N, Pond_N, WaterExclusionZones. |
| **Exclusion data** | 5 | Stored and queryable. Correct margins per water type. |
| **Performance** | 3 | Generation under 10 seconds. No editor freeze. |

### Score Bands

| Range | Meaning |
|-------|---------|
| **0-25** | No visible water or terrain was modified |
| **26-50** | River exists but buried/floating or terrain was carved |
| **51-70** | River works, minor elevation issues |
| **71-85** | Good river, optional water bodies present |
| **86-100** | Excellent: river fills channel perfectly, terrain untouched, clean hierarchy |

---

## Quick Reference for Agents

| Goal | What to do |
|------|------------|
| Water buried in terrain | Increase Y offset above terrain (try 0.4-0.5). |
| Water floating above channel | Decrease Y offset (try 0.2-0.3). |
| River missing at map edges | Extend polyline BEYOND terrain bounds. Clamp raycasts to bounds, extrapolate Y. |
| Raycasts miss terrain | Clamp XZ to terrain bounds before raycasting. Interpolate for out-of-bounds points. |
| Terrain was modified | WRONG. Undo all terrain changes. Water only ADDS meshes, never modifies terrain. |
| Want lakes on small map | Don't. Lakes are optional and only for maps > 500x500 with suitable terrain. |
| Too many ponds | Reduce count. Ponds are bonus, not required. |

---

## Technical Notes

### Why no terrain carving?
Previous benchmark iterations tried carving basins for lakes/ponds. This created problems:
1. Undoing carving if something goes wrong is nearly impossible.
2. It violates layer isolation — each layer should only ADD, not modify previous layers.
3. Real water fills existing low terrain. Carving is artificial.
4. The river valley is already carved during terrain generation (Layer 1) — that's the right place for it.

### Water surface elevation
The water Y should be terrain surface + 0.3 units. Too low and it's invisible (buried). Too high and it floats visibly above the channel walls. 0.3 is a sweet spot that looks like the channel is filled with water.

### River mesh construction
The quad-strip approach (two vertices per sample along path) produces a clean ribbon mesh. Key: compute forward direction, then perpendicular (right = cross(up, forward)), emit left/right vertices at center ± right * halfWidth.

### Handling beyond-bounds points
The river extends off the map. Raycasts at those positions miss the terrain. Solution: clamp the raycast XZ to within terrain bounds, get the height at the clamped position, use that as the water Y for the beyond-bounds segment.

### Layer isolation principle
Each benchmark layer operates on top of previous layers without modifying them:
- Layer 1 (Terrain): Creates mesh, textures, grass. FINAL.
- Layer 2 (Splat): Adds texture detail. Does not modify mesh. FINAL.
- Layer 3 (Water): Adds water surfaces. Does not modify mesh or textures. FINAL.
- Layer 4+ (Objects): Places objects via raycast. Does not modify anything above.
