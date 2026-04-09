# Prompt: Visual Polish Pass (Lighting + Graphics)

Use this prompt with a Unity-capable coding agent (Unity MCP) to apply the same lighting/graphics polish we just used in this project.

## What this pass does

This pass improves visual quality in a stylized low-poly scene by tuning:
- directional sunlight and shadows,
- ambient light + fog depth,
- post-processing (global volume),
- ground material response.

It is intentionally subtle (no extreme bloom, no heavy vignette).

## Copy/Paste Prompt

```text
Apply a stylized low-poly visual polish pass in the active Unity scene.

1) Directional Light tuning
- Find "Directional Light" (or first directional light in scene).
- Set:
  - color = (1.00, 0.965, 0.90)
  - intensity = 1.55
  - shadows = Soft
  - shadowStrength = 0.82
  - shadowBias = 0.045
  - shadowNormalBias = 0.35
- If object exists by name, set rotation:
  - Euler(42, 28, 0)

2) Environment / atmosphere
- Set RenderSettings ambient mode = Trilight.
- Set ambient colors:
  - sky = (0.57, 0.66, 0.79)
  - equator = (0.47, 0.55, 0.62)
  - ground = (0.29, 0.33, 0.30)
- Enable fog:
  - fogMode = ExponentialSquared
  - fogColor = (0.73, 0.80, 0.90)
  - fogDensity = 0.003

3) Global Volume post-processing
- Ensure a global Volume exists named "Global Volume".
- Ensure VolumeProfile exists.
- Ensure overrides exist and set:
  - ColorAdjustments:
    - postExposure = 0.12
    - contrast = 12
    - saturation = 6
    - colorFilter = (1.00, 0.985, 0.97)
  - Bloom:
    - active = true
    - threshold = 1.15
    - intensity = 0.22
    - scatter = 0.62
  - Vignette:
    - active = true
    - intensity = 0.15
    - smoothness = 0.45

4) Ground material tweak
- Load `Assets/Materials/Ground.mat` if present.
- Set (if properties exist):
  - Smoothness = 0.06
  - Metallic = 0.00
  - BaseColor/Color = (0.40, 0.59, 0.35)

5) Save and report
- Mark edited assets dirty if needed.
- Save assets.
- Return a short summary of which sections were updated.
```

## Expected Result

- Better depth and readability from lighting and fog.
- Cleaner color balance with subtle stylized grade.
- Ground looks less plastic, more natural.
- Scene remains performant and not over-processed.

## Optional follow-up presets

After this pass, optionally create mood variants by only changing these controls:
- **Golden hour:** lower sun angle, warmer light color, slightly higher fog density
- **Midday clean:** cooler-white sun, reduced fog density, slightly lower vignette
- **Overcast:** softer contrast, cooler fog, lower bloom intensity
