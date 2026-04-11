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
       a = [110,   38,    14,    5,     2   ]
     (Use seeded per-octave offsets so different runs produce different worlds.)
   - Continental tilt: bias the south edge lower so water has somewhere to drain to.
       tilt = 35 * ((wz - minZ) / (maxZ - minZ) - 0.5)   // negative on the south half
       h   = fbm(wxw, wzw) - tilt
   - Lowland flatten (keeps depressions readable as basins, not abyss):
       if (h < 0) h *= 0.22;
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
   - Carve a soft-falloff disk centred on c into the DEM:
       for each cell within widthCells radius:
         t = distance / widthCells                                // 0..1
         falloff = 1 - smoothstep(0, 1, t)                        // 1 at center → 0 at edge
         dem[n] -= depthMeters * falloff                          // additive over overlaps
   - 3x3 box blur ONE pass over river-band cells only, to soften the carve.
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

# Layer 2 — Multi-Layer Context-Aware Terrain Bake

## Prerequisites

- `ProceduralMeshGround` must exist with valid mesh, normals, and MeshCollider.
- `RiverPath` with waypoint children must exist (OR `FlowMap.exr` for richer river data — see Section 5).
- `Ground.mat` (URP Lit placeholder) must exist from Layer 1.
- Layer 1 must be completed.

## Approach

Layer 2 **abandons the classic "4-channel splat map + 4 tileable texture sets" approach**. Instead it uses **direct per-pixel procedural baking with N context rules and heightlerp blending** — the technique from Ghost Recon Wildlands / Far Cry 5 (GDC 2018) and Ryan Brucks's widely-cited Heightlerp post.

**Why:** With baked output, there's no reason to be constrained by 4 RGBA channels. Evaluate 10+ layers per texel, keep the top 2, blend with height-aware transitions. This produces river banks that look different from hilltops, south slopes different from north slopes, and valleys different from ridges — all without custom shaders, all as baked textures assigned to a single URP Lit material.

An earlier version of this spec tried the classic 4-biome splat + tileable approach. It looked like a checkerboard of garish noise patterns because tileable textures at realistic tile counts (~40 repeats) expose their repetition brutally. Don't go back to that approach.

## Context data computed at bake time

For every output texel, sample (from the mesh and precomputed grids):

- **height** (m) — bilinear from mesh vertex Y
- **slope** — `1 - max(0, normal.y)`, bilinear from mesh normals
- **curvature** — `h - avg(h ± 1 neighbor)` on the vertex grid, bilinear. Positive = ridge, negative = valley.
- **aspect** — `-dot(normalXZ, sunDirXZ)` where both are normalized XZ vectors. Positive = sun-facing slope (south-ish for a northern-hemisphere sun), negative = shaded (north-ish).
- **distToRiver** — distance (m) to nearest river cell. See "River distance and width" below.
- **widthAtNearestRiver** — the width radius (in cells) of whichever river cell is closest. Allows zones to scale with river size. See "River distance and width" below.

## River distance and width (critical)

Use the **`FlowMap.exr` G-channel river mask**, NOT the `RiverPath` polyline. `RiverPath` only contains the primary river's ~36 waypoints — secondary rivers are invisible to it, and even the primary is coarsely sampled. The FlowMap G channel is `1` inside every carved river cell of every river system.

Compute two grids via a **Chamfer distance feature transform** (two-pass, O(N)):

- `distGrid[i]` = Euclidean-approximate meters from cell `i` to the nearest river-mask cell
- `widthGrid[i]` = the `widthRadius` (FlowMap A channel) of that nearest cell, propagated along with the distance

Initialisation: river cells (G > 0.5) set `dist=0` and `width = max(A, 3)` (clamp to minimum 3 cells ≈ 6m because EXR interpolation sometimes writes A=0 at river-mask pixels). Non-river cells set `dist=BIG, width=0`.

Forward pass (top-left→bottom-right) and backward pass (bottom-right→top-left): at each cell, if any neighbor offers a shorter distance, copy BOTH its distance+edge-cost AND its `width` value. This propagates the nearest-cell's width to the whole grid.

