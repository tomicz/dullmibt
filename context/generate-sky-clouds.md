# Prompt: Sky & Clouds (Primitive-Based Volumetric Clouds)

Use this prompt with a Unity-capable coding agent (Unity MCP) to generate a procedural cloud system using primitive meshes (spheres/ellipsoids). Clouds are built from clusters of scaled, overlapping primitives — not particles, not textures, not Unity's built-in cloud system.

## Critical Rule: Previous Layers are READ-ONLY

**Do NOT modify terrain, water, props, or lighting settings.** This layer only ADDS cloud GameObjects to the scene. Each benchmark layer only adds — never modifies previous layers' output.

## Prerequisites

- Layers 1-4 must be completed (terrain, splat, water, props).
- Layer 5 (lighting) should be done so clouds interact with the existing sun direction.
- `ProceduralMeshGround` must exist to determine terrain bounds for cloud placement.

---

## Copy/Paste Prompt

```text
Generate a procedural cloud system using primitive meshes in the active Unity scene.
Clouds are clusters of overlapping scaled spheres — NOT particles, NOT Unity terrain clouds.
Do NOT modify terrain, water, props, or lighting. This layer only ADDS cloud objects.

IMPORTANT RULES:
- transform.localScale is used ONLY on individual cloud puffs (spheres) to shape them.
  The parent cloud group uses scale (1,1,1).
- Clouds must be HIGH above the terrain — minimum 120m above the highest terrain point.
- Clouds cast NO shadows on the ground (shadow casting disabled).
- Each cloud is a cluster of 8-20 overlapping spheres, not a single sphere.

1. READ TERRAIN DATA
   - Find "ProceduralMeshGround" — get mesh bounds to determine world extents.
   - Find terrain height range from mesh bounds (need max height for cloud altitude).
   - Compute terrain center and footprint size.

2. CLOUD ALTITUDE
   - Cloud base height = terrain max height + 120 (minimum).
   - Cloud height band = 30m (clouds span a 30m vertical range).
   - All clouds exist between cloudBase and cloudBase + 30.

3. CLOUD DISTRIBUTION
   Target: 15-25 cloud groups for a 500x500 map.
   Scale with map size: ~1 cloud per 100x100 area.

   Distribution:
   - Spread clouds across the full map footprint (not clustered in one area).
   - Use grid with large jitter (cell size = mapSize / 5, jitter = +-40% of cell).
   - Clouds should NOT all be the same size — vary from small wisps to large formations.
   - Leave some cells empty (30% skip chance) to create open sky areas.

   Cloud types (vary across the sky):
   - Cumulus (60%): Large, puffy, rounded tops, flat bottoms.
     Width: 30-60m, Height: 15-25m, Depth: 25-45m.
   - Wisp (25%): Thin, elongated, stretched horizontally.
     Width: 40-80m, Height: 5-10m, Depth: 15-25m.
   - Small puff (15%): Tiny accent clouds.
     Width: 10-20m, Height: 8-12m, Depth: 10-18m.

4. PER-CLOUD GENERATION
   Each cloud group is a parent GameObject containing 8-20 child spheres ("puffs").

   For each cloud group:
   - Compute unique seed: baseSeed(99999) + cloudIndex.
   - Choose cloud type based on distribution percentages.
   - Create parent "Cloud_N" under "Clouds" container.
   - Parent position = grid cell center + jitter, Y = cloudBase + random(0, heightBand).
   - Parent scale = (1, 1, 1).

   Generate puffs (child spheres):
   - Each puff is a Unity sphere primitive (GameObject.CreatePrimitive(PrimitiveType.Sphere)).
   - Remove the MeshCollider from each puff (visual only, no physics).
   - Position each puff randomly within the cloud envelope:
     X: random(-cloudWidth*0.4, cloudWidth*0.4)
     Y: random(-cloudHeight*0.3, cloudHeight*0.4) — bias upward for puffy tops
     Z: random(-cloudDepth*0.4, cloudDepth*0.4)
   - Scale each puff:
     Base radius = cloudWidth * random(0.15, 0.35)
     X scale = base * random(0.8, 1.4) — horizontal stretch
     Y scale = base * random(0.5, 0.9) — vertically flattened
     Z scale = base * random(0.8, 1.3)
   - For cumulus: bias bottom puffs flatter (Y scale * 0.5), top puffs rounder.
   - For wisps: all puffs stretched horizontally (X scale * 2.0), very flat (Y scale * 0.3).

5. CLOUD MATERIAL
   Create ONE shared material: Assets/Materials/Cloud.mat
   - Shader: Universal Render Pipeline/Lit
   - Surface Type: Opaque (NOT transparent — opaque white reads better as clouds)
   - Base Color: (0.95, 0.95, 0.97, 1.0) — near-white with slight blue tint
   - Smoothness: 0.1 (matte, diffuse look)
   - Metallic: 0.0
   - Shadow Casting: OFF on all cloud renderers (clouds must not shadow the ground)
   - Receive Shadows: OFF

   Assign this single material to ALL puff renderers.

6. HIERARCHY
   Clouds (root)
   ├── Cloud_0 (cumulus)
   │   ├── Puff_0 (sphere)
   │   ├── Puff_1 (sphere)
   │   └── Puff_N
   ├── Cloud_1 (wisp)
   │   └── ...
   └── Cloud_N

7. PERFORMANCE
   - Total puff count: 200-400 for a 500x500 map.
   - Remove MeshCollider from every puff (just visual).
   - Each puff is a standard Unity sphere (~500 verts) — lightweight.
   - Generation should complete in under 10 seconds.

8. ENVIRONMENT
   - Do NOT change fog settings.
   - Do NOT change lighting settings.
   - Do NOT modify the skybox.

9. REPORT
   Return summary:
   - Cloud count by type (cumulus, wisp, small puff)
   - Total puff (sphere) count
   - Cloud altitude range
   - Material path
   - Confirm: no previous layers modified
```

