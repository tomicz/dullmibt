# Darko Unity LLM Intelligence Benchmark — Full Prompt

**Two ways to start:**
- **Repo access:** Point your agent at this file. It has everything.
- **No repo:** Copy this entire file and paste it to your agent as the first message.

---

## Before you begin

Pick a **run ID** — e.g. `claude-code-2026-04-10`, `codex-2026-04-10`, `cursor-2026-04-10`.
Replace every `{run-id}` below with your run ID before sending to the agent.

---

## Agent Instructions

Paste this block to your agent first (fill in run ID and execution path):

```
You are a Unity Editor agent. You will build a complete procedural 3D landscape in Unity by executing all 7 layers below in order, without stopping between layers.

Execution: [MCP — call execute_code for each layer] OR [Scripts — write a C# Editor script per layer that I will run manually in order]

Run output path: Assets/BenchmarkRuns/{run-id}/
Scene path: Assets/BenchmarkRuns/{run-id}/run-scene.unity

Global rules (apply to every layer):
0) SCENE SETUP — do this first, before any layer:
   - Create a brand new empty scene using EditorSceneManager.NewScene(NewSceneSetup.DefaultGameObjects, NewSceneMode.Single).
   - Ensure the run folder exists: AssetDatabase.CreateFolder as needed to create Assets/BenchmarkRuns/{run-id}/.
   - Immediately save the new scene to Assets/BenchmarkRuns/{run-id}/run-scene.unity using EditorSceneManager.SaveScene.
   - Do NOT work in the existing active scene. Do NOT reset or modify any existing scene. Always create fresh.
1) Execute all 7 layers in order. Do not stop between layers unless an error requires human input.
2) ALL assets (textures, materials, settings) must be saved under Assets/BenchmarkRuns/{run-id}/. Never write to Assets/Materials/, Assets/Textures/, or any path outside the run folder.
3) After each layer, verify: hierarchy names exist, MeshCollider on terrain, read console for errors. Save the scene (EditorSceneManager.SaveScene). Report layer completion before proceeding.
4) Scale is ALWAYS (1,1,1) for every GameObject. Map size controlled by localSizeX/Z. Noise uses world-space coordinates directly.
5) Each layer only ADDS to the scene. Never modify previous layers' terrain mesh, textures, water, props, or lighting.
6) FRESH GENERATION: Every texture, material, and mesh must be generated from scratch by your own code. If a file already exists at the target path, delete it first and regenerate.
7) When terrain geometry changes (water system carving), rebake ALL terrain textures before proceeding.
8) Props placed via downward raycast to ProceduralMeshGround only. No floating objects.
9) Output a per-layer summary (counts, key asset paths, material slots assigned) before starting the next layer.

The 7 layers follow. Execute them in order.
```

---

---

# Layer 1 — Terrain + Surface PBR

## World Scale Rules

**1 unit = 1 Unity unit = 1 primitive cube width/length.** The transform scale is ALWAYS `(1, 1, 1)`. Map size is controlled entirely by `localSizeX` and `localSizeZ`.

| Map size | localSizeX | localSizeZ | World footprint | Suggested vertsX/Z |
|----------|-----------|-----------|----------------|-------------------|
| 10x10 | 10 | 10 | 10x10 units | 51x51 |
| 100x100 | 100 | 100 | 100x100 units | 201x201 |
| 500x500 | 500 | 500 | 500x500 units | 401x401 |
| 1000x1000 | 1000 | 1000 | 1000x1000 units | 501x501 |

**Critical rule: Larger maps must generate MORE content, NOT stretch.** Because noise is sampled using world-space coordinates (wx, wz = vertex position directly, no scale multiplication), a 1000x1000 map naturally covers more noise space and produces more mountains, more hills, more rivers. The terrain must never look stretched — a mountain on a 100x100 map should be the same physical size as a mountain on a 1000x1000 map; the larger map simply has room for more of them.

**Vertex density:** Target ~1 vertex per world unit. A 500x500 map needs ~501x501 vertices. A 1000x1000 map needs ~1001x1001 (but 501x501 is acceptable for performance, giving ~2 units per vertex).

---

## Layer 1 Prompt

