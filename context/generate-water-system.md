# Prompt: Water System Generation (Rivers, Lakes, Ponds)

Use this prompt with a Unity-capable coding agent (Unity MCP) to generate a complete water system on `ProceduralMeshGround`. This layer adds visible water surfaces to the river valley carved in Layer 1, carves and fills lakes in natural low points, and scatters small ponds in flat areas.

## Prerequisites

- `ProceduralMeshGround` must exist with a valid mesh, UVs, and MeshCollider.
- Terrain must have height variation (mountains, hills, plains) — rivers need elevation change.
- `RiverPath` GameObject must exist (created by terrain generation) with child transforms defining the river spline.
- Terrain splat mapping (Layer 2) should be complete — water system will trigger a full rebake.

## Water body types

| Type | Size | Shape | Where it belongs | Basin |
|------|------|-------|-----------------|-------|
| **River** | Half-width 8-12, length spans map | Long strip following RiverPath spline | Valley carved by terrain generator, flows off map edges | Already carved by Layer 1. This layer adds the water surface only. |
| **Lake** | Radius 25-60 | Irregular (noise-deformed circle, 8-12 control points) | Natural low points where terrain height < 20% of height range, or where rivers widen | Carve basin: depth 4-8, smooth cosine falloff over bankWidth 15-25 |
| **Pond** | Radius 8-18 | Smooth circle/oval | Plains and gentle hills, slope < 0.15, away from rivers and lakes | Carve basin: depth 2-4, smooth cosine falloff over bankWidth 8-12 |

---

## Copy/Paste Prompt