Sample per output texel via bilinear on `distGrid`, nearest-neighbor on `widthGrid` (preserve discontinuities at watershed boundaries).

## Critical smoothstep note

All rule formulas use **HLSL-style `smoothstep(edge0, edge1, x)`**: `0 when x<=edge0, 1 when x>=edge1, cubic 3t²−2t³ in between`. Do **NOT** use Unity's `Mathf.SmoothStep(from, to, t)` — that function returns a smoothly-interpolated value between `from` and `to` (not a [0,1] edge mask) and will produce garbage weights. Write your own:

```
float ss(float e0, float e1, float x) {
    float t = saturate((x - e0) / (e1 - e0));
    return t * t * (3f - 2f * t);
}
```

## Layers (10)

Each has: base RGB, noise seed + frequency, height-noise seed + frequency, heightlerp bias, and PBR mask values.

| # | Name | Weight rule | Base color |
|---|------|-------------|-----------|
| 0 | grass-lush | `(1 - ss(0.1, 0.35, slope)) * ss(30, -5, h) * (0.5 + 0.5 * ss(0.2, -0.5, curv))` | dark damp green |
| 1 | grass-dry | `(1 - ss(0.1, 0.4, slope)) * ss(-5, 25, h) * (1 - ss(35, 55, h))` | olive |
| 2 | grass-bleached | `(1 - ss(0.05, 0.4, slope)) * ss(-0.1, 0.5, aspect) * ss(-5, 20, h) * (1 - ss(38, 55, h))` | pale yellow-green |
| 3 | rock-exposed | `ss(0.22, 0.4, slope) + ss(35, 55, h) * ss(-0.2, 0.5, curv) * 0.9` (clamp 1.2) | gray |
| 4 | rock-mossy | `rockyness * ss(0.4, -0.4, aspect) * ss(0.3, -0.3, curv) * ss(55, 15, h)` | mossy green-gray |
| 5 | mud-wet | `ss(mudOuter, mudInner, distR) * (1 - ss(0.1, 0.35, slope)) * 1.6` | dark brown |
| 6 | dirt-dry | `steepLowAlt` only (do NOT add a river-bank fringe here) | earth brown |
| 7 | scree | `ss(0.28, 0.45, slope) * ss(15, 45, h) * (1 - ss(0.55, 0.75, slope))` | gravel |
| 8 | snow-packed | `ss(50, 62, h) * (1 - ss(0.12, 0.32, slope))` | white |
| 9 | snow-blown | `ss(55, 68, h) * ss(0.1, 0.28, slope) * 0.6` | pale gray-white |

- `steepLowAlt = ss(0.4, 0.65, slope) * ss(25, 0, h)`
- `rockyness = ss(0.15, 0.35, slope) + ss(30, 50, h) * 0.4`

### Width-scaled mud zone

The mud-wet zone scales with the size of the nearest river:

```
widthMeters = widthAtNearestRiver * meshCellMeters   // meshCellMeters ≈ 2m typically
mudFar   = max(2.0, widthMeters * 0.5)               // center-out radius
mudInner = mudFar * 0.6                              // full-opacity radius
mudOuter = mudFar * 1.4                              // fade-to-zero radius
```

The wide `mudInner..mudOuter` falloff range makes edges visibly soften into the surrounding layer (usually grass-lush). Big rivers get ~6–10m of mud; small streams get ~2–3m. This removes the "painted-on" look of a hard-cutoff mud band.

**Do not add a dirt-dry bank fringe around mud.** It creates a double-ring (dark mud + lighter brown) that reads as fake. Let mud fade straight into grass via heightlerp.

Per-layer noise params (suggested): `noiseFreq ≈ 0.16–0.38` cycles/m, `heightFreq ≈ 0.28–0.65`, `variation ≈ 0.04–0.16` (low for snow, higher for rock). Use independent seed offsets per layer and per field (albedo noise vs height noise).

## Bake algorithm (per output texel)