```text
In the active Unity scene, generate a realistic procedural terrain with mountains, hills, plains, and a river valley using a custom Mesh (NOT Unity Terrain).

Inputs (honor these if specified, otherwise use defaults):
- localSizeX = 500, localSizeZ = 500         // world units (1 unit = 1 Unity unit)
- vertsX = 501, vertsZ = 501                  // ~1 vertex per world unit
- seed = 42                                    // deterministic generation

IMPORTANT SCALE RULE:
- transform.localScale is ALWAYS (1, 1, 1). Never use scale to change map size.
- The map size IS localSizeX x localSizeZ in world units.
- Noise must be sampled using direct world positions (wx = vertex.x, wz = vertex.z).
  No scale multiplication. This ensures larger maps produce MORE features, not stretched ones.
- A 1000x1000 map has ~4x the mountains, hills, rivers of a 500x500 map, NOT the same
  features stretched 2x bigger.

Requirements:

1. CLEANUP
   - Delete existing "ProceduralTerrain", "ProceduralMeshGround", "RiverPath" objects if present.

2. CREATE MESH OBJECT
   - Create GameObject "ProceduralMeshGround" at (0,0,0).
   - transform.localScale = (1, 1, 1). ALWAYS. Never change this.
   - Add MeshFilter, MeshRenderer, MeshCollider.
   - Use UInt32 index format (vertex count likely exceeds 65535).

3. BUILD VERTEX GRID WITH UVs
   - Centered grid: startX = -localSizeX * 0.5, startZ = -localSizeZ * 0.5
   - For each vertex (ix, iz):
     lx = startX + (ix / (vertsX-1)) * localSizeX
     lz = startZ + (iz / (vertsZ-1)) * localSizeZ
     wx = lx    // world-space X (no scale multiplication — scale is 1)
     wz = lz    // world-space Z
   - MANDATORY: Assign UV coordinates to every vertex:
     uv[idx] = Vector2(ix / (vertsX-1), iz / (vertsZ-1))
     Without UVs, textures will not map correctly across the terrain.

4. DOMAIN WARPING (organic shapes, no axis-aligned blobs)
   - Before any noise lookup, warp the sample coordinates:
     float warpX = (PerlinNoise(wx * 0.006 + 100, wz * 0.006 + 200) - 0.5) * 80;
     float warpZ = (PerlinNoise(wx * 0.006 + 300, wz * 0.006 + 400) - 0.5) * 80;
     float swx = wx + warpX;
     float swz = wz + warpZ;

5. ZONE SELECTION (continental noise — determines terrain type)
   - Single low-frequency Perlin layer:
     float zoneVal = PerlinNoise(swx * 0.003 + 500, swz * 0.003 + 700);
   - Zone mapping (0..1 range):
     < 0.25  = Plains
     0.25-0.55 = Hills
     0.55-0.80 = Foothills (transition)
     > 0.80  = Mountains
   - Compute blend weights using smoothstep for soft transitions:
     float plainW  = 1 - smoothstep(0.20, 0.30, zoneVal);
     float hillW   = smoothstep(0.20, 0.30, zoneVal) * (1 - smoothstep(0.50, 0.60, zoneVal));
     float footW   = smoothstep(0.50, 0.60, zoneVal) * (1 - smoothstep(0.75, 0.85, zoneVal));
     float mountW  = smoothstep(0.75, 0.85, zoneVal);
   - Normalize: totalW = plainW + hillW + footW + mountW; divide each by totalW.

6. PER-ZONE NOISE STACKS (each zone has its own character)

   PLAINS (gentle, nearly flat):
   - Base elevation: 1.0
   - 2 octaves: k = [0.02, 0.06], a = [1.5, 0.4]
   - Max local relief: ~4m

   HILLS (rolling, moderate):
   - Base elevation: 12.0
   - 3 octaves: k = [0.008, 0.025, 0.07], a = [12, 5, 1.5]
   - Max local relief: ~35m

   FOOTHILLS (transition):
   - footH = lerp(hillH, mountH, smoothstep(0.55, 0.75, zoneVal))

   MOUNTAINS (dramatic peaks):
   - Base elevation: 45.0
   - 4 octaves: k = [0.005, 0.015, 0.04, 0.12], a = [50, 20, 8, 2]
   - Peak sharpening: mountH = 45.0 + raw * pow(saturate(raw/40 + 0.5), 0.4)

7. COMPOSITE HEIGHT
   - h = (plainW * plainH + hillW * hillH + footW * footH + mountW * mountH) / totalW;

8. RIVER VALLEY (spline-based carving)
   - River must flow OFF the map edges — start and end points BEYOND terrain bounds.
     Example for 500x500: start near (-270, ?, -280), end near (260, ?, 270).
   - For larger maps (1000x1000), generate MULTIPLE rivers.
     Rule of thumb: 1 river per 500x500 area of map.
   - Generate 10-12 intermediate waypoints with sinusoidal meander:
     meander amplitude = 45 world units, frequency along path = sin(t * 4.5)
   - MOUNTAIN AVOIDANCE: After generating waypoints, sample the terrain height at each
     waypoint. If a waypoint lands in a mountain zone (zoneVal > 0.75 or height > 60% of
     max), push it laterally away from the mountain center until it sits in a valley, foothill,
     or plains zone. Rivers flow AROUND mountains, not through them.
   - Smooth path using Catmull-Rom interpolation → dense polyline (200+ segments).
   - Carve channel:
     riverHalfWidth = 18, bankWidth = 22, carveDepth = 4
     U-shaped profile (quadratic: profile = (d/halfWidth)^2) for flat river bottom with gradual walls.
     Bank transition uses cosine falloff (NOT cliff edges).
     River bed elevation descends along spline (flows downhill).
   - Store path on "RiverPath" GameObject for later water layer.

9. FINAL MESH
   - Set vertices, triangles, UVs on mesh.
   - RecalculateNormals(), RecalculateTangents(), RecalculateBounds().
   - Tangents are REQUIRED for normal map lighting to work correctly.
   - Assign mesh to MeshFilter and MeshCollider.

10. TERRAIN TINT (height + slope based coloring)
    - Generate `Assets/BenchmarkRuns/{run-id}/Textures/GroundHeightTint.png` (1024x1024, sRGB).
    - Compute per-pixel: height (h), slope (1 - dot(normal, up)), distance to river.
    - Normalized height: hN = (h - minH) / (maxH - minH).

    Color rules:
    - Rock: slope > 0.12 OR hN > 0.35. Color: (0.45, 0.40, 0.33) to (0.30, 0.26, 0.22).
    - Snow: hN > 0.75 AND slope < 0.25. Blend to (0.92, 0.93, 0.96).
    - River bed: distToRiver < 18 → (0.25, 0.18, 0.12)
    - River bank: distToRiver 18-40 → blend with quadratic ease.
    - Grass/Plains: remaining → (0.22, 0.50, 0.12) to (0.28, 0.55, 0.16).

    Apply sun exposure: color *= lerp(0.82, 1.18, dot(normal, lightDir)^2).
    Assign to Assets/BenchmarkRuns/{run-id}/Materials/Ground.mat _BaseMap.

11. BAKE PBR MAPS
    - `Assets/BenchmarkRuns/{run-id}/Textures/GroundHeightMap.png` (1024x1024, linear grayscale)
    - `Assets/BenchmarkRuns/{run-id}/Textures/GroundNormalMap.png` (2048x2048, normal map type)
      Finite-difference bake, assign _BumpMap, _BumpScale = 0.8.

12. UV + MATERIAL SETUP
    - UVs MUST be assigned to mesh before textures will display.
    - All material texture tiling MUST be (1, 1). No tiling/repeat.
    - Ground.mat: _BaseMap = GroundHeightTint, _BumpMap = GroundNormalMap.
    - _BumpScale = 0.8, Smoothness = 0.06, Metallic = 0.0, _BaseColor = white.
    - RecalculateTangents() MUST be called for normal maps to function.

13. ENVIRONMENT
    - Fog MUST be disabled (RenderSettings.fog = false).

14. SAVE AND REPORT
    - Return: map size, height range, zone coverage %, river count and lengths,
      texture paths, material assignments.
```