```text
Generate a complete water system on the existing ProceduralMeshGround in the active Unity scene.
This adds river water surfaces, carves and fills lakes, and places small ponds.

IMPORTANT RULES:
- transform.localScale is ALWAYS (1, 1, 1) for all water objects.
- Read terrain mesh data (vertices, normals, bounds) before placing anything.
- All basin carving uses smooth cosine falloff — NO cliff edges.
- After ALL carving is complete, rebake ALL terrain textures (single rebake pass, not per-water-body).

Inputs (honor these if specified, otherwise use defaults):
- seed = 42
- Map size is read from ProceduralMeshGround mesh bounds.

1. READ TERRAIN AND RIVER DATA
   - Find "ProceduralMeshGround" — get MeshFilter mesh vertices, normals, bounds.
   - Record height range (minH, maxH), terrain bounds (minX, maxX, minZ, maxZ).
   - Find "RiverPath" — reconstruct the river polyline from child transforms.
     Build a dense list of (position, forward direction) samples along the spline.
   - Compute the river's elevation profile (height at each sample point).

2. GENERATE RIVER WATER SURFACE
   The river channel was already carved by terrain generation. This step adds the visible water.

   - Walk the RiverPath polyline. At each sample point:
     - Sample terrain height at river center via raycast or vertex lookup.
     - Water surface Y = riverBedHeight + waterDepth (waterDepth = 1.5-2.5, varies slightly).
     - River width at this point = riverHalfWidth * 2 (use 8-12 half-width, matching terrain carve).

   - Build a continuous quad-strip mesh along the entire path:
     - At each sample, emit two vertices: center + perpendicular * halfWidth on each side.
     - Connect adjacent samples into quads.
     - UVs: U = cross-stream (0 to 1), V = along-stream (accumulate distance / totalLength).

   - The mesh must extend BEYOND terrain bounds at both ends (river flows off map).

   - Create GameObject "RiverWater" under parent "Water".
     MeshFilter + MeshRenderer. No MeshCollider needed (visual only).
   - Assign material: Assets/Materials/Water.mat
     - If material doesn't exist, create URP Lit material:
       Surface Type = Transparent, Base Color = (0.15, 0.30, 0.42, 0.75),
       Smoothness = 0.92, Metallic = 0.0.

3. IDENTIFY LAKE LOCATIONS
   Lakes form in natural low areas. Use terrain data to find candidates:

   - Scan terrain in a coarse grid (every 40-60 units).
   - At each sample, compute average height in a radius of 30-50 units.
   - Candidate = local minimum where avgHeight < (minH + 0.20 * (maxH - minH)).
   - Additional candidates: points along the river where the valley widens
     (terrain slopes away gently on both sides for > 40 units).

   - Merge candidates closer than 50 units (keep the lowest).
   - Reject candidates too close to terrain edge (< 40 units from bounds).
   - Reject candidates on steep slopes (average slope > 0.20 in the radius).

   - Target count: 1 lake per 250x250 area of map.
     A 500x500 map gets 2-4 lakes. A 1000x1000 map gets 6-12.

   - For each lake, determine radius:
     - Base radius: 25-60 (larger maps can have larger lakes).
     - Reduce radius if it would overlap another lake or extend past terrain bounds.

4. CARVE LAKE BASINS
   For each lake candidate:

   - Generate an irregular shoreline shape:
     - Start with a circle of the chosen radius.
     - Apply 8-12 control points with noise-based radius variation (0.7x to 1.3x base radius).
     - Smooth with Catmull-Rom interpolation for natural shoreline.

   - Carve basin into terrain mesh vertices:
     - For each vertex within lake influence range (radius + bankWidth):
       - Compute distance from vertex to nearest point on shoreline polygon.
       - Inside shoreline: lower vertex by carveDepth (4-8 units).
         Use smooth bowl profile: depth = carveDepth * cos(distFromCenter / radius * PI/2).
       - Bank zone (shoreline to shoreline + bankWidth):
         Blend from carved depth to original height using cosine falloff.
         bankWidth = 15-25 units for natural shores.
       - Beyond bank zone: no modification.

   - Store each lake's center, shoreline polygon, and water surface height for later.

   - If a lake intersects the river:
     - The lake water level = max(river water level at intersection, lake basin floor + waterDepth).
     - The shoreline opens where the river enters/exits (no bank wall across river).

5. GENERATE LAKE WATER SURFACES
   For each lake:

   - Build a mesh from the irregular shoreline polygon:
     - Triangulate the shoreline polygon (fan from center or ear-clipping).
     - Water surface Y = basinFloor + waterDepth (waterDepth = 3-6 for lakes).
     - UVs: planar XZ mapping, scaled so 1 UV unit = 20 world units.

   - Create GameObject "Lake_N" under parent "Water".
     MeshFilter + MeshRenderer.
   - Assign material: Assets/Materials/Water.mat (same as river).

6. IDENTIFY AND PLACE PONDS
   Ponds are small water bodies in flat terrain, away from larger water features.

   - Candidate placement:
     - Random XZ positions within terrain bounds (edge padding = 30 units).
     - Must be in flat areas: local slope < 0.15.
     - Must be at low-to-mid elevation: height < 50% of height range.
     - Minimum distance from any river point: 40 units.
     - Minimum distance from any lake shoreline: 30 units.
     - Minimum distance between ponds: 35 units.

   - Target count: 1 pond per 150x150 area of map.
     A 500x500 map gets 6-10 ponds. A 1000x1000 map gets 20-40.

   - For each pond:
     - Radius: 8-18 units (randomized per pond).
     - Shape: smooth circle or slight oval (aspect ratio 1.0-1.4, random rotation).
     - Carve basin: depth 2-4, bankWidth 8-12, cosine falloff.
     - Water surface Y = basinFloor + waterDepth (waterDepth = 1.0-2.0).

   - Build circular/oval mesh for water surface.
     32-48 edge vertices for smooth circle. UVs: planar XZ, 1 UV unit = 10 world units.

   - Create GameObjects "Pond_N" under parent "Water".
     Assign material: Assets/Materials/Water.mat.

7. POST-CARVE TERRAIN REFRESH (MANDATORY — single pass after ALL carving)
   Lake and pond carving modified terrain vertices. ALL terrain-derived data must be rebaked.

   - Update mesh:
     mesh.vertices = modified vertices
     mesh.RecalculateNormals()
     mesh.RecalculateTangents()
     mesh.RecalculateBounds()
     Update MeshCollider (assign mesh again or call MeshCollider.sharedMesh = mesh).

   - Rebake textures (use the SAME algorithms as Layer 1 and Layer 2):
     a) GroundHeightMap.png (1024x1024, linear)
     b) GroundNormalMap.png (2048x2048, normal map)
     c) GroundHeightTint.png (1024x1024, sRGB) — recompute height+slope coloring.
        Water body interiors should tint as dark earth/mud: (0.18, 0.15, 0.10).
        Bank zones blend to surrounding terrain color.
     d) GroundGrassMask.png (1024x1024, linear) — exclude:
        - Inside any water body shoreline
        - Within shorelineMargin (6 units) of any shoreline
        - Existing slope/height rules still apply
     e) TerrainSplatMap.png (1024x1024, linear RGBA) — recompute splat weights.
        Water body interiors and banks: increase dirt channel (B).
     f) Composite albedo, normal, mask — rebake from updated splat map.

   - Reassign all textures to Ground.mat (same slots as Layer 1/2).
   - Regenerate ProceduralGrass with updated mask (no grass in water or shoreline zones).

8. EXCLUSION ZONE DATA (for downstream layers)
   Store exclusion data so future placement layers (trees, rocks, etc.) can query it:

   - Create GameObject "WaterExclusionZones" under "Water" parent.
   - For each water body, store as child transform or component:
     - Type (river/lake/pond)
     - Center position
     - Radius or bounding polygon
     - shorelineMargin = 6 for ponds, 8 for lakes, 5 for river banks

   Downstream placement rule: NO grounded objects within (waterBody + shorelineMargin).
   Bushes may approach 50% closer than trees. Flowers may approach 70% closer.
   Rocks may be placed ON shoreline banks (partially submerged look).

9. ENVIRONMENT
   - Fog MUST remain disabled (RenderSettings.fog = false).

10. SAVE AND REPORT
    Return summary:
    - River: length, average width, water depth, vertex count
    - Lakes: count, positions, radii, shapes
    - Ponds: count, positions, radii
    - Total water surface area (approximate)
    - Terrain textures rebaked (list paths)
    - Grass regenerated (count, exclusions applied)
    - Material assignments confirmed
    - Exclusion zone data stored
```