```
1. Sample context: height, slope, curv, aspect, distR, meshNormal (bilinear from grids)
2. Evaluate all 10 layer weights; keep top-2 (id0, W0) and (id1, W1)
3. If W0 < 1e-3 default to grass-dry (id=1) to avoid empty pixels
4. For top 2 layers, procedural color + height from world-space FBM:
     albedo_i = baseRGB_i * (1 + (fbm2(worldXY, noiseSeed_i, freq_i) - 0.5) * 2 * variation_i)
     layerHeight_i = fbm2(worldXY, heightSeed_i, heightFreq_i) + heightBias_i
5. Heightlerp (Brucks):
     hA = layerHeight_0 + W0 * weightScale      // weightScale ≈ 5
     hB = layerHeight_1 + W1 * weightScale
     hMax = max(hA, hB); k = 0.40               // blend width (soft transitions)
     mA = max(hA - hMax + k, 0); mB = max(hB - hMax + k, 0)
     albedo = (albedo_0 * mA + albedo_1 * mB) / (mA + mB)
   // Note: k was 0.25 in an earlier tuning. 0.40 gives much softer transitions
   // and is required when mud has a wide falloff range (otherwise mud edges
   // snap sharply even though their weight is fading).
6. Macro overlay (kills the "uniform wallpaper" look, adds meso-scale variation):
     macro = 0.5 * Perlin(worldXY / 30m) + 0.5 * Perlin(worldXY / 80m)
     albedo *= lerp(0.85, 1.15, macro)
7. Sun exposure tint (uses mesh normal, not per-layer normal):
     sunExp = saturate(dot(meshNormal, -sunDir))
     albedo *= lerp(0.85, 1.15, ss(0.2, 0.8, sunExp))
8. Normal map: gradient of the macro overlay (2 extra Perlin taps). Keep subtle.
9. Mask map: from top layer's height noise
     metallic = layerMetallic[id0]
     AO = lerp(layerAOmin[id0], layerAOmax[id0], layerHeightNoise_0)
     smoothness = lerp(layerSMmax[id0], layerSMmin[id0], layerHeightNoise_0)
```

`fbm2(worldXY, seed, freq)` is 2-octave FBM at world-space coordinates. Sampling world-space (not UV-space) means the bake doesn't need tileable textures at all — noise naturally fills the map. No tiling artifacts.

## Output files

- `Assets/BenchmarkRuns/{run-id}/Textures/GroundHeightTint.png` (1024², sRGB, albedo composite)
- `Assets/BenchmarkRuns/{run-id}/Textures/GroundNormalMap.png` (1024², linear, normal map)
- `Assets/BenchmarkRuns/{run-id}/Textures/GroundMaskMap.png` (1024², linear, R=metallic, G=AO, A=smoothness)

**No intermediate tileable texture files** are written. Earlier per-biome `Splat{Grass|Rock|Dirt|Snow}{Albedo|Normal|Mask}.png` files from the old spec should be deleted if present.

## Material assignment

On `Ground.mat` (URP Lit):
- `_BaseMap` = GroundHeightTint.png, `_BaseColor` = white
- `_BumpMap` = GroundNormalMap.png, `_BumpScale` = 1.0, keyword `_NORMALMAP` enabled
- `_MetallicGlossMap` / `_MaskMap` / `_OcclusionMap` = GroundMaskMap.png
- Keywords `_METALLICSPECGLOSSMAP`, `_MASKMAP`, `_OCCLUSIONMAP` enabled

## Performance & budget

- 10 layer rule evaluations + top-2 selection per texel (~50 inlined smoothsteps).
- 2 × 2-octave FBM for albedo + 2 × 2-octave for heightlerp + 2-tap macro + 2-tap normal gradient ≈ 14 Perlin calls per texel.
- 1024² output ≈ 14M Perlin calls → ~15–25s in CodeDom C#.
- **2048² output will likely exceed the MCP execute_code timeout**. If you need higher resolution, split the bake across multiple execute_code calls (top half / bottom half) or precompute an intermediate EXR for layer IDs and do a second pass for albedo.
- Inline the smoothstep, distance, and bilinear helpers — closure invocation overhead is measurable in CodeDom.

