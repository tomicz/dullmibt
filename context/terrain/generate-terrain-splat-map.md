# Prompt: Terrain Splat Mapping (Dynamic Multi-Texture Blending)

Use this prompt with a Unity-capable coding agent (Unity MCP) to apply splat-mapped terrain texturing to `ProceduralMeshGround`. Instead of flat colors, the terrain gets tileable PBR textures per zone — rocky mountains, grassy plains, earthy river banks, snow-capped peaks — blended smoothly via a splat map.

## Prerequisites

- `ProceduralMeshGround` must already exist with a valid mesh and MeshCollider.
- Terrain should have height variation (mountains, hills, plains) and optionally a river valley.
- This prompt runs AFTER `generate-realistic-terrain.md` (or any terrain generation that produces varied topography).

## What is a splat map?

A **splat map** is a texture where each color channel (R, G, B, A) stores the blend weight for a different terrain material. At each point on the terrain, the channels sum to ~1.0, and the final surface appearance is a weighted blend of the corresponding tileable textures.

| Channel | Terrain Type | Where it dominates |
|---------|-------------|-------------------|
| **R** | Grass | Plains, gentle hills, low-mid elevation with low slope |
| **G** | Rock | Steep slopes (>0.35), high altitude (>45m), mountain faces |
| **B** | Dirt/Earth | River bed, river banks, transition areas near water |
| **A** | Snow | Mountain peaks (>55m), only on gentle slopes (<0.3) |

---

## Copy/Paste Prompt