---

## Layer 1 Scoring Rubric (100 base points)

### Structural Quality (50 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Zone differentiation** | 10 | Mountains, hills, and plains are visually distinct. |
| **Mountain quality** | 8 | Rocky coloring on slopes/peaks. Multiple peaks. Domain warping. |
| **Hill quality** | 7 | Rolling terrain 10-30m relief. Smooth transitions. |
| **Plains quality** | 5 | Flat areas, max 4m variation. Bright green. |
| **River valley** | 10 | Meanders off map edges. Flows around mountains. Smooth banks. |
| **Zone transitions** | 5 | No hard edges. smoothstep blending. |
| **Domain warping** | 5 | Organic shapes. No grid artifacts. |

### Technical Quality (30 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Mesh + UVs** | 5 | Correct vertex count. UVs assigned. UInt32 format. Tangents calculated. |
| **World-space noise** | 5 | Noise uses wx/wz directly. Scale=(1,1,1). |
| **Terrain tint** | 5 | Rock on slopes/peaks. Snow on tops. Green plains. Brown river. |
| **Normal map** | 5 | Baked, assigned to _BumpMap. Tiling=(1,1). |
| **Height map** | 5 | Baked grayscale heightmap. Linear. |
| **Material setup** | 2 | All tiling=(1,1). _BaseColor=white. Fog off. |
| **Performance** | 3 | Generation under 15 seconds. |

### Visual Quality (20 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Landscape readability** | 8 | Terrain looks like a real place from any camera angle. |
| **Color coherence** | 8 | Rock reads as rock. No green mountain slopes. |
| **Sun influence** | 4 | Directional tint baked in. |

---

---

# Layer 2 — Terrain Splat Mapping

## Prerequisites

- `ProceduralMeshGround` must already exist with a valid mesh and MeshCollider.
- Layer 1 must be completed.

## What is a splat map?

A **splat map** is a texture where each color channel (R, G, B, A) stores the blend weight for a different terrain material.

| Channel | Terrain Type | Where it dominates |
|---------|-------------|-------------------|
| **R** | Grass | Plains, gentle hills |
| **G** | Rock | Steep slopes (>0.35), high altitude (>45m) |
| **B** | Dirt/Earth | River bed, river banks |
| **A** | Snow | Mountain peaks (>55m), gentle slopes |

---

## Layer 2 Prompt