## Scoring rubric (100 points)

| Category | Points |
|----------|--------|
| Layer count and diversity (≥6 layers actively winning different regions) | 20 |
| Context awareness (river corridors, aspect, curvature visibly affect output) | 20 |
| River realism (width-aware mud zones, smooth fade-out, no fake bank fringe) | 15 |
| Heightlerp quality (interlocking transitions, soft k≈0.4 blends, not linear fades) | 15 |
| Macro overlay + sun exposure (no wallpaper look, visible light direction) | 15 |
| Technical correctness (HLSL smoothstep, URP Lit keywords, importer settings, FlowMap feature transform, world-space noise) | 15 |

---

---

# Layer 3 — Mask-Based River Water System

## Critical Rule

**Terrain is READ-ONLY.** Water surfaces sit on the terrain — they do not carve, reshape, or recolor it. Do not touch `ProceduralMeshGround` vertices, normals, UVs, collider, or any Layer 2 textures.

## Prerequisites

- `ProceduralMeshGround` must exist with valid mesh and MeshCollider.
- `FlowMap.exr` from Layer 1 must be readable and contain: G = river mask, B = carve depth (meters), A = width radius (cells).
- `Ground.mat` and Layer 2 composites must be finalised (water is the last terrain-surface layer).

## Approach

Walking the `RiverPath` polyline and generating a quad-strip **does not work** — `RiverPath` contains only the primary river's ~36 waypoints, the strip is a straight-line approximation that clips into the actual meandering carved channel, and tributaries have no water. An earlier version of this spec did exactly that; do not return to it.

Instead, **build the water mesh directly from the `FlowMap.exr` river mask**. One quad per masked cell, dedupe shared vertices. The mesh topology exactly matches the carved channel topology, including every branch.

## Bake algorithm

```
1. Load FlowMap.exr
   - mask[c] = (fPix[c].g > 0.5)
   - carveDepth[c] = fPix[c].b (meters)

2. For each masked cell, compute "original pre-carve surface":
     origSurf[c] = carvedTerrainY(cellCenter) + carveDepth[c]
   carvedTerrainY is bilinear-sampled from the current ground mesh.

3. Water level H[c] = (max of origSurf over cells within radius 10, masked only) - sinkBelowBank
   Using the MAX over a neighborhood gives the bank-top envelope: water rises to fill the channel
   almost to the top of the bank. Radius 10 cells is enough for typical river widths.
   sinkBelowBank ≈ 1.0m controls how much bank is visible above water — tune by shifting the
   final vertex Ys uniformly.

4. 5 passes of 3x3 box blur on H, masked cells only. Flattens cross-section and longitudinal
   ripples so water reads as a fluid surface, not a bumpy copy of the carved bed.

5. Clamp: H[c] >= carvedTerrainY(cellCenter) + 0.1m (ensures at least 10cm of water depth).

6. Build mesh with vertex dedup:
   - For each masked cell emit 2 triangles on the 4 corner vertices
   - Dedup via Dictionary<int, int> keyed on (vz * (DW+1) + vx)
   - Vertex Y is the average of H[c] over the up-to-4 adjacent masked cells
   - Edge vertex inset: if not all 4 adjacent cells are masked, shift the vertex toward the centroid
     of the masked adjacent cells by ~0.15 cells. This tucks the water lip under the bank and softens
     the pixelated step outline.
   - mesh.indexFormat = UInt32 (5000+ cells can exceed 65k verts after dedupe edge cases)

7. Save mesh as Assets/BenchmarkRuns/{run-id}/Meshes/RiverWaterMesh.asset
```

### Tuning knobs

| Knob | Default | Effect |
|------|---------|--------|
| `bankRadius` (cells) | 10 | Larger = pulls water up to higher surrounding ground (wider search for bank top) |
| `sinkBelowBank` (m) | 1.0 | Larger = more bank visible above water, water sits deeper in channel |
| Min water depth (m) | 0.1 | Safety — prevents water from clipping carved bed |
| Blur passes | 5 | More = flatter water, fewer = water follows carve bumps |
| Edge inset (cells) | 0.15 | Larger = water edge pulls further from bank, softens sawtooth more |