```text
Apply terrain splat mapping to the existing ProceduralMeshGround in the active Unity scene.

This involves three steps:
A) Generate the splat map texture from terrain data
B) Generate tileable PBR texture sets for each terrain zone
C) Bake composite terrain textures by blending zone textures weighted by the splat map

1. READ TERRAIN DATA
   - Find "ProceduralMeshGround" — get its MeshFilter mesh vertices, normals, and transform scale.
   - Compute per-vertex: height (vertex.y), slope (1 - dot(normal, up)).
   - Find "RiverPath" if it exists — reconstruct the river polyline from child transforms
     for river distance queries. If no RiverPath exists, skip river-related splat logic.
   - Record height range (minH, maxH) from mesh vertices.

2. GENERATE SPLAT MAP (Assets/Textures/Procedural/TerrainSplatMap.png)
   - Resolution: 1024x1024, RGBA32, linear color space.
   - For each pixel, map to terrain vertex grid (bilinear interpolation) and compute:
     height, slope, and distance to river (if river exists).

   - Channel weight rules (order of priority, highest first):

     SNOW (A channel):
       weight = smoothstep(50, 60, height) * (1 - smoothstep(0.2, 0.35, slope))
       Snow only appears on high peaks with gentle slopes.

     DIRT (B channel):
       If river exists:
         riverBed: distance < 10 → weight = 1.0
         riverBank: distance 10-22 → weight = smoothstep(22, 10, distance) * 0.8
       If no river: weight = 0
       Also add dirt on very steep low-altitude areas:
         weight += smoothstep(0.5, 0.7, slope) * smoothstep(30, 0, height) * 0.4

     ROCK (G channel):
       weight = max(smoothstep(0.3, 0.55, slope), smoothstep(40, 55, height) * 0.6)
       Rock dominates steep slopes and high altitudes.
       Reduce where snow or dirt already dominate:
         weight *= (1 - snowWeight) * (1 - dirtWeight * 0.5)

     GRASS (R channel):
       weight = remainder to normalize: max(0, 1 - snowWeight - rockWeight - dirtWeight)
       Grass fills wherever other types don't claim.

   - Normalize: total = R + G + B + A. If total > 0, divide each by total.
   - Write RGBA pixel.

   - Import settings: sRGB OFF (linear), isReadable = true.

3. GENERATE TILEABLE PBR TEXTURE SETS (procedural, 512x512 each)
   Each terrain zone needs three textures: Albedo, Normal, Roughness/AO.
   All tileable (edges wrap seamlessly). Generate procedurally using Perlin noise patterns.

   GRASS textures (Assets/Textures/Procedural/SplatGrass*.png):
   - Albedo: Base green (0.28, 0.52, 0.18) with multi-octave noise variation.
     Add subtle yellow/brown patches. Frequency: k=0.08 for broad patches, k=0.25 for blade detail.
   - Normal: Vertical blade-like pattern. Gentle undulation suggesting grass blades.
     Frequency: k=0.15 vertical emphasis (stronger Z component variation).
   - Mask (R=metallic~0, G=AO with variation 0.6-1.0, A=smoothness 0.15-0.35):
     Low metallic, moderate AO variation, low smoothness (matte grass).

   ROCK textures (Assets/Textures/Procedural/SplatRock*.png):
   - Albedo: Gray-brown base (0.38, 0.36, 0.32) with craggy noise.
     Multi-octave: k=0.05 for large face variation, k=0.15 medium cracks, k=0.4 fine grain.
     Color variation toward brown in crevices, lighter gray on exposed faces.
   - Normal: Strong, high-contrast normals suggesting cracks and rough surface.
     Higher normalStrength (2.0-3.0). Multi-frequency for both large cracks and fine grain.
   - Mask: metallic~0.02, AO with deep crevice darkening (0.4-1.0), smoothness 0.15-0.30.

   DIRT textures (Assets/Textures/Procedural/SplatDirt*.png):
   - Albedo: Earth brown base (0.32, 0.24, 0.16) with subtle variation.
     Slightly darker in wet/river areas. k=0.06 broad, k=0.2 medium, k=0.5 fine grain.
   - Normal: Gentle, low-contrast normals. Packed earth feel, not dramatic.
     normalStrength ~1.0. Subtle pebble-like bumps at high frequency.
   - Mask: metallic~0, AO moderate (0.7-1.0), smoothness 0.20-0.45 (slightly smoother than rock,
     suggesting compacted/damp earth).

   SNOW textures (Assets/Textures/Procedural/SplatSnow*.png):
   - Albedo: Near-white base (0.92, 0.93, 0.96) with very subtle blue-gray variation.
     Gentle drift patterns at k=0.04. Minimal contrast.
   - Normal: Very soft, low-frequency undulations suggesting snowdrift shapes.
     normalStrength ~0.5. Almost flat but with gentle rolling bumps.
   - Mask: metallic~0, AO minimal (0.85-1.0), smoothness 0.40-0.65 (snow is moderately shiny).

   Import settings for all:
   - Albedo: sRGB ON, wrap mode = Repeat
   - Normal: texture type = Normal Map, wrap mode = Repeat
   - Mask: sRGB OFF (linear), wrap mode = Repeat

4. BAKE COMPOSITE TERRAIN TEXTURES
   Read the splat map and the four tileable texture sets.
   For each pixel in the output textures, sample the splat map to get weights (R,G,B,A),
   then sample each tileable texture set at a tiled UV (tiling factor = 40 for 512px tiles
   across 1024px output → each tile repeats ~80 times across the full terrain).

   Bake three composite textures:

   COMPOSITE ALBEDO (Assets/Textures/GroundHeightTint.png — replaces existing, 2048x2048):
     color = grassAlbedo * R + rockAlbedo * G + dirtAlbedo * B + snowAlbedo * A
     Apply sun exposure tint (same as terrain prompt):
       sunExposure = saturate(dot(normal, lightDir))
       color *= lerp(0.88, 1.12, smoothstep(0.2, 0.8, sunExposure))

   COMPOSITE NORMAL (Assets/Textures/Procedural/GroundNormalMap.png — replaces existing, 2048x2048):
     Blend normals using reoriented normal mapping (RNM) or simple weighted average:
       n = normalize(grassNormal * R + rockNormal * G + dirtNormal * B + snowNormal * A)
     Combine with the terrain geometry normal for macro + micro detail.

   COMPOSITE MASK (Assets/Textures/Procedural/GroundMaskMap.png — new, 1024x1024):
     Blend per-channel: metallic, AO, smoothness weighted by splat values.
     R = blended metallic, G = blended AO, B = 0, A = blended smoothness.

5. ASSIGN TO MATERIAL
   - Load Assets/Materials/Ground.mat
   - Set _BaseMap = composite albedo (GroundHeightTint.png)
   - Set _BumpMap = composite normal (GroundNormalMap.png), _BumpScale = 1.0
   - Set _MetallicGlossMap or _MaskMap = composite mask (GroundMaskMap.png)
   - Set _Smoothness from mask map (if URP Lit supports it)
   - Set _BaseColor = white (let textures drive color)
   - Mark material dirty, save assets.

6. REPORT
   Return summary:
   - Splat map path and channel coverage percentages (% of terrain that is grass/rock/dirt/snow)
   - Tileable texture set paths (12 textures: 4 zones x 3 maps)
   - Composite texture paths and resolutions
   - Material assignments confirmed
   - Tiling factor used
```

---

## Expected Result

- Terrain surface shows **visually distinct zones** with tileable texture detail:
  - Mountains have rough, craggy rock texture with visible crack normals.
  - Plains and hills have natural grass texture with blade-like normal detail.
  - River banks and bed show compacted earth/dirt texture.
  - Mountain peaks have subtle snow texture with soft drift normals.