```text
Apply terrain splat mapping to the existing ProceduralMeshGround in the active Unity scene.

Three steps:
A) Generate the splat map texture from terrain data
B) Generate tileable PBR texture sets for each terrain zone
C) Bake composite terrain textures by blending zone textures weighted by the splat map

1. READ TERRAIN DATA
   - Find "ProceduralMeshGround" — get its MeshFilter mesh vertices, normals, and transform scale.
   - Compute per-vertex: height (vertex.y), slope (1 - dot(normal, up)).
   - Find "RiverPath" if it exists — reconstruct the river polyline from child transforms.
   - Record height range (minH, maxH) from mesh vertices.

2. GENERATE SPLAT MAP (Assets/BenchmarkRuns/{run-id}/Textures/TerrainSplatMap.png)
   - Resolution: 1024x1024, RGBA32, linear color space.
   - For each pixel, map to terrain vertex grid (bilinear interpolation) and compute:
     height, slope, and distance to river.

   Channel weight rules (order of priority, highest first):

     SNOW (A channel):
       weight = smoothstep(50, 60, height) * (1 - smoothstep(0.2, 0.35, slope))

     DIRT (B channel):
       If river exists:
         riverBed: distance < 10 → weight = 1.0
         riverBank: distance 10-22 → weight = smoothstep(22, 10, distance) * 0.8
       Also add dirt on very steep low-altitude areas:
         weight += smoothstep(0.5, 0.7, slope) * smoothstep(30, 0, height) * 0.4

     ROCK (G channel):
       weight = max(smoothstep(0.3, 0.55, slope), smoothstep(40, 55, height) * 0.6)
       weight *= (1 - snowWeight) * (1 - dirtWeight * 0.5)

     GRASS (R channel):
       weight = max(0, 1 - snowWeight - rockWeight - dirtWeight)

   - Normalize: total = R + G + B + A. If total > 0, divide each by total.
   - Import settings: sRGB OFF (linear), isReadable = true.

3. GENERATE TILEABLE PBR TEXTURE SETS (procedural, 512x512 each)

   GRASS textures (Assets/BenchmarkRuns/{run-id}/Textures/SplatGrass*.png):
   - Albedo: Base green (0.28, 0.52, 0.18) with multi-octave noise variation.
   - Normal: Vertical blade-like pattern. Frequency: k=0.15.
   - Mask: R=metallic~0, G=AO 0.6-1.0, A=smoothness 0.15-0.35.

   ROCK textures (Assets/BenchmarkRuns/{run-id}/Textures/SplatRock*.png):
   - Albedo: Gray-brown base (0.38, 0.36, 0.32) with craggy noise.
   - Normal: Strong normals (normalStrength 2.0-3.0).
   - Mask: metallic~0.02, AO 0.4-1.0, smoothness 0.15-0.30.

   DIRT textures (Assets/BenchmarkRuns/{run-id}/Textures/SplatDirt*.png):
   - Albedo: Earth brown base (0.32, 0.24, 0.16).
   - Normal: Gentle normals, normalStrength ~1.0.
   - Mask: metallic~0, AO 0.7-1.0, smoothness 0.20-0.45.

   SNOW textures (Assets/BenchmarkRuns/{run-id}/Textures/SplatSnow*.png):
   - Albedo: Near-white base (0.92, 0.93, 0.96).
   - Normal: Very soft normals, normalStrength ~0.5.
   - Mask: metallic~0, AO 0.85-1.0, smoothness 0.40-0.65.

   Import settings for all:
   - Albedo: sRGB ON, wrap mode = Repeat
   - Normal: texture type = Normal Map, wrap mode = Repeat
   - Mask: sRGB OFF (linear), wrap mode = Repeat

4. BAKE COMPOSITE TERRAIN TEXTURES
   Tiling factor = 40 for 512px tiles across 1024px output.

   COMPOSITE ALBEDO (Assets/BenchmarkRuns/{run-id}/Textures/GroundHeightTint.png — replaces existing, 2048x2048):
     color = grassAlbedo * R + rockAlbedo * G + dirtAlbedo * B + snowAlbedo * A
     Apply sun exposure tint: color *= lerp(0.88, 1.12, smoothstep(0.2, 0.8, sunExposure))

   COMPOSITE NORMAL (Assets/BenchmarkRuns/{run-id}/Textures/GroundNormalMap.png — replaces existing, 2048x2048):
     n = normalize(grassNormal * R + rockNormal * G + dirtNormal * B + snowNormal * A)

   COMPOSITE MASK (Assets/BenchmarkRuns/{run-id}/Textures/GroundMaskMap.png — new, 1024x1024):
     R = blended metallic, G = blended AO, B = 0, A = blended smoothness.

5. ASSIGN TO MATERIAL
   - Load Assets/BenchmarkRuns/{run-id}/Materials/Ground.mat
   - Set _BaseMap = composite albedo
   - Set _BumpMap = composite normal, _BumpScale = 1.0
   - Set _MaskMap = composite mask
   - Set _BaseColor = white
   - Mark material dirty, save assets.

6. REPORT
   Return: splat map path, channel coverage %, tileable texture paths, composite texture paths,
   material assignments confirmed.
```

---

## Layer 2 Scoring Rubric (100 points)

| Category | Points |
|----------|--------|
| Splat Map Quality (channel separation, transitions, river awareness, normalization) | 30 |
| Tileable Texture Quality (visual identity, tileability, PBR correctness, color coherence) | 30 |
| Composite Bake Quality (blending, tiling detail, normal composition, sun exposure) | 25 |
| Technical Quality (texture specs, material setup, performance) | 15 |

