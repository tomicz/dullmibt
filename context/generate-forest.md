# Prompt: Forest Generation (Tree Placement Layer)

Use this prompt with a Unity-capable coding agent (Unity MCP) to populate the terrain with procedural trees. This is a **placement layer** — it reads terrain and water data but does NOT modify them.

## Critical Rule: Previous Layers are READ-ONLY

**Do NOT modify terrain mesh, terrain textures, water surfaces, or any previous layer output.** Trees are placed ON TOP of the existing terrain via raycasting. Each benchmark layer only ADDS — never modifies previous layers.

## Prerequisites

- `ProceduralMeshGround` must exist with a valid mesh and MeshCollider.
- `WaterExclusionZones` must exist (from water layer) to avoid placing trees in water.
- `RiverPath` must exist (from terrain layer) for river-distance queries.
- Tree generation algorithm from [context/props/generate-tree.md](props/generate-tree.md).

## Copy/Paste Prompt

```text
Generate a forest of procedural trees on the existing ProceduralMeshGround.
Each tree is a unique procedural mesh built from the tree generation algorithm.
Do NOT modify the terrain, water, or any previous layer output.

IMPORTANT RULES:
- transform.localScale is ALWAYS (1, 1, 1) for all objects.
- Every tree uses a UNIQUE seed (baseSeed + treeIndex) so no two trees look alike.
- Raycast DOWN from above to find ground position for each tree.
- Respect WaterExclusionZones — no trees in water or within exclusion margins.
- No trees on steep slopes (slope > 0.5) or mountain peaks (height > 85% of max).
- Trees must not penetrate each other — enforce minimum spacing.

1. READ TERRAIN AND EXCLUSION DATA
   - Find "ProceduralMeshGround" — get mesh bounds.
   - Find "WaterExclusionZones" — parse river halfwidth, margin, lake/pond data.
   - Find "RiverPath" — reconstruct river polyline for distance queries.
   - Compute terrain height range from mesh bounds.

2. PLACEMENT RULES
   Height zones (normalized height 0-1):
   - hN < 0.05: river valley floor — NO trees
   - hN 0.05-0.60: primary tree zone — FULL density
   - hN 0.60-0.75: alpine transition — REDUCED density (50%)
   - hN > 0.75: mountain peaks — NO trees

   Slope rules:
   - slope < 0.35: full density
   - slope 0.35-0.50: reduced density (50%)
   - slope > 0.50: no trees

   Water exclusion:
   - River: no trees within riverHalfWidth + exclusionMargin
   - Lakes: no trees within lakeRadius + exclusionMargin
   - Ponds: no trees within pondRadius + exclusionMargin

   Minimum spacing between trees: 4-8m (randomized per tree, trees are 5-8m tall)

3. DENSITY AND DISTRIBUTION
   Target density: 1 tree per 10x10m cell (adjustable).
   For a 500x500 map: expect 400-600 trees depending on terrain.

   Distribution method:
   - Grid-based with jitter: divide terrain into cells, place 0-1 tree per cell
   - Jitter: random offset within cell (+-40% of cell size)
   - Skip cells that fail height/slope/water checks
   - Density varies by height zone (see placement rules above)

   Clustering (natural forests aren't uniform):
   - Use a low-frequency noise (k=0.01) to create dense/sparse zones
   - Noise > 0.4: place tree. Noise < 0.4: skip (creates natural clearings)

4. PER-TREE GENERATION
   For each accepted placement position:
   - Compute unique seed: baseSeed(12345) + treeIndex
   - Raycast to get exact ground Y
   - Generate tree mesh using the recursive branching algorithm (see props/generate-tree.md)
   - Vary tree size with terrain context:
     - Valley/plains trees: full size (trunkHeight 5-8m)
     - Hill trees: slightly smaller (4-6m)
     - Alpine trees: smaller, sparser (3-5m)

5. HIERARCHY
   Trees (parent container)
   ├── Tree_0 (seed=12345)
   │   ├── Trunk
   │   └── Leaves
   ├── Tree_1 (seed=12346)
   │   ├── Trunk
   │   └── Leaves
   └── ... (Tree_N)

6. MATERIALS (shared across all trees)
   Create ONCE, assign to all:
   - Assets/Materials/TreeBark.mat (Baked Lit, brown)
   - Assets/Materials/TreeLeaves.mat (Baked Lit, green, double-sided)
   See props/generate-tree.md for exact material setup.

7. PERFORMANCE
   - Keep total tree count reasonable: 400-600 for 500x500 map.
   - Each tree should be < 10K verts.
   - Consider combining distant trees into shared meshes if performance is an issue.
   - Generation should complete within 30 seconds.

8. REPORT
   Return summary:
   - Total trees placed
   - Trees per height zone
   - Placement failures (water exclusion, slope, spacing)
   - Seed range used
   - Total vertex count
   - Confirm: terrain was NOT modified
   - Confirm: water was NOT modified
```