## Water material

Path: `Assets/BenchmarkRuns/{run-id}/Materials/Water.mat`

- Shader: `Universal Render Pipeline/Lit` (fallback Standard)
- Surface type: **Transparent** (`_Surface = 1`)
- Blend mode: **Alpha** (`_Blend = 0`)
- Base Color: `(0.15, 0.35, 0.45, 0.30)` — **desaturated teal, not pure blue**. Pure blue reads as plastic.
- Alpha: 0.30–0.45 depending on desired transparency
- Smoothness: **0.95**. Below 0.9 reads as wet paint.
- Metallic: **0.0**. Water is dielectric; any metallic kills Fresnel.
- `_SrcBlend = SrcAlpha`, `_DstBlend = OneMinusSrcAlpha`, `_ZWrite = 0`
- `renderQueue = 3000` (Transparent)
- Keywords: `_SURFACE_TYPE_TRANSPARENT` enabled; `_ALPHATEST_ON`, `_ALPHABLEND_ON`, `_ALPHAPREMULTIPLY_ON`, `_SPECULAR_SETUP` disabled
- `_SpecularHighlights = 1`, `_EnvironmentReflections = 1` — this is where the "water look" comes from without a custom shader (sky reflection via reflection probe)

**Do not enable `_SPECULAR_SETUP`** — it switches URP Lit to the specular workflow and breaks metallic/smoothness reflections.

## Hierarchy

```
Water/
  RiverWater (MeshFilter + MeshRenderer, shadows off, receive shadows off)
  WaterExclusionZones/
    River_Exclusion_margin5_halfwidth{int}
```

Exclusion zones are consumed by Layer 4 to keep props away from water. Current spec only records the river; lakes/ponds are noted as optional for future expansion.

## Environment

`RenderSettings.fog = false`.

## Common pitfalls

- **Water pixelated on edges** — use the edge-vertex inset trick. Larger inset (0.3–0.4 cells) softens more but if too large the water visibly pulls away from the bank.
- **Water follows the bed (wavy surface)** — smoothing isn't aggressive enough OR you're using a min-pull instead of the max-over-radius envelope. Switch to max-of-origSurf.
- **Water buried below terrain** — you used `terrainY + lift` instead of the `origSurf - sinkBelowBank` formula. The carved bed is low; water sitting just above the bed is mostly hidden behind the banks.
- **Water spilling over banks** — `sinkBelowBank` too small OR `bankRadius` too large (reaching further uphill to higher "bank" candidates). Try reducing bankRadius to 8.
- **Looks like plastic, not water** — alpha too high (≥0.8) OR smoothness too low (<0.9) OR metallic > 0 OR pure-blue base color OR `_SPECULAR_SETUP` accidentally enabled.
- **Catmull-Rom polyline approach** — do not. See "Approach" section.

## Scoring rubric (100 points)

| Category | Points |
|----------|--------|
| River coverage (all river systems including tributaries, not just primary) | 25 |
| Water surface flatness (flat cross-section, no bed ripples) | 20 |
| Channel fill level (water near bank top, banks hide edges, no mud bank strip visible between water and terrain) | 20 |
| Material realism (transparent, reflective, desaturated teal, specular highlights) | 15 |
| Technical correctness (terrain untouched, UInt32 index, importer settings, URP Lit transparent flags, exclusion hierarchy) | 20 |

---

---

# Layer 4 — Pine Forest (Poisson-Disk Clumping with Forest Mask)

## Critical Rule

**Do NOT modify terrain mesh, terrain textures, water surfaces, or any previous layer output.** All props placed ON TOP via position sampling from the terrain mesh.

## Prerequisites

- `ProceduralMeshGround` with valid mesh and normals.
- `FlowMap.exr` from Layer 1 (for water exclusion via river mask distance transform).
- Layer 2 composites and Layer 3 water complete.

## Scope

Layer 4 generates **pine trees only**. Earlier versions of this spec included oaks, bushes, rocks and understory, but practical testing revealed:

