# Prompt: Generate Realistic Procedural Terrain (Mountains, Hills, Plains, River Valley)

Use this prompt with a Unity-capable coding agent (Unity MCP) to generate a realistic landscape with distinct geographic features using a procedural mesh. This is **not** random noise bumps — the terrain must read as a real place with mountains, rolling hills, flat plains, and a carved river valley.

---

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

## Copy/Paste Prompt

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
     or plains zone. Rivers flow AROUND mountains, not through them. A river cutting through
     a tall mountain peak looks artificial.
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
    - Generate `Assets/Textures/GroundHeightTint.png` (1024x1024, sRGB).
    - Compute per-pixel: height (h), slope (1 - dot(normal, up)), distance to river.
    - Normalized height: hN = (h - minH) / (maxH - minH).

    Color rules (these thresholds produce visually distinct zones):
    - Rock: slope > 0.12 OR hN > 0.35 (aggressive — mountains MUST look rocky, not green)
      Steeper slopes = darker rock. Color: (0.45, 0.40, 0.33) to (0.30, 0.26, 0.22).
    - Snow: hN > 0.75 AND slope < 0.25. Blend to (0.92, 0.93, 0.96).
    - River bed: distToRiver < 18 → (0.25, 0.18, 0.12)
    - River bank: distToRiver 18-40 → blend dirt/rock to terrain color with quadratic ease (softer transition).
      In mountain zones (hN > 0.4): use rock color, not dirt.
    - Grass/Plains: remaining areas → (0.22, 0.50, 0.12) to (0.28, 0.55, 0.16).

    Apply sun exposure: color *= lerp(0.82, 1.18, dot(normal, lightDir)^2).
    Assign to Ground.mat _BaseMap.

11. BAKE PBR MAPS
    - `GroundHeightMap.png` (1024x1024, linear grayscale)
    - `GroundNormalMap.png` (2048x2048, normal map type)
      Finite-difference bake, assign _BumpMap, _BumpScale = 0.8.

12. UV + MATERIAL SETUP (critical — without this, textures show as solid color)
    - UVs MUST be assigned to mesh (step 3) before textures will display.
    - All material texture tiling MUST be (1, 1). No tiling/repeat.
    - Ground.mat: _BaseMap = GroundHeightTint, _BumpMap = GroundNormalMap.
    - _BumpScale = 0.8, Smoothness = 0.06, Metallic = 0.0, _BaseColor = white.
    - RecalculateTangents() MUST be called for normal maps to function.

13. ENVIRONMENT
    - Fog MUST be disabled (RenderSettings.fog = false).
      Fog is a separate benchmark layer — it must not obscure terrain evaluation.

14. SAVE AND REPORT
    - Return: map size, height range, zone coverage %, river count and lengths,
      texture paths, material assignments.