---

## Expected Result

- **River** is a continuous, visible water surface flowing along the carved valley and extending off map edges. Water sits slightly above the river bed, not buried in terrain.
- **Lakes** are large, irregularly shaped water bodies in natural low points. Shorelines are organic, not perfect circles. Basins are smooth bowls, not cliff-walled pits.
- **Ponds** are small, smooth circles/ovals scattered in flat terrain. Clearly separate from rivers and lakes.
- All water shares the same material with a blue-green transparent look.
- Terrain around carved basins has smooth banks with cosine falloff — no cliff edges.
- Terrain textures are fully rebaked: tint, normal, heightmap, grass mask, splat map, composites.
- No grass inside water bodies or within shoreline margins.
- Exclusion zone data is stored for downstream placement layers.

---

## Scoring Rubric (100 points)

### River Quality (25 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Water surface follows path** | 8 | Continuous quad-strip mesh along RiverPath. No gaps. Extends off map edges. |
| **Correct elevation** | 7 | Water sits above river bed, not buried. Flows downhill (surface Y descends along path). |
| **Width and shape** | 5 | Consistent width matching carved channel. Smooth, no jagged edges. |
| **Visual quality** | 5 | Transparent blue-green, readable as water. Distinct from terrain. |

### Lake Quality (25 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Natural placement** | 7 | Lakes in genuine low points. Not on mountain slopes or ridges. |
| **Irregular shoreline** | 7 | Noise-deformed shape, not perfect circles. 8-12 control points, smooth interpolation. |
| **Basin carving** | 6 | Smooth bowl profile. Cosine falloff banks. No cliff edges. |
| **River connection** | 5 | If lake intersects river, shoreline opens naturally. Water levels match. |