---

---

# Layer 3 — Water System

## Critical Rule

**The water layer must NEVER modify terrain vertices, normals, UVs, or any terrain textures.** Water surfaces sit on the terrain — they do not carve or reshape it.

## Prerequisites

- `ProceduralMeshGround` must exist with valid mesh, UVs, and MeshCollider.
- `RiverPath` must exist (from Layer 1).

---

## Layer 3 Prompt

```text
Generate water surfaces on the existing ProceduralMeshGround in the active Unity scene.
This adds visible water meshes that sit ON TOP of the terrain. Do NOT modify the terrain mesh.

IMPORTANT RULES:
- DO NOT modify ProceduralMeshGround vertices, normals, UVs, or collider. Terrain is READ-ONLY.
- DO NOT rebake any terrain textures. They are final.
- transform.localScale is ALWAYS (1, 1, 1) for all water objects.

1. READ TERRAIN AND RIVER DATA
   - Find "ProceduralMeshGround" — get MeshFilter mesh bounds.
   - Find "RiverPath" — reconstruct the river polyline from child transforms.
   - Build a dense polyline using Catmull-Rom interpolation (200+ segments).

2. GENERATE RIVER WATER SURFACE (mandatory)
   - Walk the RiverPath polyline. At each sample point:
     - Raycast DOWN to find terrain surface height at river center.
     - Water surface Y = terrainHeight + 0.3.
     - River half-width = 14-18 (match the carved channel).
   - Build a continuous quad-strip mesh along the entire path.
   - Smooth the water Y values (3-5 passes of neighbor averaging).
   - Mesh MUST extend beyond terrain bounds at both ends.
   - Create GameObject "RiverWater" under parent "Water".

3. WATER MATERIAL
   - Path: Assets/BenchmarkRuns/{run-id}/Materials/Water.mat
   - URP Lit, Surface Type = Transparent
   - Base Color = (0.12, 0.28, 0.40, 0.80)
   - Smoothness = 0.95, Metallic = 0.1
   - ZWrite = 0, SrcBlend = SrcAlpha, DstBlend = OneMinusSrcAlpha

4. OPTIONAL: LAKES (only for maps > 500x500 with suitable terrain)
   - Scan for flat low areas (avgHeight < 20% of height range, slope < 0.15).
   - Lake = flat circular water surface at terrain minimum + 0.3.
   - Use fan mesh with 32-48 edge vertices.
   - Create as "Lake_N" under "Water" parent.

5. OPTIONAL: PONDS (bonus for large maps)
   - Only if map > 500x500, radius 8-18.
   - Create as "Pond_N" under "Water" parent.

6. EXCLUSION ZONE DATA
   - Create "WaterExclusionZones" under "Water" parent.
   - For each water body, add a child GameObject with naming:
     "River_Exclusion_margin5_halfwidth16"
     "Lake_0_r35_margin8"
     "Pond_0_r12_margin6"

7. ENVIRONMENT
   - Fog MUST remain disabled (RenderSettings.fog = false).

8. REPORT
   Return: river length/width/vertex count, lakes count, ponds count, material path,
   exclusion zones stored, confirm terrain was NOT modified.
```

---

## Layer 3 Scoring Rubric (100 points)

| Category | Points |
|----------|--------|
| River Quality (follows channel, elevation, extends off map, width, smooth flow, visual) | 50 |
| Optional Water Bodies (lake/pond placement, shape, count scaling) | 20 |
| Technical Quality (terrain untouched, scale, material, hierarchy, exclusion data, performance) | 30 |

---

---

# Layer 4 — Props (Trees & Rocks)

## Critical Rule

**Do NOT modify terrain mesh, terrain textures, water surfaces, or any previous layer output.** All props placed ON TOP via raycasting.

## Prerequisites

- `ProceduralMeshGround` with MeshCollider.
- `WaterExclusionZones` from Layer 3.
- `RiverPath` from Layer 1.

---

## Layer 4 Prompt

