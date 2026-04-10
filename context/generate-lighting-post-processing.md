# Prompt: Lighting & Post-Processing Setup

Use this prompt with a Unity-capable coding agent (Unity MCP) to configure realistic lighting, shadows, and post-processing for the procedural terrain scene. This layer makes the world look finished — proper sun angle, ambient light, shadow quality, and cinematic color grading.

## Critical Rule: Previous Layers are READ-ONLY

**Do NOT modify terrain mesh, terrain textures, water surfaces, or props.** This layer only configures lighting, shadows, environment settings, and post-processing. It does not add or remove any GameObjects from previous layers.

## Prerequisites

- Layers 1-4 must be completed (terrain, splat, water, props all present in scene).
- URP (Universal Render Pipeline) must be the active render pipeline.
- Camera must exist in the scene.

---

## Copy/Paste Prompt

```text
Configure lighting, shadows, and post-processing for the existing procedural terrain scene.
Do NOT modify terrain, water, props, or any previous layer output. This layer only configures
rendering settings to make the scene look polished and realistic.

1. DIRECTIONAL LIGHT (Sun)
   - Find or create a Directional Light named "Sun".
   - Rotation: (50, -30, 0) — sun angled from upper-left, creates long shadows across terrain.
   - Color: warm sunlight (1.0, 0.96, 0.88) — NOT pure white.
   - Intensity: 1.2 (bright but not blown out).
   - Shadow type: Soft Shadows.
   - Shadow resolution: 2048 (or highest available).
   - Shadow distance: 300 (covers most of the visible terrain).
   - Shadow cascades: 4 (for quality shadow transitions at distance).
   - Shadow bias: 0.05 (minimize shadow acne without peter-panning).
   - Shadow normal bias: 0.4.

2. AMBIENT LIGHTING
   - Ambient mode: Gradient (NOT flat color).
   - Sky color: (0.55, 0.65, 0.82) — soft blue sky bounce.
   - Equator color: (0.45, 0.50, 0.45) — muted greenish horizon bounce.
   - Ground color: (0.20, 0.18, 0.15) — dark earth bounce from below.
   - Ambient intensity: 1.0.

3. SKYBOX
   - Use the default URP procedural skybox or a gradient skybox.
   - If procedural: Sun Size = 0.04, Sun Size Convergence = 5,
     Atmosphere Thickness = 1.0, Exposure = 1.3.
   - Sky tint: (0.55, 0.65, 0.85) — slightly blue, not gray.
   - Ground tint: (0.30, 0.28, 0.25) — warm dark ground below horizon.

4. POST-PROCESSING VOLUME
   - Create a Global Volume GameObject named "PostProcessVolume".
   - Create a NEW Volume Profile asset at Assets/Settings/TerrainPostProcessProfile.asset.
   - Assign the profile to the Volume. Set Volume weight = 1.0, Priority = 1.
   - The camera MUST have post-processing enabled (renderPostProcessing = true).

   Add these overrides to the profile:

   TONEMAPPING:
   - Mode: ACES (filmic look, prevents blown-out highlights).

   BLOOM:
   - Threshold: 1.5 (only very bright areas bloom — sun reflections on water).
   - Intensity: 0.15 (subtle, not glowy).
   - Scatter: 0.6.

   COLOR ADJUSTMENTS:
   - Post Exposure: 0.1 (slight brightness boost).
   - Contrast: 10 (subtle punch).
   - Saturation: 8 (slightly more vivid colors).
   - Color Filter: (1.0, 0.98, 0.95) — very subtle warm tint.

   VIGNETTE:
   - Intensity: 0.25 (subtle darkening at screen edges).
   - Smoothness: 0.4.

   AMBIENT OCCLUSION (SSAO):
   - Intensity: 0.5 (visible AO in crevices and at prop bases).
   - Radius: 0.3.
   - Direct Lighting Strength: 0.25.
   - If SSAO is not available in current URP version, skip this.

5. SHADOW SETTINGS (URP Asset)
   - If accessible, configure the URP render pipeline asset:
     Shadow Distance: 300.
     Shadow Cascades: 4.
     Soft Shadows: enabled.
   - If the URP asset is not programmatically accessible, configure via the light.

6. FOG (optional — subtle depth cue)
   - RenderSettings.fog = true (override previous layer's fog=false now that all layers are done).
   - Fog mode: Linear.
   - Fog start: 150.
   - Fog end: 500.
   - Fog color: (0.60, 0.68, 0.78) — blue-gray distance haze.
   - Fog should NOT obscure nearby terrain — only add depth at distance.

7. CAMERA SETUP
   - Find the Main Camera.
   - Near clip: 0.3, Far clip: 1500.
   - Field of view: 60.
   - Ensure URP camera data has:
     renderPostProcessing = true
     antialiasing = SubPixelMorphologicalAntiAliasing (SMAA) or FXAA
     antialiasingQuality = High

8. REPORT
   Return summary:
   - Sun: rotation, color, intensity, shadow settings
   - Ambient: mode, colors
   - Post-processing: profile path, overrides enabled
   - Fog: enabled/disabled, range
   - Camera: FOV, clip planes, AA mode
   - Confirm: no previous layer objects were modified
```

