# Prompt: Props Generation (Trees, Rocks)

Use this prompt with a Unity-capable coding agent (Unity MCP) to populate the terrain with procedural props — trees and rocks. This is a **placement layer** — it reads terrain and water data but does NOT modify them.

## Critical Rule: Previous Layers are READ-ONLY

**Do NOT modify terrain mesh, terrain textures, water surfaces, or any previous layer output.** All props are placed ON TOP of the existing terrain via raycasting. Each benchmark layer only ADDS — never modifies previous layers.

## Prerequisites

- `ProceduralMeshGround` must exist with a valid mesh and MeshCollider.
- `WaterExclusionZones` must exist (from water layer) to avoid placing trees in water.
- `RiverPath` must exist (from terrain layer) for river-distance queries.
- Prop generation algorithms:
  - Trees: [props/generate-tree.md](props/generate-tree.md)
  - Rocks: [props/generate-rock.md](props/generate-rock.md)

## Copy/Paste Prompt

```text
Generate all props (trees, rocks) on the existing ProceduralMeshGround.
Each prop is a unique procedural mesh. Do NOT modify the terrain, water, or any previous layer output.

IMPORTANT RULES:
- transform.localScale is ALWAYS (1, 1, 1) for all objects.
- Every prop uses a UNIQUE seed (baseSeed + propIndex) so no two look alike.
- Raycast DOWN from above to find ground position for each prop.
- Respect WaterExclusionZones — no props in water or within exclusion margins.
- No props on steep slopes (slope > 0.5) or mountain peaks (height > 85% of max).
- Props must not penetrate each other — enforce minimum spacing.

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

5. ROCKS (see props/generate-rock.md for mesh details)
   - Place in rocky terrain zones: slope > 0.25 OR height > 60% of max.
   - Also scatter some in plains and near river banks.
   - Target: 200-400 rocks for 500x500 map.
   - Cell size: 8m with jitter.
   - Vary size: 0.3-2m radius.
   - Water exclusion margin: 50% of tree margin (rocks can be near shoreline).

6. HIERARCHY
   Props (root)
   ├── Trees
   │   ├── Tree_0 → Trunk + Leaves
   │   └── Tree_N → Trunk + Leaves
   └── Rocks
       ├── Rock_0
       └── Rock_N

7. MATERIALS (shared across all props of same type)
   Create ONCE, assign to all:
   - Assets/Materials/TreeBark.mat (Baked Lit, brown)
   - Assets/Materials/TreeLeaves.mat (Baked Lit, green, double-sided)
   - Assets/Materials/RockGray.mat (Baked Lit, gray)
   See props/*.md for exact material setup.

8. PERFORMANCE
   - Total prop count: 600-1000 for 500x500 map.
   - Each prop should be < 10K verts.
   - Generation should complete within 60 seconds.

9. REPORT
   Return summary:
   - Per type: count placed, placement failures
   - Total vertex count
   - Seed range used
   - Confirm: terrain was NOT modified
   - Confirm: water was NOT modified
```

---

## Expected Result

- A forest of **visually unique** trees covering appropriate terrain areas — dense in valleys/plains, sparse at altitude, absent on peaks and steep slopes.
- **Rocks** scattered on slopes, high terrain, and near river banks. Varied sizes and shapes.
- No props in water or within their respective exclusion margins.
- Natural-looking distribution with clustering and clearings (not a uniform grid).
- All props grounded via raycast — no floating, no buried.
- Clean hierarchy: Props → Trees/Rocks → individual items.

---

## Scoring Rubric (100 points)

### Placement Quality (40 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Height-zone respect** | 10 | All prop types in valid zones only. No trees on peaks, rocks in appropriate zones. |
| **Water exclusion** | 10 | No props in water. Trees use full exclusion margin, rocks on shoreline OK. |
| **Slope filtering** | 5 | No trees on cliffs. Rocks on slopes. |
| **Natural distribution** | 8 | Clustering with clearings for all types. Not a uniform grid. |
| **Spacing** | 4 | No overlapping props. Minimum distance enforced per type. |
| **Grounding** | 3 | All props sit on terrain surface. No floating. |

### Prop Quality (30 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Tree uniqueness** | 8 | Each tree visibly different. Seed system works. |
| **Tree structure** | 8 | Recursive branching, natural proportions, proper taper, cross-billboard foliage. |
| **Rock variety** | 7 | Size variation (0.3-2m). Singles, lines, and mounds present. Distorted icospheres. |
| **Materials** | 7 | Baked Lit for all. No glow. Correct colors per type. Shared materials. |

### Technical Quality (20 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Layer isolation** | 8 | Terrain and water NOT modified. Placement only. |
| **Scale rule** | 3 | All localScale = (1,1,1) for every prop. |
| **Hierarchy** | 4 | Clean: Props → Trees/Rocks → individual items. |
| **Performance** | 5 | Generates in < 60s. Total 600-1000 props. Each < 10K verts. |

### Visual Quality (10 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Scene reads as populated landscape** | 5 | Forest and rocky outcrops visible in appropriate areas. |
| **Density feels natural** | 3 | Not too sparse, not too uniform. Each type has appropriate coverage. |
| **Shadow quality** | 2 | Props cast shadows on terrain. Adds depth to scene. |

### Score Bands

| Range | Meaning |
|-------|---------|
| **0-25** | No props or props in wrong locations (water, peaks) |
| **26-50** | Some props placed but uniform grid, no exclusion, or all identical |
| **51-70** | Good placement, some variety, minor issues |
| **71-85** | Natural landscape with both prop types, proper exclusions |
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
| Rocks in wrong zones | Rocks go on slopes > 0.25, high terrain, and near river banks. |
| Props overlapping | Each type has minimum spacing. Check distance to all nearby props. |
| Wrong exclusion margins | Trees=full margin, Rocks=on shoreline OK. |
| All props same material | Create shared materials ONCE, assign to all instances of same type. |