```text
Generate all props (trees, rocks) on the existing ProceduralMeshGround.
Each prop is a unique procedural mesh. Do NOT modify terrain, water, or any previous layer output.

IMPORTANT RULES:
- transform.localScale is ALWAYS (1, 1, 1) for all objects.
- Every prop uses a UNIQUE seed (baseSeed + propIndex).
- Raycast DOWN from above to find ground position.
- Respect WaterExclusionZones.
- No props on steep slopes (slope > 0.5) or mountain peaks (height > 85% of max).

1. READ TERRAIN AND EXCLUSION DATA
   - Find "ProceduralMeshGround" — get mesh bounds.
   - Find "WaterExclusionZones" — parse river halfwidth, margin, lake/pond data.
   - Find "RiverPath" — reconstruct river polyline.
   - Compute terrain height range.

2. PLACEMENT RULES
   Height zones (normalized height 0-1):
   - hN < 0.05: no trees
   - hN 0.05-0.60: full density
   - hN 0.60-0.75: reduced density (50%)
   - hN > 0.75: no trees

   Slope rules:
   - slope < 0.35: full density
   - slope 0.35-0.50: reduced density (50%)
   - slope > 0.50: no trees

   Water exclusion: no trees within riverHalfWidth + exclusionMargin or lakeRadius + exclusionMargin.
   Minimum spacing between trees: 4-8m.

3. DENSITY AND DISTRIBUTION
   Target: 1 tree per 10x10m cell. ~400-600 trees for a 500x500 map.
   Grid-based with jitter (+-40% of cell size).
   Density noise (k=0.01): noise > 0.4 place, noise < 0.4 skip (natural clearings).

4. TREE GENERATION (per tree)
   Unique seed: baseSeed(12345) + treeIndex.
   Use recursive branching algorithm:

   PARAMETERS (randomize +-20% per tree, 1 unit = 1 meter):
   - Trunk height: 5-8m, base radius: 0.25-0.45m, top radius: 0.04-0.08m
   - Recursion depth: 4-5, main branches from trunk: 4-6
   - Branch start height: 35-50% up trunk
   - Child branch length = parent * 0.55-0.75
   - Child branch radius = parent end * 0.45-0.65
   - Branch angle: 22-45 degrees
   - Taper exponent: 1.3 (exponential)
   - Gravity droop: depth * 0.06
   - Random perturbation: +-0.08 on direction vector

   TRUNK AND BRANCH MESH:
   - Each segment: tapered cylinder with 6 radial segments
   - Trunk: 8-10 length segments, main branches: 4-5, sub-branches: 2-3
   - Vertex colors: bark brown RGB(0.22, 0.13, 0.06) * random(0.8, 1.2)

   LEAF CLUSTERS (cross-billboard):
   - Each cluster = 3 intersecting quads at 0, 60, 120 degree rotations
   - Quad size: 1.0-2.0m
   - Place at branch endpoints and along branches (50% point)
   - Depth 3+ branches get leaf clusters
   - Vertex colors: RGB(0.08, 0.25, 0.03) * random(0.6, 1.0)
   - Double-sided rendering.

   MATERIALS (create ONCE, reuse across all trees):
   - Assets/BenchmarkRuns/{run-id}/Materials/TreeBark.mat
     Shader: Universal Render Pipeline/Baked Lit
     _BaseColor: (0.32, 0.20, 0.10, 1.0), _Cull: 2
   - Assets/BenchmarkRuns/{run-id}/Materials/TreeLeaves.mat
     Shader: Universal Render Pipeline/Baked Lit
     _BaseColor: (0.15, 0.35, 0.08, 1.0), _Cull: 0, _AlphaClip: 1

   HIERARCHY per tree:
   Tree_N (empty parent, localScale=1,1,1)
   ├── Trunk (MeshFilter + MeshRenderer, TreeBark.mat)
   └── Leaves (MeshFilter + MeshRenderer, TreeLeaves.mat)

5. ROCK GENERATION
   Place in rocky zones: slope > 0.25 OR height > 60% of max.
   Also scatter in plains and near river banks.
   Target: 200-400 rocks for 500x500. Cell size: 8m with jitter.

   ROCK SHAPES: mostly spheres and some cubes, flatten Y scale for grounded silhouettes.
   Vary size: 0.3-2m radius. Random yaw and slight tilt.
   After raycast, embed rock into ground by ~12-26% of its world-space height.
   Water exclusion margin: 50% of tree margin (rocks can be near shoreline).

   MATERIAL: Assets/BenchmarkRuns/{run-id}/Materials/RockGray.mat
   Shader: Universal Render Pipeline/Baked Lit, low smoothness, neutral gray.

6. HIERARCHY
   Props (root)
   ├── Trees
   │   └── Tree_N → Trunk + Leaves
   └── Rocks
       └── Rock_N

7. REPORT
   Return: trees placed, rocks placed, total vertex count, seed range,
   confirm terrain NOT modified, confirm water NOT modified.
```

---

## Layer 4 Scoring Rubric (100 points)

| Category | Points |
|----------|--------|
| Placement Quality (height zones, water exclusion, slope, distribution, spacing, grounding) | 40 |
| Prop Quality (tree uniqueness, tree structure, rock variety, materials) | 30 |
| Technical Quality (layer isolation, scale, hierarchy, performance) | 20 |
| Visual Quality (populated landscape, density, shadows) | 10 |

---

---

# Layer 5 — Lighting & Post-Processing

## Critical Rule

**Do NOT modify terrain, water, props, or any previous layer output.** This layer only configures rendering settings.

## Prerequisites

- Layers 1-4 completed. URP active. Camera in scene.

---

## Layer 5 Prompt