- **Bushes** generated from small stems + overlapping ellipsoids look terrible up-close (dark blobs with visible trunk stubs). They either need dedicated asset-quality meshes or they should be skipped.
- **Oaks** with fat-trunk + canopy-ellipsoid look OK but visually compete with pines rather than adding realism. A single-species forest reads as more consistent.
- **Rocks and understory** are deferred as future work — the current stack (terrain + water + pine forest) is self-contained and visually coherent.

If you want multiple species, plan separate material+mesh asset pairs per species and keep species distinct in the hierarchy for culling/LOD.

## Approach

**Poisson-disk sampling with a sharply-contrasted forest mask.** Uniform jittered grids produce a visible regular pattern; standard Poisson-disk alone gives even distribution. What makes the forest read as real is **clumping**: dense cores, sparse edges, bare clearings.

### Forest mask

A 3-octave FBM sharpened via smoothstep gives clear forest/clearing boundaries:

```
n1 = PerlinNoise(wx * 0.005, wz * 0.005)        // 200m patches
n2 = PerlinNoise(wx * 0.012, wz * 0.012) * 0.5  // 80m variation
n3 = PerlinNoise(wx * 0.03,  wz * 0.03)  * 0.25 // 30m fine noise
raw = (n1 + n2 + n3) / (1 + 0.5 + 0.25)
forestMask = smoothstep(0.35, 0.60, raw)        // 0 = clearing, 1 = dense forest
```

The smoothstep remap is what creates **hard** forest boundaries instead of a gentle density gradient. Tuning `(0.35, 0.60)` controls how much of the map is forest vs clearing. Wider gap = softer edges, tighter = more abrupt.

### Variable Poisson radius

```
r(wx, wz) = lerp(10m, 3m, forestMask(wx, wz))
```

Dense cores get 3m spacing (very tight), edges and transition zones get 10m. Points outside the forest mask (`forestMask < 0.05`) are rejected entirely.

### Bridson's algorithm with multi-seeding

Standard Bridson's starts from one seed and expands. With a disconnected forest mask (multiple islands), a single seed only fills the island it started in. Fix: **seed 30 random valid points spaced ≥ 8m apart** before running the main loop. This guarantees all islands get populated.

Main loop: k = 30 candidate tries per active point, annulus `[r, 2r]`. Neighbor check over a 7×7 grid window (3 cells around to catch neighbors within max possible radius). Accept if no existing point is within the **max of the two radii** (variable-r Poisson rule).

### Placement filter

Reject candidates where any of:
- `height < -2m` (underwater/mud zones)
- `height > 55m` (snow line)
- `slope > 0.45` (too steep)
- `distToRiver < 4m` (water exclusion)
- `forestMask < 0.05` (clearing)

## Per-tree attributes

- **Power-law scale**: `scale = 0.75 + pow(rand01, 2.5) * 0.9` → range 0.75–1.65, distribution biased toward small (many saplings, few giants)
- **Orientation slerp**: `upTree = slerp(worldUp, terrainNormal, 0.2)` → 20% tilt toward terrain slope, keeps trees visibly vertical but with subtle lean
- **Random yaw** `[0, 360°]`
- **Sink** `-0.2m to -0.35m` random so trunk bases don't float on raycast imprecision

## Pine mesh

Two sub-meshes (one per material), reused across all 5000+ instances via GPU instancing:

### Trunk (`PineBark.asset`)

- Single 8-radial tapered cylinder
- Base radius 0.22m, top radius 0.08m, height 6m
- Bottom + top fan caps

### Foliage (`PineLeaves.asset`)

Three stacked cones for a layered pine silhouette:

| Cone | Y start | Base radius | Top radius | Height |
|------|---------|-------------|------------|--------|
| 1 (bottom) | 2.4m | 1.6m | 0.3m | 1.8m |
| 2 (middle) | 3.8m | 1.25m | 0.2m | 1.8m |
| 3 (top)    | 5.2m | 0.85m | 0.0m | 1.8m |

Each cone uses `addCyl` with 8 radial segments. Cones share no vertices (cleaner shading via `RecalculateNormals`).