### Pond Quality (15 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Placement rules** | 5 | Flat terrain, low elevation, away from rivers/lakes. Minimum spacing enforced. |
| **Basin and surface** | 5 | Smooth carve, visible water, not buried. |
| **Count scaling** | 5 | Pond count scales with map size. Not too few, not too many. |

### Terrain Refresh (25 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Mesh update** | 5 | Normals, tangents, bounds recalculated. MeshCollider updated. |
| **Texture rebake** | 8 | All 6+ textures rebaked from post-carve geometry. Correct algorithms reused. |
| **Grass update** | 5 | Grass regenerated with water exclusion. No grass in water or shoreline zones. |
| **Splat map update** | 4 | Dirt channel increased in water/bank areas. Composite textures rebaked. |
| **Single pass** | 3 | All carving done FIRST, then ONE rebake pass. Not rebaking per water body. |

### Technical Quality (10 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Scale rule** | 2 | All water objects scale = (1,1,1). Size from mesh geometry. |
| **Hierarchy** | 3 | Clean: Water parent → RiverWater, Lake_N, Pond_N, WaterExclusionZones. |
| **Exclusion data** | 3 | Stored and queryable for downstream layers. Correct margins per type. |
| **Performance** | 2 | Generation under 20 seconds. No editor freeze. |

### Score Bands

| Range | Meaning |
|-------|---------|
| **0-25** | Broken or no visible water |
| **26-50** | Some water exists but wrong placement, buried surfaces, or no terrain refresh |
| **51-70** | Water system works but missing types, poor carving, or incomplete rebake |
| **71-85** | Good water system, minor issues with shorelines or refresh |
| **86-100** | Excellent: all three water types, natural placement, full terrain refresh |

---

## Quick Reference for Agents

| Goal | What to do |
|------|------------|
| Water buried in terrain | Increase waterDepth offset. Check basin was actually carved. |
| Lake shoreline too circular | Add more control points with wider noise amplitude (0.6x-1.4x). |
| Cliff edges on carved basin | Use cosine falloff, not linear. Increase bankWidth. |
| Pond on mountainside | Enforce slope < 0.15 and height < 50% checks. |
| Grass in water | Check grass mask rebake excludes water bodies + shorelineMargin. |
| Splat map wrong after carving | Recompute splat weights from post-carve heights/slopes. |
| River water not flowing downhill | Water surface Y must descend along RiverPath direction. |
| Lake overlaps river awkwardly | Open the shoreline polygon where river enters/exits. Match water levels. |
| Too few/many water bodies | Check count scaling rules vs map size. |

---

## Technical Notes

### Why one rebake pass?
Carving multiple basins then rebaking once is both faster and simpler than rebaking after each water body. The rebake algorithms (height tint, normal map, splat map, grass mask) are expensive — running them N times wastes time and risks inconsistency.

### Water material
All water types share one material. Visual differences come from mesh shape and size, not material variation. A single transparent URP Lit material with blue-green tint reads well for rivers, lakes, and ponds. A more advanced benchmark tier could test animated water shaders.

### River mesh construction
The quad-strip approach (two vertices per sample, connected into quads along the path) produces a clean, continuous mesh that follows curves naturally. This is the standard technique for ribbon/river rendering on arbitrary paths.

### Lake-river intersection
When a lake sits on a river, the lake effectively widens the river into a pool. The shoreline polygon should have a gap where the river enters and exits. The water levels must match — if they don't, there will be a visible step in the water surface.

### Exclusion zone margins
Different object types have different shoreline clearances. Trees need the most space (full margin). Bushes can be closer (natural shoreline vegetation). Rocks can be ON the bank (partially submerged boulders look natural). These margins are stored with the exclusion data so downstream layers can query per-type distances.