```text
Configure lighting, shadows, and post-processing for the existing procedural terrain scene.
Do NOT modify terrain, water, props, or any previous layer output.

1. DIRECTIONAL LIGHT (Sun)
   - Find or create Directional Light named "Sun".
   - Rotation: (45, 150, 0).
   - Color: (1.0, 0.97, 0.92) — warm, NOT pure white.
   - Intensity: 3.0.
   - Shadow type: Soft Shadows.
   - Shadow resolution: 2048.
   - Shadow distance: 300.
   - Shadow cascades: 4.
   - Shadow bias: 0.05, normal bias: 0.4.

2. AMBIENT LIGHTING
   - Ambient mode: Gradient.
   - Sky color: (0.85, 0.88, 0.95).
   - Equator color: (0.70, 0.72, 0.68).
   - Ground color: (0.45, 0.42, 0.38).
   - Ambient intensity: 1.5.

3. SKYBOX
   - Procedural skybox: Sun Size = 0.04, Convergence = 5, Atmosphere Thickness = 1.0, Exposure = 1.3.
   - Sky tint: (0.55, 0.65, 0.85). Ground tint: (0.30, 0.28, 0.25).

4. POST-PROCESSING VOLUME
   - Create Global Volume "PostProcessVolume".
   - Create NEW Volume Profile: Assets/BenchmarkRuns/{run-id}/Settings/TerrainPostProcessProfile.asset.
   - Set weight = 1.0, Priority = 1.
   - Camera must have renderPostProcessing = true.

   TONEMAPPING: Mode = ACES.

   BLOOM: Threshold = 1.5, Intensity = 0.15, Scatter = 0.6.

   COLOR ADJUSTMENTS:
   - Post Exposure = 0.6, Contrast = 5, Saturation = 8.
   - Color Filter = (1.0, 0.98, 0.95).

   VIGNETTE: Intensity = 0.05, Smoothness = 0.4.

   AMBIENT OCCLUSION (SSAO): Intensity = 0.5, Radius = 0.3, Direct Lighting Strength = 0.25.
   Skip if SSAO not available in current URP version.

5. SHADOW SETTINGS
   Configure URP pipeline asset if accessible: Shadow Distance 300, Cascades 4, Soft Shadows on.

6. FOG (optional)
   - Default: RenderSettings.fog = false.
   - If enabled: Linear, start = 250, end = 600, color = (0.60, 0.68, 0.78).

7. CAMERA SETUP
   - Near clip: 0.3, Far clip: 1500, FOV: 60.
   - renderPostProcessing = true, SMAA or FXAA antialiasing, quality = High.

8. REPORT
   Return: sun settings, ambient settings, post-processing overrides enabled,
   fog state, camera settings, confirm no previous layers modified.
```

---

## Layer 5 Scoring Rubric (100 points)

| Category | Points |
|----------|--------|
| Lighting Quality (sun direction/color/intensity, shadows, ambient, skybox) | 35 |
| Post-Processing Quality (tonemapping, bloom, color grading, vignette, SSAO, volume setup) | 35 |
| Environment Quality (fog, camera setup, overall atmosphere) | 20 |
| Technical Quality (layer isolation, asset cleanliness, performance) | 10 |

---

---

# Layer 6 — Sky & Clouds

## Critical Rule

**Do NOT modify terrain, water, props, or lighting.** This layer only ADDS cloud GameObjects.

## Prerequisites

- Layers 1-4 completed. `ProceduralMeshGround` must exist for terrain bounds.

---

## Layer 6 Prompt

```text
Generate a procedural cloud system using primitive meshes in the active Unity scene.
Clouds are clusters of overlapping scaled spheres — NOT particles, NOT Unity terrain clouds.
Do NOT modify terrain, water, props, or lighting. This layer only ADDS cloud objects.

IMPORTANT RULES:
- Parent cloud group uses scale (1,1,1). Individual puffs may use scale for shaping.
- Clouds must be minimum 120m above highest terrain point.
- Clouds cast NO shadows on the ground.
- Each cloud is a cluster of 8-20 overlapping spheres.

1. READ TERRAIN DATA
   - Find "ProceduralMeshGround" — get mesh bounds for world extents and max height.

2. CLOUD ALTITUDE
   - Cloud base height = terrain max height + 120.
   - Cloud height band = 30m.

3. CLOUD DISTRIBUTION
   Target: 15-25 cloud groups for 500x500 map (~1 per 100x100 area).
   Grid with large jitter (cell = mapSize/5, jitter +-40%). 30% skip chance for open sky.

   Cloud types:
   - Cumulus (60%): Width 30-60m, Height 15-25m, Depth 25-45m.
   - Wisp (25%): Width 40-80m, Height 5-10m, Depth 15-25m.
   - Small puff (15%): Width 10-20m, Height 8-12m, Depth 10-18m.

4. PER-CLOUD GENERATION
   Unique seed: baseSeed(99999) + cloudIndex.
   Create parent "Cloud_N" under "Clouds" (position at grid center+jitter, scale 1,1,1).

   Puffs (child spheres):
   - GameObject.CreatePrimitive(PrimitiveType.Sphere).
   - Remove MeshCollider from each puff.
   - Position randomly within cloud envelope (bias upward for puffy tops).
   - Scale: base = cloudWidth * random(0.15, 0.35), X * random(0.8,1.4), Y * random(0.5,0.9).
   - Cumulus: bottom puffs flatter (Y*0.5), top puffs rounder.
   - Wisps: X * 2.0, Y * 0.3.

5. CLOUD MATERIAL
   ONE shared material: Assets/BenchmarkRuns/{run-id}/Materials/Cloud.mat
   - URP Lit, Opaque (NOT transparent).
   - Base Color: (0.95, 0.95, 0.97, 1.0), Smoothness: 0.1, Metallic: 0.0.
   - Shadow Casting: OFF. Receive Shadows: OFF.

6. HIERARCHY
   Clouds (root)
   └── Cloud_N
       └── Puff_N (spheres)

7. REPORT
   Return: cloud count by type, total puff count, altitude range, material path,
   confirm no previous layers modified.
```

