# Prompt: Generate Procedural Mesh Terrain (No Unity Terrain Tool)

Use this prompt with a Unity-capable coding agent (Unity MCP) to generate uneven terrain using a mesh created from scratch. This version is **size-driven**: when the user says a size/scale, regenerate the mesh geometry for that size instead of only scaling an existing object.

## Why scaling used to look flat

- **Noise on normalized grid (0..1 only):** If height uses only `tx, tz` in `[0,1]`, larger `sizeX/sizeZ` stretches the same number of bumps over more meters, so relief looks almost flat from above.
- **Non-uniform transform scale `(sx, 1, sz)`:** Stretching X/Z but not Y keeps vertical displacement in local units; bumps become tiny compared to horizontal span, so it reads as a flat plane.

**Fix:** Sample noise using **actual XZ positions in mesh/world space** (meters along the ground), with frequencies expressed **per meter**, not per normalized cell. Optionally scale amplitudes slightly with footprint so large areas stay readable.

---

## Copy/Paste Prompt

```text
In the active Unity scene, generate an uneven ground procedurally using a custom Mesh (NOT Unity Terrain).

Inputs (agent must honor these if the user specifies them; otherwise use defaults):
- requestedScale = (1, 1, 1)   // examples: (5,1,5), (10,1,10), (100,1,100)
- localSizeX = 100
- localSizeZ = 100
- vertsX = 201, vertsZ = 201   // increase for very large surfaces

Interpretation rule:
- If user provides scale `(sx,1,sz)`, APPLY it on the object and REGENERATE the mesh heights for that scale in the same run.
- Do not only run `transform.localScale = ...` on an existing mesh. Always rebuild vertices and collider.

Requirements:
1. Delete existing ground objects if present:
   - "ProceduralTerrain"
   - "ProceduralMeshGround"
2. Create GameObject "ProceduralMeshGround" at (0,0,0), and set `transform.localScale = requestedScale`.
3. Add MeshFilter, MeshRenderer, MeshCollider.
4. Build a centered grid:
   - startX = -localSizeX * 0.5, startZ = -localSizeZ * 0.5
   - For each vertex index (x,z), compute local positions:
     lx = startX + (x / (vertsX-1)) * localSizeX
     lz = startZ + (z / (vertsZ-1)) * localSizeZ
5. Height displacement — MUST use world/mesh-space XZ, NOT only normalized 0..1 tx,tz:
   - convert local to world sample position:
     wx = lx * requestedScale.x
     wz = lz * requestedScale.z
   - Use layered Perlin with frequencies in **cycles per unit** (examples, tune for subtle terrain):
     float k0 = 0.04f;   // broad undulation per meter
     float k1 = 0.12f;   // medium detail
     float k2 = 0.28f;   // small detail
   - Offsets (constants) to break symmetry, e.g. +17.1f, +42.3f, etc.
   - n0 = PerlinNoise(wx * k0 + o0, wz * k0 + o0b) - 0.5f
   - n1 = PerlinNoise(wx * k1 + o1, wz * k1 + o1b) - 0.5f
   - n2 = PerlinNoise(wx * k2 + o2, wz * k2 + o2b) - 0.5f
   - Base amplitudes (local Y): a0=0.35f, a1=0.12f, a2=0.04f
   - Scale amplitude by requested horizontal scale so bigger `x/z` scales do not become flat:
     float scaleComp = max(requestedScale.x, requestedScale.z);
     h = (n0*a0 + n1*a1 + n2*a2) * scaleComp;
   - Clamp gently, e.g. h = Mathf.Clamp(h, -1.2f, 1.2f) (adjust if user wants flatter or rougher).
6. Vertex = (lx, h, lz). Build triangle indices, RecalculateNormals, RecalculateBounds.
7. UInt32 index buffer if vertex count > 65535.
8. Assign "Assets/Materials/Ground.mat" if present; else create URP Lit (fallback Standard) green ground material.
9. Apply hybrid terrain shading (required):
   - compute terrain `minH` and `maxH` from generated mesh vertices
   - generate a texture (e.g. `Assets/Textures/GroundHeightTint.png`) by mapping height to color gradient:
     - low elevations: darker green
     - high elevations: brighter/more saturated green
   - include subtle sun influence in the shading bake:
     - use terrain normal and directional light direction (N dot L)
     - brighten sun-facing areas slightly and darken away-facing areas slightly
     - keep this influence subtle so height gradient remains the primary style cue
   - recommended model:
     - `hybrid = lerp(heightColor * 0.9, heightColor * 1.1, sunExposure)`
     - where `sunExposure` is remapped/smoothed from `saturate(dot(normal, lightDir))`
   - set this texture on ground material (`_BaseMap` and/or `_MainTex`)
   - set material tint color to white so texture colors are preserved
10. Generate procedural surface-detail PNG maps from the same height data (required):
   - create/ensure `Assets/Textures/Procedural/`
   - bake `Assets/Textures/Procedural/GroundHeightMap.png` (grayscale; low=dark, high=bright)
   - bake `Assets/Textures/Procedural/GroundNormalMap.png` from the height field using finite differences:
     - sample neighboring heights `hL,hR,hD,hU`
     - compute tangent-space-ish normal:
       - `dx = (hR - hL) * normalStrength`
       - `dz = (hU - hD) * normalStrength`
       - `n = normalize(float3(-dx, 1, -dz))`
     - encode to color: `rgb = n * 0.5 + 0.5`, write PNG
   - import normal texture as a normal map:
     - if URP Lit, assign `_BumpMap` and `_BumpScale`
     - if Standard, assign `_BumpMap` and `_BumpScale`
   - this assignment is mandatory: the generated normal map must be visibly applied on `ProceduralMeshGround` material in the same run
   - optional but recommended for procedural grass distribution:
     - bake `Assets/Textures/Procedural/GroundGrassMask.png`
     - white where slope is low and height is in mid/high band, black elsewhere
     - slope hint: `slope01 = 1 - saturate(dot(normal, up))`
     - grass mask hint: `grass = step(slope01, 0.35) * smoothstep(0.35, 0.75, height01)`
11. Add procedural grass directly on the mesh terrain (required, no Unity Terrain details system):
   - create child object `ProceduralGrass` under `ProceduralMeshGround`
   - clear previous generated grass children if present before respawning
   - spawn lightweight grass instances only where `GroundGrassMask` allows placement
   - placement rules:
     - use jittered grid or blue-noise-like sampling across terrain footprint
     - reject steep areas using slope threshold from mesh normals
     - use deterministic random seed so regeneration is stable/repeatable
   - per-instance variation:
     - random yaw rotation
     - small random scale variation (e.g. 0.8..1.25)
     - slight color variation around base grass tint
   - grass material/shading (mandatory for visible 3D effect):
     - create/use `Assets/Materials/ProceduralGrass.mat`
     - shader: URP Lit when available (fallback Standard with cutout)
     - enable alpha clipping/cutout so grass cards keep blade silhouette
     - set material to render both sides (double-sided/cull off) for card grass
     - assign a grass normal map to the grass material (`_BumpMap`) and set `_BumpScale` (start around `0.8`)
     - if no external grass normal exists, generate `Assets/Textures/Procedural/GrassNormalMap.png` (simple blade-like normal pattern) and use it
     - ensure grass mesh has valid normals/tangents (recalculate/provide tangents) so normal map lighting works
     - optional wind (small): vertex offset by world-space noise + time; keep subtle
   - PBR-like procedural texture set (required for best quality):
     - generate `Assets/Textures/Procedural/GrassAlbedoAlpha.png`
       - RGB: natural green variation (base + hue/value noise + blade-tip brightening + occasional dry/yellow tint)
       - A: blade opacity mask (soft edges, solid center vein)
     - generate `Assets/Textures/Procedural/GrassNormalMap.png`
       - blade longitudinal normals (vertical fiber direction) + micro noise
     - generate `Assets/Textures/Procedural/GrassMaskMap.png` (channel packed)
       - R (Metallic): near 0 for organic grass (`0.0..0.08`)
       - G (AO): darker at blade base/veins (`0.55..1.0`)
       - B (Detail/unused): optional micro variation
       - A (Smoothness): non-plastic leaf response (`0.15..0.45`, wetter grass can go higher)
     - assign maps to grass material:
       - Base map/albedo = `GrassAlbedoAlpha`
       - Normal = `GrassNormalMap`
       - Mask/metallic-smoothness map = `GrassMaskMap` (or Standard equivalents)
     - set physically-plausible defaults:
       - metallic low (`~0.02`)
       - smoothness medium-low (`~0.28`)
       - normal strength (`0.8..1.6`)
       - alpha cutoff (`0.35..0.55`) to avoid fuzzy halos
     - keep color space and import settings consistent:
       - albedo sRGB ON
       - normal map import type = Normal Map
       - mask map sRGB OFF (linear)
   - lighting response tuning (required to feel PBR-ish):
     - enable/specify normal-based shading on both sides of cards
     - add subtle variation by world-space up factor (tops slightly brighter than bases)
     - avoid over-saturated pure green; clamp to natural luminance range
     - if URP supports it in project setup, prefer lit shading model with additional lights enabled
   - rendering approach (pick best available):
     - preferred: GPU instancing (`Graphics.DrawMeshInstanced`/equivalent) with simple grass quad/cross mesh
     - fallback: pooled mesh child instances with shared material
   - performance guardrails:
     - expose density control (instances per square unit)
     - clamp max instances (example: 20k-100k depending on target)
     - frustum-distance culling or chunk-based enable/disable for very large terrains
12. Return confirmation including:
   - requestedScale
   - world footprint (`localSizeX * requestedScale.x`, `localSizeZ * requestedScale.z`)
   - height range min/max
   - note that noise used world-space `wx,wz`
   - include height-tint texture path
   - include generated PNG paths (height, normal, and grass mask if created).
   - include grass generation stats (placed instance count, density, slope threshold).
   - include grass material info:
     - shader name
     - grass normal map path
     - bump scale value
     - alpha clip enabled state
     - grass albedo/mask map paths
     - metallic and smoothness values used
     - alpha cutoff value

Constraints:
- Do not use Unity Terrain / TerrainData.
- Object name exactly "ProceduralMeshGround".
- Uneven, natural ground — not tall mountains unless user asks.
- Do not rely on normalized (tx,tz) alone for noise input; wx,wz are required for scalable terrain.
```