---

## Expected Result

- **15-25 cloud formations** floating high above the terrain.
- Each cloud is a **cluster of overlapping white spheres** that together read as a single cloud shape.
- **Cumulus clouds** are large, puffy, with rounded tops and flatter bottoms.
- **Wisps** are thin, stretched, horizontal formations.
- **Small puffs** add variety and fill gaps.
- Clouds are **spread across the sky** with some open areas (not all bunched together).
- Clouds do **NOT cast shadows** on the terrain (disabled shadow casting).
- From below, the sky reads as a partly cloudy day — some blue sky visible between clouds.

---

## Scoring Rubric (100 points)

### Cloud Shape Quality (35 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Cluster composition** | 10 | Each cloud is 8-20 overlapping spheres, not a single sphere. Reads as a volumetric shape. |
| **Type variety** | 8 | Mix of cumulus (large puffy), wisps (thin stretched), and small puffs. Not all same shape. |
| **Size variation** | 7 | Clouds range from small wisps to large formations. Not all identical size. |
| **Puff shaping** | 5 | Individual puffs are flattened/stretched, not perfect spheres. Cumulus have flat bottoms. |
| **Overlap quality** | 5 | Puffs overlap enough to merge visually. No obvious individual sphere outlines. |

### Distribution Quality (30 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Sky coverage** | 10 | Clouds spread across full map footprint. Not clustered in one corner. |
| **Open sky areas** | 8 | Some areas have no clouds — partly cloudy, not overcast. |
| **Altitude correctness** | 7 | All clouds well above terrain. No clouds intersecting mountains or floating too low. |
| **Count appropriate** | 5 | 15-25 groups for 500x500. Not too few (empty sky) or too many (overcast). |

### Technical Quality (25 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **No shadow casting** | 8 | Clouds do NOT cast shadows on terrain. Shadow casting disabled on all renderers. |
| **Layer isolation** | 5 | No terrain, water, prop, or lighting modifications. |
| **Hierarchy** | 4 | Clean: Clouds → Cloud_N → Puff_N. |
| **Material** | 4 | Single shared opaque white material. No glow, no transparency artifacts. |
| **Performance** | 4 | 200-400 puffs total. No MeshColliders. Generates in < 10s. |

### Visual Quality (10 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Reads as clouds** | 5 | From ground level, formations look like clouds, not floating balls. |
| **Sky feels natural** | 5 | Mix of cloud sizes and open sky creates a natural partly-cloudy atmosphere. |

### Score Bands

| Range | Meaning |
|-------|---------|
| **0-25** | No clouds or single floating spheres |
| **26-50** | Some cloud clusters but poor distribution or all identical |
| **51-70** | Decent clouds with variety, minor issues |
| **71-85** | Good cloud system with proper shapes, distribution, and no shadows |
| **86-100** | Excellent: natural sky with varied cloud types, proper altitude, clean implementation |

---

## Quick Reference for Agents

| Problem | Solution |
|---------|----------|
| Clouds too low / in terrain | Increase cloud base height. Must be 120+ above terrain max. |
| All clouds same size | Vary cloud type (cumulus/wisp/puff) and randomize envelope dimensions. |
| Clouds look like single balls | Need 8-20 overlapping puffs per cloud. Puffs must overlap significantly. |
| Clouds casting shadows | Set shadowCastingMode = Off on every MeshRenderer. |
| Too many clouds / slow | Reduce cloud count or puffs per cloud. Target 200-400 total puffs. |
| Clouds all in one area | Use grid distribution with jitter across full map footprint. |
| Clouds too uniform | Add 30% skip chance for empty cells. Vary type and size per cloud. |
| Wisps too thick | Reduce Y scale (0.3) and increase X scale (2.0) for wisps. |
| Can see individual spheres | Increase puff count per cloud and/or increase puff sizes so they overlap more. |

---

## Technical Notes

### Why primitives instead of particles or textures?
This benchmark tests whether an AI agent can build 3D objects with spatial reasoning. Particle systems and billboarded textures require different skills (parameter tuning). Primitive-based clouds test mesh placement, scale manipulation, and spatial composition — core skills for procedural world building.

### Why opaque instead of transparent?
Transparent cloud materials create sorting artifacts when many overlapping transparent spheres render on top of each other. Opaque white spheres with a matte finish read cleanly as clouds without depth-sorting issues. The slight blue tint in the white color helps them blend with the sky.

### Shadow casting
Cloud shadows on the ground create harsh, unrealistic dark patches that obscure terrain evaluation. In a benchmark context, ground readability is more important than cloud shadow realism. Shadow casting is disabled to keep the terrain clearly visible.

### Cloud altitude
Clouds below or near mountain peaks look wrong and can intersect terrain. The 120m minimum above terrain max ensures clouds are always clearly in the sky, even when viewed from mountain peaks.