- Transitions between zones are **smooth blends**, not hard edges.
- Close-up inspection shows **tileable texture detail** (not flat color) across the entire terrain.
- Normal map adds both **macro terrain shape** and **micro surface detail** from zone textures.
- The terrain reads as a real landscape even when the camera is close to the ground.

---

## Scoring Rubric (100 points)

### Splat Map Quality (30 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Channel separation** | 8 | Each channel clearly maps to the correct terrain zone. Grass on plains, rock on steep/high, dirt near river, snow on peaks. |
| **Smooth transitions** | 8 | No hard edges between zones. smoothstep blending creates natural gradients. |
| **River awareness** | 7 | River bed and banks are correctly identified with dirt channel. Dirt fades with distance from river. |
| **Normalization** | 7 | Channels sum to ~1.0 everywhere. No oversaturated or black areas. |

### Tileable Texture Quality (30 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Visual identity** | 8 | Each zone has a distinct, recognizable texture. Rock looks rocky, grass looks grassy, etc. |
| **Tileability** | 7 | Textures wrap seamlessly. No visible seam lines when tiled. |
| **PBR correctness** | 8 | Normal maps add convincing surface detail. Mask maps have physically plausible metallic/smoothness/AO values. |
| **Color coherence** | 7 | Colors are natural and muted. No neon, no oversaturation. Rock is gray-brown, grass is natural green, dirt is earth brown, snow is near-white. |

### Composite Bake Quality (25 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Blending correctness** | 8 | Composite textures show proper weighted blending. No channel leaking or wrong zone showing. |
| **Tiling detail** | 7 | Close-up shows tileable texture detail, not flat color. Tiling factor produces appropriate scale. |
| **Normal composition** | 5 | Composite normal includes both terrain macro shape and zone micro detail. |
| **Sun exposure** | 5 | Subtle directional tint baked in. Sun-facing slopes slightly brighter. |

### Technical Quality (15 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Texture specs** | 5 | Correct resolutions, color spaces (sRGB for albedo, linear for normal/mask), import settings. |
| **Material setup** | 5 | All maps assigned to correct material slots. _BumpScale > 0. Tint color = white. |
| **Performance** | 5 | Bake completes in under 15 seconds. Texture count reasonable. No editor freeze. |

### Score Bands

| Range | Meaning |
|-------|---------|
| **0-25** | Broken or no splat mapping applied |
| **26-50** | Splat map exists but textures are flat colors or wrong zones |
| **51-70** | Working splat map with textures but poor transitions or tiling issues |
| **71-85** | Good splat mapping with distinct zones, minor quality issues |
| **86-100** | Excellent terrain texturing, smooth blends, convincing PBR detail |

---

## Quick Reference for Agents

| Goal | What to do |
|------|------------|
| Snow line too low/high | Adjust the height threshold in snow weight (default: smoothstep 50-60) |
| Rock showing on flat ground | Increase slope threshold in rock weight (default: smoothstep 0.3-0.55) |
| Dirt zone too wide/narrow | Adjust river distance thresholds (default: bed <10, bank 10-22) |
| Textures too repetitive | Reduce tiling factor (default: 40) or add a second noise layer to break tiling |
| Transitions too sharp | Widen smoothstep ranges in splat weight calculations |
| Grass too bright/dark | Adjust grass albedo base color (default: 0.28, 0.52, 0.18) |
| Rock too smooth | Increase rock normal strength (default: 2.0-3.0) and lower smoothness |
| Snow too flat | Increase snow normal strength (default: 0.5) |
| Tiling visible at distance | Add a macro variation layer that subtly modulates albedo brightness across the terrain |

---

## Technical Notes

### Why bake composites instead of using a custom shader?
A custom multi-texture shader (sampling 4 texture sets + splat map at runtime) is the industry standard for shipped games. However, for a benchmark testing LLM capability, baking composites is more practical:
1. It works with the standard URP Lit shader — no shader authoring required.
2. It produces a deterministic, inspectable result (you can open the texture and verify).
3. It tests the same spatial reasoning and PBR knowledge.
4. A follow-up benchmark tier could test runtime shader-based splatting as a harder challenge.

### Tiling factor
The tiling factor controls how many times each tileable texture repeats across the terrain. At tiling=40 with 512px tile textures baked into a 2048px composite, each tile covers ~12.5 world units. This gives visible texture detail at close range without being too repetitive at distance.

### Splat map vs height tint
The existing `GroundHeightTint.png` uses flat colors per zone. The splat map system replaces this with actual textured surfaces. The splat map itself is a data texture (not displayed directly) — it drives the composite bake that replaces the height tint.