## Expected Result

- One `ProceduralMeshGround` object.
- One `ProceduralGrass` child system attached to `ProceduralMeshGround`.
- Scale reflects user request (`1,1,1`, `5,1,5`, `10,1,10`, `100,1,100`, etc.).
- Mesh is regenerated (not just transformed), collider updated, and terrain remains visibly uneven at larger scales.
- Terrain color varies by elevation (darker lows, greener highs) and is subtly influenced by sun direction.
- Surface depth reads more 3D because a generated normal-map PNG is assigned to the material.
- Grass appears procedurally on flatter/eligible parts of the generated mesh terrain.
- Grass has visible lighting depth (normal-mapped shading), so cards read as more 3D.
- Grass uses a procedural PBR-like texture set (albedo+alpha, normal, mask/smoothness) for more realistic response.
- No Unity Terrain in the scene.

## Quick reference for agents

| Goal | What to do |
|------|------------|
| Bigger ground, same “bump size” in meters | Keep `k0,k1,k2` fixed and sample with world `wx,wz`; increase resolution when needed. |
| Add visual depth to terrain | Regenerate hybrid tint texture from mesh heights + sun exposure and assign to ground material base map. |
| Add stronger fake 3D detail | Bake `GroundNormalMap.png` from generated heights and assign as `_BumpMap` with tuned `_BumpScale` (e.g. 0.4-1.2). |
| Prepare procedural grass placement | Bake `GroundGrassMask.png` using slope + height thresholds for later spawning/instancing. |
| Actually place grass on terrain | Spawn procedural instances under `ProceduralGrass` using mask + slope filter + density cap. |
| Grass still looks flat | Verify `ProceduralGrass.mat` has `_BumpMap` assigned, `_BumpScale > 0`, alpha clip on, and grass mesh tangents are valid. |
| Grass looks too plastic | Lower smoothness (A channel or slider), keep metallic near zero, and reduce saturated albedo greens. |
| Grass edges look fuzzy | Raise alpha cutoff slightly and tighten alpha mask to keep blade silhouettes crisp. |
| Flatter overall | Lower `a0,a1,a2` or tighten clamp. |
| Rougher (still not mountains) | Raise `a2` / `k2` slightly or loosen clamp a little. |
| User wants `localScale (10,1,10)` | Apply that scale and regenerate mesh; multiply displacement by `max(sx,sz)` so it does not look flat. |
