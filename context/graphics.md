# Prompt: Graphics V2 (Shadows + Atmosphere + Material Polish)

Use this prompt with a Unity-capable coding agent (Unity MCP) to apply the latest graphics pass for dense procedural worlds (many trees, rocks, bushes, ponds, clouds).

## Purpose

This pass improves readability and visual cohesion without overprocessing:
- softer cloud impact on shadows,
- better tree/object shadow quality and distance,
- stronger atmospheric depth,
- balanced post-processing,
- material tuning for terrain, foliage, rocks, and water.

## Copy/Paste Prompt

```text
Apply the project graphics V2 pass in the active Unity scene.

1) Directional light tuning
- Find `Directional Light` (or first directional light).
- Set:
  - color = (1.00, 0.96, 0.90)
  - intensity = 1.45
  - shadows = Soft
  - shadowStrength = 0.78
  - shadowBias = 0.03
  - shadowNormalBias = 0.22
  - rotation = Euler(40, 32, 0)

2) Cloud shadow softening
- Find `VolumetricClouds`.
- For all child renderers:
  - `shadowCastingMode = Off`
  - `receiveShadows = false`

3) URP shadow distance + cascades
- Access active `UniversalRenderPipelineAsset`.
- Set:
  - `shadowDistance = 260`
  - `shadowCascadeCount = 4`
  - `cascade2Split = 0.35` (if supported)
  - `cascade4Split = (0.08, 0.22, 0.52)` (if supported)

4) Ensure non-cloud world objects participate in shadows
- For renderers under:
  - `Trees`
  - `ConeTrees`
  - `GroundDetails`
  - `Ponds`
- Set:
  - `shadowCastingMode = On`
  - `receiveShadows = true`

5) Atmosphere pass
- RenderSettings:
  - fog = true
  - fogMode = ExponentialSquared
  - fogColor = (0.70, 0.79, 0.89)
  - fogDensity = 0.0038
  - ambientMode = Trilight
  - ambientSky = (0.56, 0.65, 0.77)
  - ambientEquator = (0.46, 0.53, 0.60)
  - ambientGround = (0.30, 0.33, 0.31)

6) Global Volume post-processing retune
- Ensure `Global Volume` exists and is global.
- Ensure profile has:
  - ColorAdjustments:
    - postExposure = 0.08
    - contrast = 10
    - saturation = 8
    - colorFilter = (1.00, 0.99, 0.97)
  - Bloom:
    - active = true
    - threshold = 1.2
    - intensity = 0.18
    - scatter = 0.6
  - Vignette:
    - active = true
    - intensity = 0.12
    - smoothness = 0.42

7) Material balancing
- `Assets/Materials/RockGray.mat` (PBR-style rock setup):
  - use procedural texture set:
    - `Assets/Textures/Procedural/RockAlbedo.png`
    - `Assets/Textures/Procedural/RockNormal.png`
    - `Assets/Textures/Procedural/RockMaskMap.png`
  - assign:
    - `_BaseMap`/`_MainTex` = `RockAlbedo`
    - `_BumpMap` = `RockNormal`
    - `_MaskMap` (or `_MetallicGlossMap`) = `RockMaskMap`
  - tune:
    - metallic ~= `0.01`
    - smoothness ~= `0.24..0.30`
    - bump scale ~= `1.0..1.35`
  - keep rock palette neutral gray and avoid over-bright specular highlights
- `Assets/Materials/BushGreen.mat`:
  - smoothness = 0.06
  - color ~= (0.24, 0.52, 0.23)
- `Assets/Materials/WaterPond.mat`:
  - color ~= (0.17, 0.47, 0.67, alpha 0.86)
  - smoothness = 0.96
- `Assets/Materials/LeavesGreen.mat`:
  - color ~= (0.26, 0.76, 0.30)
- `Assets/Materials/ConeDark.mat`:
  - color ~= (0.10, 0.30, 0.11)

9) Rock composition style (world readability)
- Prefer fewer, larger rocks over many small ones:
  - target large sparse set (example: `35..70` rocks for this world scale)
  - use larger scales to create landmark silhouettes
  - keep wide spacing so terrain and vegetation remain readable
- Grounding rule:
  - no floating rocks
  - embed rocks into terrain by percentage (roughly `18%..35%` for large boulders)

10) Save and report
- Mark changed assets dirty.
- Save assets.
- Return summary counts for updated sections and key shadow settings.
```

## Expected Result

- Cleaner depth and distance readability in dense scenes.
- Softer cloud lighting influence (without noisy cloud shadows).
- Better object shadow continuity at longer range.
- More coherent color balance across terrain, foliage, rocks, and ponds.