---

## Layer 6 Scoring Rubric (100 points)

| Category | Points |
|----------|--------|
| Cloud Shape Quality (cluster composition, type variety, size variation, puff shaping, overlap) | 35 |
| Distribution Quality (sky coverage, open sky areas, altitude, count) | 30 |
| Technical Quality (no shadow casting, layer isolation, hierarchy, material, performance) | 25 |
| Visual Quality (reads as clouds, natural sky feel) | 10 |

---

---

# Layer 7 — Horizon Closure (Mountain Ring)

## Critical Rule

**Do NOT modify terrain, water, props, lighting, or clouds.** This layer only ADDS mountain ring meshes outside the playable terrain.

## Prerequisites

- Layer 1 (terrain) must be completed. `ProceduralMeshGround` must exist.

---

## Layer 7 Prompt

```text
Generate a procedural mountain ring around the edges of the existing terrain to close the horizon.
The ring hides the void beyond the map edges. Do NOT modify any previous layer output.

IMPORTANT RULES:
- transform.localScale is ALWAYS (1, 1, 1) for the ring parent.
- Ring sits OUTSIDE terrain bounds (does not overlap playable area).
- Mountains must blend visually with terrain edges.
- Ring must be continuous — no gaps.

1. READ TERRAIN DATA
   - Find "ProceduralMeshGround" — get mesh bounds.
   - Compute terrain world bounds: minX, maxX, minZ, maxZ.
   - Get terrain height at edges and terrain max height.

2. RING GEOMETRY
   Ring dimensions:
   - Inner edge: overlaps 20m INTO terrain bounds for seamless blending.
   - Outer edge: 150-250m beyond terrain bounds.
   - Build as single continuous mesh, 300-400 perimeter sample points, 25-30 radial steps.

   Per radial strip:
   - Inner vertices: height = terrain height at that XZ (seamless weld).
   - Height by distance from edge:
     3 octaves, k=[0.008, 0.02, 0.06], a=[40, 15, 5].
     Height multiplier = smoothstep(0, ringWidth, distFromEdge).
     Peak height: 60-120% of terrain max height.
   - Outermost row: Y = -20 (hides bottom edge).

3. RING MATERIAL
   - Use the SAME Assets/BenchmarkRuns/{run-id}/Materials/Ground.mat from terrain.
   - Map UVs to terrain UV space for seamless texture extension.
   - Call RecalculateTangents().

4. RING FOREST
   - Add temporary MeshCollider to ring mesh for raycasting.
   - Scatter trees in band from 15m inside terrain edge to 100m outside.
   - Noise-based density (k=0.015, threshold 0.35). Cell size: 8m.
   - Target: 1500-2500 ring trees for 500x500 map.
   - Smaller trees (4-6.5m trunk height) — distance trees.
   - Skip: height < 0, height > 80% of max, slope > 0.45.
   - Use same TreeBark.mat and TreeLeaves.mat. Add under existing "Trees" parent.
   - Remove temporary MeshCollider when done.

5. HIERARCHY
   HorizonRing (root, position 0,0,0, scale 1,1,1)
   └── Ring_Full (single continuous mesh)

6. REPORT
   Return: vertex count, ring tree count, ring width, height range of ring mountains,
   material used, confirm seamless blend, confirm no previous layers modified.
```

---

## Layer 7 Scoring Rubric (100 points)

| Category | Points |
|----------|--------|
| Geometry Quality (continuous ring, mountain shapes, height variation, corner coverage, outer edge) | 35 |
| Blending Quality (height match at seam, no overlap artifacts, visual consistency) | 30 |
| Technical Quality (layer isolation, hierarchy, performance, scale) | 20 |
| Visual Quality (reads as mountains, atmospheric blending) | 15 |

---

---

## Overall Scoring Guide

| Layer | Max Score |
|-------|-----------|
| Layer 1 — Terrain | 100 (× map size multiplier: 0.5x–1.3x) |
| Layer 2 — Splat Map | 100 |
| Layer 3 — Water | 100 |
| Layer 4 — Props | 100 |
| Layer 5 — Lighting | 100 |
| Layer 6 — Sky & Clouds | 100 |
| Layer 7 — Horizon | 100 |
| **Total** | **700** |

| Score | Meaning |
|-------|---------|
| 0–2 per layer | Broken scene, compile errors, floating objects, wrong scale |
| 3–5 per layer | Layer incomplete or significant errors |
| 6–8 per layer | Layer complete with minor issues |
| 9–10 per layer | All criteria met, verified, reproducible |

---

*Benchmark by Darko Tomic. Community: [darkounity.com](https://darkounity.com)*
