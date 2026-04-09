# Prompt: Generate Volumetric Cloud Systems (Primitive Shapes)

Use this prompt with a Unity-capable coding agent (Unity MCP) to generate realistic cloud **systems** using overlapping primitive spheres.

## Purpose

This prompt defines a reusable cloud workflow:
- chunked cloud distribution (large groups + medium groups + singles),
- each cloud built from multiple sphere puffs,
- soft white cloud material,
- grouped under one parent object,
- visual-only setup (no colliders on puffs).

## Copy/Paste Prompt

```text
In the active Unity scene, create realistic volumetric cloud systems using primitive spheres.

Cloud requirements:
1. Create/replace a parent GameObject named `VolumetricClouds`.
2. Build chunk hierarchy under parent:
   - large merged chunks: `CloudChunk_L1..N`
   - medium chunks: `CloudChunk_M1..N`
   - singles container: `CloudSingles` with `CloudSingle_<index>`
3. Place all cloud systems in sky height band (default center around y=28..32 with variation).
3. Create/load material `Assets/Materials/CloudSoftWhite.mat`:
   - shader: URP Lit (fallback Standard)
   - color: soft white (example RGB: 0.95, 0.97, 1.0)
4. Chunked distribution (required):
   - Large merged groups: few but prominent (e.g. 4..8 groups)
   - Medium groups: more frequent (e.g. 8..16 groups)
   - Singles: scattered standalone clouds (e.g. 30..80)
5. For each cloud object, create overlapping puffs:
   - large-group clouds: ~9..15 puffs each
   - medium-group clouds: ~7..11 puffs each
   - singles: ~5..9 puffs each
   - each puff named `CloudPuff_<index>`
6. Puff shaping rules:
   - overlap puffs to form dense center and softer edges
   - slightly flatter lower puffs, fuller middle, lighter top
   - avoid perfectly symmetric layouts
7. Assign cloud material to each puff renderer.
8. Remove puff colliders (visual-only clouds).
9. Return confirmation including:
   - large group count
   - medium group count
   - singles count
   - total cloud objects
   - total puff count

Suggested presets:
- Balanced:
  - large groups: 4
  - medium groups: 8
  - singles: 32
- Dense (current preferred look):
  - large groups: 7
  - medium groups: 14
  - singles: 70

Constraints:
- Use primitive spheres only (no particle system, no VFX graph, no terrain cloud tools).
- Keep cloud systems grouped under `VolumetricClouds`.
- Keep naming exactly as specified unless user requests different names.
```

## Expected Result

- A `VolumetricClouds` parent with large chunks, medium chunks, and singles.
- Natural sky composition: some big merged cloud masses, some medium groups, and isolated clouds.
- Convincing volumetric cloud silhouette from primitive geometry.