## Materials

Two URP/Lit materials, **`enableInstancing = true`** on both (mandatory for rendering 5k+ trees):

| Material | Base color | Smoothness | Metallic |
|----------|-----------|------------|----------|
| `TreeBarkPine.mat` | (0.22, 0.14, 0.08) | 0.08 | 0 |
| `TreeLeafPine.mat` | (0.08, 0.20, 0.08) | 0.12 | 0 |

## Hierarchy

```
Props/
  Trees/
    Tree_N (transform: pos + rot + scale)
      Trunk  (MeshFilter=PineBark,   MeshRenderer=TreeBarkPine)
      Leaves (MeshFilter=PineLeaves, MeshRenderer=TreeLeafPine)
```

5000+ Tree_N GameObjects, all sharing 2 meshes and 2 materials. URP batches them via GPU instancing — the renderer only issues ~2 draw calls per frame for the entire forest.

## Expected output

For a 500×500m map with seed 98765:
- ~5000 pine trees (exact count varies with FBM realisation)
- Grid 237×237 cells, ~10k Bridson iterations
- Forest coverage ~25–35% of map area
- Dense cores at 3m spacing, sparse edges at 10m
- Multiple disconnected forest islands all populated

## Common pitfalls

- **Uniform grid-with-jitter instead of Poisson-disk** → visible lattice in the canopy.
- **No forest mask** → uniform density looks like a tree farm.
- **Soft forest mask (no smoothstep sharpening)** → no clear forest boundaries, no clearings.
- **Single Bridson seed** → only one island gets populated if the mask has multiple.
- **Uniform scale ±20%** → canopy looks flat and unreal. Use power-law distribution.
- **Zero orientation slerp (all trees vertical)** → trees on slopes look like they're on stilts.
- **Full slerp (all trees perpendicular to slope)** → trees on steep hillsides visibly tilt sideways, unreal.
- **`enableInstancing = false`** → 5k trees = 10k draw calls = frame rate collapse.
- **Tiny bush/small prop meshes without real assets** → low-poly blobs look terrible up close. Don't ship them without real authored meshes.

## Scoring rubric (100 points)

| Category | Points |
|----------|--------|
| Forest clumping (clear dense cores + bare clearings, not uniform density) | 25 |
| Placement correctness (no trees on water/peaks/steep slopes/snow) | 20 |
| Orientation + scale realism (20% slerp, power-law scale, sink) | 15 |
| Mesh quality (cone-stacked pine silhouette reads as pine, not abstract shape) | 15 |
| Technical correctness (GPU instancing, shared materials, no scene file bloat, hierarchy) | 15 |
| Layer isolation (terrain/water/textures untouched) | 10 |

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

## Overall Scoring Guide

| Layer | Max Score |
|-------|-----------|
| Layer 1 — Terrain | 100 (× map size multiplier: 0.5x–1.3x) |
| Layer 2 — Splat Map | 100 |
| Layer 3 — Water | 100 |
| Layer 4 — Props | 100 |
| Layer 5 — Lighting | 100 |
| Layer 6 — Sky & Clouds | 100 |
| **Total** | **600** |

| Score | Meaning |
|-------|---------|
| 0–2 per layer | Broken scene, compile errors, floating objects, wrong scale |
| 3–5 per layer | Layer incomplete or significant errors |
| 6–8 per layer | Layer complete with minor issues |
| 9–10 per layer | All criteria met, verified, reproducible |

---