```

---

## Expected Result

- Terrain reads as a **real landscape**, not random noise bumps.
- **Mountains** are clearly identifiable with **rocky gray-brown coloring** on slopes and peaks.
- **Hills** roll gently between mountains and plains, green with muted alpine tones at higher elevations.
- **Plains** are visibly flat, bright green, open areas at low elevation.
- **River valley** meanders across the terrain and **flows off map edges**. Smooth banks, not cliffs.
- Rock coloring is **aggressive** — steep slopes and high elevations must NOT be green.
- River banks in mountain areas show rock, not dirt.
- On a larger map, there are proportionally MORE features, not the same features stretched.

---

## Map Size Scoring

Larger maps test whether the LLM can scale terrain generation without stretching or performance collapse.

| Map size | Base multiplier | What "good" looks like |
|----------|----------------|----------------------|
| 10x10 | 0.5x | Tiny terrain. At least one hill and valley visible. Limited scope. |
| 100x100 | 0.8x | Small terrain. Mountains, plains, and a river section visible. |
| 500x500 | 1.0x | Standard benchmark. Full landscape with all feature types. |
| 1000x1000 | 1.3x | Large terrain. Multiple mountain ranges, river systems. No stretching. |

The final score = rubric score * map size multiplier. A perfect 500x500 scores 100. A perfect 1000x1000 scores 130.

---

## Scoring Rubric (100 base points)

### Structural Quality (50 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Zone differentiation** | 10 | Mountains, hills, and plains are visually distinct. A human can identify each. |
| **Mountain quality** | 8 | Rocky coloring on slopes/peaks. Multiple peaks. Domain warping for organic shapes. |
| **Hill quality** | 7 | Rolling terrain 10-30m relief. Smooth transitions. Green coloring. |
| **Plains quality** | 5 | Flat areas, max 4m variation. Bright green. Readable as open ground. |
| **River valley** | 10 | Meanders off map edges. Flows AROUND mountains, not through them. Smooth banks. Flows downhill. Brown/rock coloring in bed/banks. |
| **Zone transitions** | 5 | No hard edges. Foothills as gradual transition. smoothstep blending. |
| **Domain warping** | 5 | Organic shapes. No grid artifacts or circular blobs. |

### Technical Quality (30 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Mesh + UVs** | 5 | Correct vertex count. UVs assigned (0-1 mapping). UInt32 format. Tangents calculated. |
| **World-space noise** | 5 | Noise uses wx/wz directly. Scale=(1,1,1). Larger maps = more features, not stretching. |
| **Terrain tint** | 5 | Rock on slopes/peaks (not green). Snow on tops. Green plains. Brown river. All smooth. |
| **Normal map** | 5 | Baked, assigned to _BumpMap. Tiling=(1,1). Tangents present. Adds visible detail. |
| **Height map** | 5 | Baked grayscale heightmap. Linear. Correct range. |
| **Material setup** | 2 | All tiling=(1,1). _BaseColor=white. Smoothness/Metallic correct. Fog off. |
| **Performance** | 3 | Generation under 15 seconds. No editor freeze. |

### Visual Quality (20 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Landscape readability** | 8 | From any camera angle, terrain looks like a real place. Clear geographic features. |
| **Color coherence** | 8 | Natural colors. Rock reads as rock. No green mountain slopes. Snow on peaks. |
| **Sun influence** | 4 | Directional tint baked in. Sun-facing slopes brighter. Adds depth. |

### Score Bands

| Range | Meaning |
|-------|---------|
| **0-30** | Broken or flat terrain |
| **31-50** | Terrain exists but wrong approach or green mountains |
| **51-70** | Recognizable landscape, missing elements |
| **71-85** | Good terrain, minor issues |
| **86-100** | Excellent. All features, correct coloring, proper PBR |

---

## Quick Reference for Agents

| Goal | What to do |
|------|------------|
| Mountains are green, not rocky | Lower slope threshold for rock (try 0.10). Increase height-based rock weight. |
| Texture shows as solid color | Check mesh.uv is assigned. Check material tiling = (1,1). |
| Normal map has no effect | Call mesh.RecalculateTangents(). Check _BumpScale > 0. |
| River doesn't reach map edges | Start/end points must be BEYOND terrain bounds (e.g., +-localSizeX*0.55). |
| River cuts through mountains | Push waypoints laterally away from mountain zones. Sample zoneVal at each waypoint. |
| Larger map looks stretched | Noise must use wx/wz directly with scale=(1,1,1). NOT normalized coords. |
| Want more features on big map | They come automatically from world-space noise. More area = more noise cycles. |
| Terrain too smooth | Add high-frequency noise octave. |
| Terrain too noisy | Reduce high-frequency amplitudes. |

---

## Technical Notes

### Why scale must be (1,1,1)
Using transform scale (e.g., 5,1,5) introduces a hidden coordinate transform that breaks the "1 unit = 1 Unity unit" rule. It also means noise must be multiplied by scale to get world coordinates, which is error-prone and unintuitive. With scale=(1,1,1), vertex positions ARE world positions. Simple, correct, no confusion.

### Why world-space noise prevents stretching
Perlin noise frequencies (e.g., k=0.003) are in cycles per world unit. A 500-unit span covers 500*0.003 = 1.5 noise cycles. A 1000-unit span covers 3.0 cycles — double the features, same feature SIZE. This is the key property that makes larger maps richer, not stretched.

### UV coordinates
The mesh MUST have UVs assigned: `mesh.uv = uvArray` where each vertex gets `(ix/(vertsX-1), iz/(vertsZ-1))`. Without UVs, Unity cannot map 2D textures onto the 3D mesh. The terrain will appear as a single flat color. This is the #1 mistake LLMs make with procedural meshes.

### Tangents for normal maps
`mesh.RecalculateTangents()` must be called AFTER setting vertices, triangles, normals, and UVs. Without tangents, the normal map will produce incorrect lighting — the surface will look flat or have artifacts despite having a normal map assigned.

### Material tiling
All texture slots on Ground.mat must have tiling set to `(1, 1)` and offset `(0, 0)`. If any slot (especially _BumpMap) has a different tiling value, the texture will repeat across the terrain creating visible tiling artifacts.

### River flowing off edges
Rivers that start/end inside the terrain look artificial — they appear to start from nowhere and end abruptly. River start and end points must be placed BEYOND the terrain bounds so the river naturally enters and exits the map edge. This is a small detail that dramatically improves realism.