---

## Expected Result

- Scene has a **warm directional sun** casting soft shadows across the terrain.
- **Ambient lighting** provides soft fill from sky and ground, preventing pitch-black shadows.
- Mountains cast **long, visible shadows** across valleys and plains.
- **Post-processing** adds cinematic polish: filmic tonemapping, subtle bloom on water, mild vignette.
- Colors are **slightly warm and vivid** without being oversaturated.
- **Distance fog** adds atmospheric depth — far mountains fade into blue haze.
- Props (trees, rocks) cast shadows on the terrain, adding visual grounding.
- The scene reads as a **time of day** (late morning / golden hour feel).

---

## Scoring Rubric (100 points)

### Lighting Quality (35 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Sun direction** | 8 | Angled light creating visible shadows across terrain. Not directly overhead (flat). Not too low (everything in shadow). |
| **Sun color/intensity** | 5 | Warm tint, not pure white. Intensity balanced — terrain readable, not blown out. |
| **Shadow quality** | 10 | Soft shadows. Visible on terrain from mountains and props. No shadow acne. Reasonable shadow distance. |
| **Ambient lighting** | 7 | Gradient ambient (not flat). Shadows have visible fill light, not pitch black. Sky/ground colors appropriate. |
| **Skybox** | 5 | Blue sky visible. Not default gray. Sun disk present. Ground color below horizon. |

### Post-Processing Quality (35 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Tonemapping** | 8 | ACES or similar filmic curve. Highlights don't clip to white. Shadows retain detail. |
| **Bloom** | 5 | Subtle. Only on bright highlights (water, sun-facing slopes). Not a glow filter on everything. |
| **Color grading** | 8 | Slight warmth, mild saturation boost. Scene feels cinematic, not flat or oversaturated. |
| **Vignette** | 4 | Subtle edge darkening. Not a tunnel. Draws eye to center. |
| **SSAO** | 5 | Visible ambient occlusion in crevices, at tree bases, rock edges. Adds depth. |
| **Volume setup** | 5 | Global Volume with dedicated profile. Camera has post-processing enabled. |

### Environment Quality (20 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Fog** | 8 | Linear fog with reasonable range. Distant terrain fades to blue-gray. Near terrain unaffected. |
| **Camera setup** | 5 | Appropriate FOV, clip planes. Anti-aliasing enabled. |
| **Overall atmosphere** | 7 | Scene has a sense of time-of-day and atmosphere. Not flat default lighting. Reads as a finished scene. |

### Technical Quality (10 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Layer isolation** | 5 | No terrain, water, or prop modifications. Lighting only. |
| **Asset cleanliness** | 3 | Volume profile saved as asset. No orphaned objects. |
| **Performance** | 2 | Settings don't cause editor slowdown. Shadow distance reasonable. |

### Score Bands

| Range | Meaning |
|-------|---------|
| **0-25** | Default lighting or broken post-processing |
| **26-50** | Basic directional light but no post-processing or atmosphere |
| **51-70** | Decent lighting with some post-processing, minor issues |
| **71-85** | Good lighting, shadows, and post-processing. Scene feels polished. |
| **86-100** | Excellent: cinematic atmosphere, proper shadows, filmic grading, depth fog |

---

## Quick Reference for Agents

| Problem | Solution |
|---------|----------|
| Shadows too dark / no fill | Set ambient mode to Gradient with appropriate sky/ground colors. |
| Everything looks flat | Sun angle too high (overhead). Lower X rotation to 40-55 for longer shadows. |
| Bloom too strong | Raise threshold (1.5+), lower intensity (0.1-0.2). |
| Colors washed out | Enable ACES tonemapping. Add slight saturation and contrast. |
| Fog hides everything | Increase fog start distance. Use Linear mode, not Exponential. |
| Post-processing not visible | Check camera has renderPostProcessing = true. Check Volume is Global. |
| Shadow acne on terrain | Increase shadow bias (0.05-0.1). |
| Shadows cut off too close | Increase shadow distance (200-400). |
| SSAO not available | Some URP versions don't include SSAO. Skip if not present. |
| Sky is gray | Configure skybox material or use procedural skybox with blue tint. |

---

## Technical Notes

### Why ACES tonemapping?
ACES (Academy Color Encoding System) provides a filmic S-curve that compresses highlights and lifts shadows. Without tonemapping, bright areas clip harshly to white and the scene looks flat. ACES is the industry standard for realistic rendering.

### Shadow cascades
4 cascades distribute shadow resolution across distance bands. Near the camera, shadows are sharp. Far away, they're softer but still present. Without cascades, either near shadows are blurry or far shadows disappear entirely.

### Ambient gradient vs flat
Flat ambient (single color) fills all shadows equally, making the scene look artificially lit. Gradient ambient uses different colors from sky (blue), horizon (neutral), and ground (dark warm), creating natural-feeling fill light that varies with surface orientation.

### Fog as depth cue
Even subtle fog dramatically improves scene readability by creating aerial perspective — distant objects appear lighter and bluer, mimicking real atmospheric scattering. Start distance must be far enough that nearby terrain is unaffected.