---

## Expected Result

- A forest of **visually unique** trees covering appropriate terrain areas.
- Dense in valleys and plains, sparse at altitude, absent on peaks and steep slopes.
- No trees in water or within exclusion margins.
- Natural-looking distribution with clustering and clearings (not a uniform grid).
- All trees grounded via raycast — no floating, no buried trunks.
- Clean hierarchy under a single "Trees" parent.

---

## Scoring Rubric (100 points)

### Placement Quality (40 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Height-zone respect** | 10 | Trees in valid zones only. No trees on peaks or in valley floor. |
| **Water exclusion** | 10 | No trees in river, lakes, or within exclusion margins. |
| **Slope filtering** | 5 | No trees on cliffs. Reduced density on moderate slopes. |
| **Natural distribution** | 8 | Clustering with clearings. Not a uniform grid. |
| **Spacing** | 4 | No overlapping trees. Minimum distance enforced. |
| **Grounding** | 3 | All trees sit on terrain surface. No floating. No buried. |

### Tree Quality (30 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Uniqueness** | 8 | Each tree is visibly different. Seed system works. |
| **Structural quality** | 8 | Recursive branching, natural proportions, proper taper. |
| **Foliage** | 7 | Cross-billboard clusters. Readable canopy. |
| **Size variation** | 4 | Trees vary in size. Alpine trees smaller than valley trees. |
| **Materials** | 3 | Baked Lit, no glow, bark and leaf materials distinct. |

### Technical Quality (20 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Layer isolation** | 8 | Terrain and water NOT modified. Placement only. |
| **Scale rule** | 3 | All localScale = (1,1,1). |
| **Hierarchy** | 4 | Clean: Trees → Tree_N → Trunk + Leaves. |
| **Performance** | 5 | Generates in < 30s. Reasonable vertex count. No editor freeze. |

### Visual Quality (10 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Forest reads as forest** | 5 | From overview, terrain looks forested in appropriate areas. |
| **Density feels natural** | 3 | Not too sparse, not too uniform. Natural-looking coverage. |
| **Shadow quality** | 2 | Trees cast shadows on terrain. Adds depth to scene. |

### Score Bands

| Range | Meaning |
|-------|---------|
| **0-25** | No trees or trees in wrong locations (water, peaks) |
| **26-50** | Trees placed but uniform grid, no exclusion, or all identical |
| **51-70** | Good placement, some trees unique, minor issues |
| **71-85** | Natural forest, unique trees, proper exclusions |
| **86-100** | Excellent: natural distribution, all unique, perfect exclusions, clean hierarchy |

---

## Quick Reference for Agents

| Problem | Solution |
|---------|----------|
| Trees in water | Check WaterExclusionZones. Parse river halfwidth + margin. |
| Trees on mountain peaks | Filter by normalized height. No trees above 75-85% of height range. |
| All trees identical | Each tree needs unique seed: baseSeed + treeIndex. |
| Trees floating | Raycast must hit ProceduralMeshGround. Check MeshCollider exists. |
| Uniform grid pattern | Add jitter to grid positions. Use noise-based density modulation. |
| Too many trees / slow | Reduce density. Target 400-600 for 500x500 map. |
| Trees on cliffs | Compute slope from nearby height samples. Skip if slope > 0.5. |