*Benchmark by Darko Tomic. Community: [darkounity.com](https://darkounity.com)*

---

---

## CodeDom / MCP execute_code — Known Failure Patterns

When running this benchmark via Unity MCP (`execute_code`), the compiler backend is **CodeDom (C# 6)**. Several patterns cause silent or noisy failures that invalidate a run. This section documents every failure observed in practice so future agents avoid repeating them.

---

### 1. `using` directives inside the method body

**What fails:**
```csharp
using UnityEditor;          // ❌ — not allowed here
using UnityEngine.Rendering;
```

**Why:** `execute_code` compiles your code as a *method body*, not a full class file. `using` directives must appear at the top of a file, outside a method. Inside a method body they are a syntax error.

**Fix:** Use fully-qualified names everywhere.
```csharp
UnityEditor.AssetDatabase.Refresh();                        // ✅
UnityEditor.SceneManagement.EditorSceneManager.SaveScene(…);
UnityEngine.Object.DestroyImmediate(go);
```

---

### 2. Variable name collision in the same scope

**What fails:**
```csharp
float[] bw = brushWeights.ToArray();   // declared earlier
…
using (var bw = new System.IO.BinaryWriter(…)) { … }  // ❌ — same name
```

**Why:** CodeDom compiles all local variables into a single flat scope. A name used earlier in the method cannot be re-declared, even in a nested block like `using`.

**Fix:** Give each variable a distinct name for its entire existence in the method (`bWgt`, `binW`, etc.).

---

### 3. Creating thousands of GameObjects in one `execute_code` call

**What fails:** Generating 5 000+ trees (each with 2 child GameObjects) inside a single call — Unity disconnects and sometimes crashes, returning `{"success":false,"message":null,"data":null}`.

**Why:** The MCP execute_code call has a hard timeout. Allocating ~15 000 GameObjects plus running a Perlin-heavy Bridson loop in the same call exceeds it.

**Fix:** Split into two calls:
1. **Part 1** — compute positions (algorithm only), write results to a binary file under the run folder.
2. **Part 2** — read the binary, create GameObjects. If still too many, batch across further calls.

---

### 4. Jagged-array shorthand `{{…}, {…}}` not allowed

**What fails:**
```csharp
float[][] data = new float[][] { {1f, 2f}, {3f, 4f} };  // ❌
int[][]   ranges = new int[][] { {8, 20}, {6, 14} };     // ❌
```

**Why:** CodeDom C# 6 requires each inner array to have an explicit `new T[]` constructor when initialised inline inside a jagged-array expression.

**Fix:** Declare each inner array as a named variable first:
```csharp
float[] row0 = new float[] { 1f, 2f };
float[] row1 = new float[] { 3f, 4f };
float[][] data = new float[][] { row0, row1 };  // ✅
```

---

### 5. `AssetDatabase.DeleteAsset` blocked by default safety checks

**What fails:** Any call to `AssetDatabase.DeleteAsset` returns a blocked-pattern error when `safety_checks` is at its default value of `true`.

**Fix:** Pass `safety_checks: false` on any `execute_code` call that needs to delete or overwrite assets. Always do this for calls that write textures, meshes, or materials (delete-then-recreate is the standard fresh-generation pattern required by the benchmark).

---

### 6. Wrong Unity API assumptions

Three specific API mistakes caused compilation errors:

| Mistake | Correct form |
|---------|-------------|
| `Object.FindObjectsOfType<T>()` — `Object` is ambiguous between `System.Object` (`object`) and `UnityEngine.Object` | `UnityEngine.Object.FindObjectsOfType<T>()` |
| `sunLight.shadowDistance = 300f` — `shadowDistance` is not a property of `UnityEngine.Light` | `QualitySettings.shadowDistance = 300f` |
| `profile.Add<ScreenSpaceAmbientOcclusion>()` — type cannot be resolved as `VolumeComponent` in CodeDom context | Wrap in `try { } catch (Exception) { }` or omit SSAO entirely; it is optional in the Layer 5 spec |

---

### Summary checklist for every `execute_code` call

- [ ] No `using` directives — use fully-qualified names
- [ ] No duplicate variable names anywhere in the method, even in nested blocks
- [ ] No single call that both runs a heavy algorithm **and** creates thousands of GameObjects — split the calls
- [ ] Jagged arrays always use `new T[] { … }` for each inner row
- [ ] Add `safety_checks: false` whenever the call uses `AssetDatabase.DeleteAsset` or `File.Delete`
- [ ] Always write `UnityEngine.Object.DestroyImmediate`, not `Object.DestroyImmediate`
- [ ] Shadow distance → `QualitySettings.shadowDistance`, not `Light.shadowDistance`
