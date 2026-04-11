# Darko Unity LLM Intelligence Benchmark — Full Prompt

**Two ways to start:**
- **Repo access:** Point your agent at this file. It has everything.
- **No repo:** Copy this entire file and paste it to your agent as the first message.

---

## Learn Unity with others

If you are running this benchmark, you are exactly the kind of builder who benefits from structured learning and peer feedback.

Visit **[darkounity.com](https://darkounity.com)** — the **Dark Unity** learning community. A place to **learn Unity** with guidance, share progress, and connect with people who care about shipping real projects. Whether you pass or fail this benchmark, come learn Unity there: fundamentals, URP, tooling, and project discipline compound faster when you are not alone.

---

## Before you begin

The benchmark auto-generates a run ID based on the agent name + today's date, so you don't have to pick one manually. You CAN override it by specifying a custom ID in the prompt below if you want.

---

## Agent Instructions

Paste this block to your agent first:

```
You are a Unity Editor agent. You will build a complete procedural 3D landscape in Unity by executing all 7 layers below in order, without stopping between layers.

Run ID: [leave blank to auto-generate, OR specify a custom ID like "my-test-run"]

STEP 1 — EXECUTION MODE DETECTION (do this first, before anything else):
Check your available tools for Unity MCP. Look for any tool that lets you execute C# code inside the Unity Editor at runtime — typically named "execute_code", "unity_execute", "mcp__unity__execute_code", or similar, provided by a Unity MCP server.

- If Unity MCP IS available: YOU MUST use it. Execute every layer by calling the MCP execute_code tool with inline C#. Do NOT write .cs files to the project. Do NOT create an Editor runner script. Do NOT use AssetDatabase imports of your own scripts. This is non-negotiable when MCP is present — the benchmark is explicitly designed to test runtime editor manipulation via MCP, not traditional scripting workflows. Writing script files when MCP is available is a failure mode and invalidates the run.
- If Unity MCP is NOT available: fall back to writing C# Editor scripts under Assets/BenchmarkRuns/{run-id}/Editor/ that the user will run manually via [MenuItem] entries. State clearly at the start that you are in script-fallback mode because no MCP tool was found.

Announce your detected execution mode (MCP or script-fallback).

STEP 2 — SCENE SETUP (the VERY NEXT thing, before Layer 1 and before any generation code):
This step is critical. If MCP is available, the VERY FIRST execute_code call you make must be this scene setup — nothing else comes before it. Do not generate terrain, materials, or anything else until the new scene is created, opened as active, and saved.

   a) Determine the run ID:
      - If the user specified a custom Run ID above, use it.
      - Otherwise, auto-generate: "{your-name}-{YYYY-MM-DD}".
        - your-name: your best honest self-identification as a single lowercase slug, in this
          order of preference:
            1) Exact model if you know it (e.g. "claude-opus-4-6", "gpt-5", "gemini-2-5-pro")
            2) Model family if unsure of version (e.g. "claude-opus", "gpt", "gemini")
            3) Wrapper/tool name if you don't know the model (e.g. "codex", "cursor",
               "windsurf", "claude-code", "copilot")
            4) "unknown-agent" as a last resort
          Use hyphens, no spaces. Do not guess versions you don't actually know.
        - YYYY-MM-DD: get from C# System.DateTime.Now.ToString("yyyy-MM-dd").
      - Collision handling: if Assets/BenchmarkRuns/{run-id}/ already exists, append
        "-HHmm" from System.DateTime.Now to make it unique.
      - Announce the final run ID to the user.

   b) Create the run folder BEFORE creating the scene:
      - Use AssetDatabase.CreateFolder recursively to ensure Assets/BenchmarkRuns/{run-id}/ exists.
      - Call AssetDatabase.Refresh() after creating folders.

   c) Create a brand new empty scene AND open it as the active scene:
      var newScene = EditorSceneManager.NewScene(NewSceneSetup.DefaultGameObjects, NewSceneMode.Single);
      - NewSceneMode.Single REPLACES the current scene and makes the new scene active. This is intentional — the existing scene must NOT be modified.
      - Verify: EditorSceneManager.GetActiveScene() should return the new scene.

   d) Immediately save the new scene to disk at the run folder path:
      EditorSceneManager.SaveScene(newScene, "Assets/BenchmarkRuns/{run-id}/run-scene.unity");
      - This makes the scene persistent and tied to the run folder.
      - After saving, verify the file exists at that path before continuing.

   e) Verify the active scene is now the run scene:
      EditorSceneManager.GetActiveScene().path must equal "Assets/BenchmarkRuns/{run-id}/run-scene.unity".
      If it does not, STOP and report the failure. Do not proceed to Layer 1.

   f) HARD RULE: do NOT work in, modify, or reset any pre-existing scene. Always create fresh. If the previous active scene had unsaved changes, discard them (NewSceneMode.Single handles this — do not prompt).

Only after scene setup is complete and verified, proceed to Layer 1.

Global rules (apply to every layer after scene setup):
0) Before starting Layer 1, confirm scene setup (Step 2 above) completed successfully and the active scene is the run scene.
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

# Layer 1 — Terrain Mesh + Hydrology + Data Side-Channels

Layer 1 is **only** the bare terrain mesh, the river network carved into it, and the data side-channels that downstream layers will read. **No PBR. No splat. No grass. No tinting.** The Ground material this layer assigns is a flat gray placeholder; Layer 2 replaces its maps.

## World Scale Rules

**1 unit = 1 Unity unit.** The transform scale is ALWAYS `(1, 1, 1)`. Map size is controlled entirely by `localSizeX` and `localSizeZ`.

| Map size | localSizeX | localSizeZ | World footprint | Suggested vertsX/Z |
|----------|-----------|-----------|----------------|-------------------|
| 100x100 | 100 | 100 | 100x100 units | 101x101 |
| 250x250 | 250 | 250 | 250x250 units | 251x251 |
| 500x500 | 500 | 500 | 500x500 units | 501x501 |
| 1000x1000 | 1000 | 1000 | 1000x1000 units | 501x501 (2u/vert) |

**Critical rule: Larger maps must generate MORE content, NOT stretch.** Noise frequencies are expressed in cycles-per-meter and are sampled at world-space `(wx, wz)`. A mountain on a 100x100 map is the same physical size as a mountain on a 1000x1000 map — the larger map simply contains more of them.

**Vertex density:** target roughly 1 vertex per world unit, capped at 501×501 for performance.

## Random Seed Rule

**Layer 1 must use a fresh random seed on every run.** This is intentional: the benchmark exists to evaluate models against terrains they have never seen. A fixed seed lets a model overfit one map.

- Seed source: `unchecked((int)System.DateTime.Now.Ticks)`.
- The actual integer used MUST be written into `terrain.json` so any run can be replayed for debugging.
- Every randomized step in the pipeline (FBM offsets, droplet starts, river noise, etc.) must seed from this single value (typically via `System.Random(seed)`), not from `UnityEngine.Random` global state.

---

## Layer 1 Prompt

```text
In the active Unity scene, generate a realistic procedural terrain with mountains, valleys, plains, and a river network using a custom Mesh (NOT Unity Terrain).

Layer 1 is terrain mesh + rivers + data side-channels ONLY.
Do NOT bake PBR/splat textures, do NOT create grass, do NOT tint by height.
Layer 2 will replace the placeholder material's maps. Layer 3 will add water surfaces.

Inputs (honor these if specified, otherwise use defaults):
- localSizeX = 500, localSizeZ = 500     // world units
- vertsX    = 501, vertsZ    = 501       // ~1 vertex per world unit
- seed      = unchecked((int)System.DateTime.Now.Ticks)   // RANDOM per run, recorded to JSON

IMPORTANT SCALE RULE:
- transform.localScale is ALWAYS (1, 1, 1).
- Noise is sampled using direct world coordinates (wx = vertex.x, wz = vertex.z).
- Larger maps produce MORE features, never stretched ones.

Requirements:

1. CLEANUP
   - Delete existing "ProceduralTerrain", "ProceduralMeshGround", "RiverPath" GameObjects if present.
   - Delete any pre-existing files at the output paths listed below before regenerating.

2. CREATE MESH OBJECT
   - GameObject "ProceduralMeshGround" at (0,0,0), localScale (1,1,1).
   - Add MeshFilter, MeshRenderer, MeshCollider.
   - Mesh.indexFormat = UInt32 (vertex count exceeds 65535 at 501x501).

3. VERTEX GRID + UVs
   - Centered grid: startX = -localSizeX * 0.5, startZ = -localSizeZ * 0.5.
   - For each (ix, iz): lx = startX + ix * (localSizeX / (vertsX-1)),
                        lz = startZ + iz * (localSizeZ / (vertsZ-1)).
   - wx = lx, wz = lz (no scale multiplication).
   - Assign UVs: uv[idx] = (ix / (vertsX-1), iz / (vertsZ-1)). Required for downstream layers.

4. BASE HEIGHT FIELD — FBM with domain warping and a gentle continental tilt
   - Domain warp (organic shapes, no axis-aligned banding):
       wxw = wx + (PerlinNoise(wx * 0.005 + 11.1, wz * 0.005 + 23.7) - 0.5) * 80;
       wzw = wz + (PerlinNoise(wx * 0.005 + 41.3, wz * 0.005 + 57.9) - 0.5) * 80;
   - 5-octave FBM with frequencies in cycles-per-meter and amplitudes in meters:
       k = [0.003, 0.008, 0.020, 0.050, 0.120]
       a = [75,    42,    16,    5,     2   ]
     (Use seeded per-octave offsets so different runs produce different worlds.)
     Rationale: a[0] reduced from 110→75 so the broadest continental swing
     does not produce single deep chasms (-100m+) once the lowland flatten is gone.
     a[1]/a[2] slightly raised to keep visible mid-frequency relief.
   - Gentle continental tilt: bias one edge lower so water has somewhere to drain to.
     Keep this small — erosion + Priority-Flood do the real basin work, the tilt is
     only here to break perfect symmetry and give rivers a preferred outflow direction.
       tilt = 12 * ((wz - minZ) / (maxZ - minZ) - 0.5)
       h   = fbm(wxw, wzw) - tilt
   - Do NOT post-flatten lowlands. Earlier specs multiplied negative heights by a small
     constant to "keep basins readable", but this turns most of the map into a
     featureless pancake when combined with the tilt. Erosion (Step 5) and the
     Priority-Flood depression fill (Step 6) already produce natural drainable basins.
   - Store the result as a width*height float[] DEM. This DEM is the working surface
     for all subsequent hydrology steps.

5. HYDRAULIC EROSION (Sebastian Lague droplet simulation)
   - ~100,000 droplets (scale linearly with cell count for non-default sizes).
   - Per-droplet: random start cell, inertia 0.05, sediment capacity factor 4,
     erode speed 0.4, deposit speed 0.3, evaporate 0.01, gravity 4, max lifetime 30,
     erosion brush radius 3 (precomputed weighted disk).
   - Erosion mutates the DEM in place; this carves natural valleys and ridges.

6. PRIORITY-FLOOD DEPRESSION FILL (Barnes 2014)
   - Build a min-heap of all edge cells keyed by elevation.
   - Pop lowest, raise each unvisited neighbour to max(neighbour, popped + epsilon)
     where epsilon = 5e-4. Push neighbour onto heap.
   - Result: a hydrologically corrected DEM with NO interior pits — every cell drains
     downhill to the boundary. This is the precondition for connected rivers.

7. D8 FLOW DIRECTION + ACCUMULATION
   - For each interior cell, choose the steepest of 8 neighbours weighted by
     1/sqrt(2) for diagonals; record the flow direction index.
   - Compute flow accumulation via Kahn's topological sort:
       inDegree[c] = number of upstream neighbours pointing to c
       seed queue with cells whose inDegree == 0
       acc[c] = 1 + sum of acc[upstream]; push downstream neighbour when its
       inDegree drops to 0.
   - Output: int[] accumulation, where each cell value = number of upstream cells.

8. RIVER NETWORK EXTRACTION
   - Threshold: keep cells with accumulation >= 98.5th percentile of all cells
     (i.e. top 1.5%). Mark these as "river-eligible".
   - Find mouths: river-eligible cells on the map boundary, sorted by accumulation.
     Take up to the top 3 mouths whose accumulation >= 5x the threshold — these
     are the dominant outflows.
   - Upstream BFS from each mouth following the reversed D8 graph: walk into any
     upstream river-eligible cell. Tag every visited cell with its mouth's river ID.
     The result is up to 3 connected, edge-to-edge river systems.
   - Reject orphan cells (river-eligible but not reached from any mouth).

9. VARIABLE-WIDTH RIVER CARVING
   - For each river cell c:
       accNorm = sqrt(acc[c] / maxAcc)                            // 0..1
       widthCells = (0.55 + accNorm * 1.05) * 3.5 * jitter        // jitter 0.75..1.25 from Perlin
       depthMeters = 4.0 * (0.55 + accNorm * 0.95)
   - SKIP centers within 2 cells of the map boundary. Carving right at the mouth
     produces thin vertical walls at the mesh edge because the carve disk is only
     partially applied inside the map. Let mouths just exit the map naturally.
   - Carve a soft-falloff disk centred on c into the DEM:
       for each cell within widthCells radius:
         t = distance / widthCells                                // 0..1
         falloff = 1 - smoothstep(0, 1, t)                        // 1 at center → 0 at edge
         dAdd = depthMeters * falloff
         // HARD CAP: no cell may be carved more than 6m below its original elevation.
         // Adjacent centerline cells stack additively, and without this cap a dense
         // cluster near a mouth can carve 30m+ creating a canyon that reads as a
         // mesh artifact. 6m is enough for visible river valleys without spikes.
         newCarve = carveDepth[n] + dAdd
         if (newCarve > 6.0) dAdd = 6.0 - carveDepth[n]
         dem[n] -= dAdd
         carveDepth[n] = min(newCarve, 6.0)
   - 5x5 box blur ONE pass over the river-band NEIGHBOURHOOD: any cell within 2
     cells of a carved cell gets smoothed, not just the carved cells themselves.
     This prevents cliff walls between a fully-carved cell and its uncarved neighbour
     which previously produced visible vertical "leakage" in the mesh.
   - Record per-cell carve depth and width radius for the FlowMap channels.

10. WRITE DEM BACK TO MESH
    - Vertex y = dem[ix, iz].
    - mesh.vertices = ..., mesh.triangles = ..., mesh.uv = ...
    - RecalculateNormals(), RecalculateTangents(), RecalculateBounds().
    - Save the mesh as an asset:
        Assets/BenchmarkRuns/{run-id}/Meshes/GroundMesh.asset
    - Assign to MeshFilter.sharedMesh and MeshCollider.sharedMesh.

11. PLACEHOLDER GROUND MATERIAL (Layer 2 will overwrite its maps)
    - Path: Assets/BenchmarkRuns/{run-id}/Materials/Ground.mat
    - Shader: Universal Render Pipeline/Lit (fallback Standard).
    - _BaseColor = (0.55, 0.55, 0.55, 1), Smoothness = 0.10, Metallic = 0.0.
    - No textures assigned. This is intentional — Layer 2 supplies them.

12. DATA SIDE-CHANNEL: HEIGHTMAP EXR
    - Path: Assets/BenchmarkRuns/{run-id}/Data/Heightmap.exr
    - Encode in C# from a TextureFormat.RGBAFloat Texture2D using EncodeToEXR().
      Do NOT pass EXRFlags.OutputAsFloat — Unity's EXR importer collapses the asset
      to RGBAHalf on disk anyway, and 16-bit float (~0.05m worst case at 75m) is
      well below any meaningful terrain precision. Half is the canonical on-disk
      format for this channel.
    - Channel layout:
        R = world-space height in meters (raw, NOT normalized)
        G = 0, B = 0, A = 1
    - Importer settings (set explicitly via TextureImporter then SaveAndReimport):
        sRGB OFF, filterMode Point, wrapMode Clamp,
        textureCompression None, isReadable TRUE.
      The importer defaults to non-readable for EXR; you MUST flip isReadable on
      or downstream layers cannot call GetPixels().

13. DATA SIDE-CHANNEL: FLOWMAP EXR
    - Path: Assets/BenchmarkRuns/{run-id}/Data/FlowMap.exr
    - Encode the same way (RGBAFloat source Texture2D → EncodeToEXR(), no
      OutputAsFloat flag). Final on-disk format will be RGBAHalf — that is correct.
    - Channel layout (one cell per pixel, exact same dimensions as the vertex grid):
        R = normalized flow accumulation (acc / maxAcc), 0..1
        G = river mask (1.0 inside any carved river band, 0 outside)
        B = carve depth in meters at this cell (0 outside river bands)
        A = centerline width radius in cells (0 outside river bands)
    - Importer settings: sRGB OFF, filterMode Point, wrapMode Clamp,
      textureCompression None, isReadable TRUE.

14. DATA SIDE-CHANNEL: TERRAIN.JSON
    - Path: Assets/BenchmarkRuns/{run-id}/Data/terrain.json
    - Required fields:
        seed                    int    (the actual seed used — for replay)
        localSizeX, localSizeZ  int
        vertsX, vertsZ          int
        minHeight, maxHeight    float  meters
        riverSystemCount        int    1..3
        riverCellCount          int    total cells flagged as river
        widthRadiusMin/Max      float  cells
        carveDepthMin/Max       float  meters
        heightmapPath           string asset path to Heightmap.exr
        flowmapPath             string asset path to FlowMap.exr
        meshPath                string asset path to GroundMesh.asset
        channels                object describing R/G/B/A meaning of each EXR
    - Pretty-printed JSON, UTF-8, no BOM.

15. RIVERPATH GAMEOBJECT (compatibility surface for Layer 3)
    - Create empty GameObject "RiverPath" at (0,0,0), scale (1,1,1).
    - Pick the river system with the largest total cell count as the "primary" river.
    - Walk its centerline from highest mouth to lowest using upstream-then-downstream
      ordering on the D8 graph; sample ~30-40 evenly spaced waypoints along it.
    - For each waypoint, create a child GameObject "Waypoint_{i}" at the waypoint's
      world position (x, terrainHeightAtCell, z). Scale (1,1,1).
    - This is the compatibility surface Layer 3 reads to build the river water mesh.
    - Additional river systems (if any) live only in the FlowMap.exr — Layer 3 may
      read FlowMap.G to discover them.

16. ENVIRONMENT
    - RenderSettings.fog = false.

17. REPORT
    Return a per-layer summary:
      seed used, map size, vertex count, height range,
      droplets simulated, depressions filled count,
      maxAccumulation, riverSystemCount, riverCellCount,
      width radius range (cells), carve depth range (meters),
      mesh path, material path, Heightmap.exr path, FlowMap.exr path, terrain.json path,
      RiverPath waypoint count.
```

---

## Layer 1 Scoring Rubric (100 base points)

### Hydrology Quality (45 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Pit-free DEM** | 8 | Priority-Flood (or equivalent) applied; no interior sinks remain. |
| **Edge-to-edge rivers** | 12 | At least one continuous river crosses the map from interior to a boundary mouth. |
| **River count scales with map** | 5 | 1–3 dominant river systems on a 500x500; more on larger maps. |
| **Variable river width** | 8 | Width grows with sqrt(accumulation); narrowest tributaries clearly thinner than trunk. |
| **Natural valley cross-section** | 6 | Soft-falloff carve, no cliff walls, banks blend into surrounding terrain. |
| **Erosion realism** | 6 | Hydraulic erosion droplets clearly shaped ridges/valleys before river extraction. |

### Technical Quality (35 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Random seed per run** | 6 | Seed sourced from DateTime.Now.Ticks; recorded in terrain.json. |
| **Reproducibility metadata** | 4 | terrain.json contains seed + all parameters needed to replay. |
| **Heightmap.exr** | 5 | EXR (RGBAHalf on disk), R = raw meters, sRGB off, point filter, clamp, uncompressed, isReadable=true. |
| **FlowMap.exr** | 6 | EXR (RGBAHalf on disk) with documented R/G/B/A = accNorm/mask/depth/width channels, isReadable=true. |
| **Mesh + UVs + tangents** | 5 | Correct vertex count, UInt32 index, UVs assigned, tangents recalculated. |
| **Layer isolation** | 5 | Placeholder Ground.mat is plain gray; no PBR/splat/grass/tinting bled in. |
| **Performance** | 4 | Full pipeline (FBM + erosion + Priority-Flood + carve) under 15 seconds for 501x501. |

### Visual Quality (20 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Landscape readability** | 8 | Reads as a coherent place: mountains drain into valleys, valleys host rivers. |
| **River believability** | 8 | Rivers visibly meander, widen downstream, exit the map at logical low points. |
| **No artifacts** | 4 | No grid banding, no axis-aligned stair-stepping, no orphan pits or floating water. |

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
