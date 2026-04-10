# Prompt: Horizon Closure (Procedural Mountain Ring)

Use this prompt with a Unity-capable coding agent (Unity MCP) to generate a ring of mountains around the terrain edges that hides the infinite void beyond the map. This is the final polish layer — it makes the world feel enclosed and complete rather than floating in empty space.

## Critical Rule: Previous Layers are READ-ONLY

**Do NOT modify terrain mesh, terrain textures, water, props, lighting, or clouds.** This layer only ADDS new mountain ring meshes outside the playable terrain bounds.

## Prerequisites

- Layer 1 (terrain) must be completed — `ProceduralMeshGround` must exist.
- The mountain ring sits OUTSIDE the terrain bounds, not overlapping it.

---

## Copy/Paste Prompt

```text
Generate a procedural mountain ring around the edges of the existing terrain to close the horizon.
The ring hides the void beyond the map edges. Do NOT modify any previous layer output.

IMPORTANT RULES:
- transform.localScale is ALWAYS (1, 1, 1) for the ring parent.
- The ring meshes sit OUTSIDE the terrain bounds — they do NOT overlap the playable area.
- Mountains must blend visually with the terrain edges (similar coloring at the boundary).
- The ring must be continuous — no gaps where the void is visible.

1. READ TERRAIN DATA
   - Find "ProceduralMeshGround" — get mesh bounds.
   - Compute terrain world bounds: minX, maxX, minZ, maxZ.
   - Get terrain height at edges (sample mesh vertices near bounds).
   - Get terrain max height (for ring mountain scale reference).

2. RING GEOMETRY
   The ring is a continuous strip of procedural mountain mesh that surrounds the terrain.

   Ring dimensions:
   - Inner edge: starts at terrain bounds (or 5-10m overlap for seamless blending).
   - Outer edge: extends 150-250m beyond terrain bounds.
   - Ring width: 150-250m (the depth of the mountain wall).

   Build the ring as 4 side segments (North, South, East, West) plus 4 corner segments.
   Each segment is a separate mesh for manageable vertex counts.

   Per segment:
   - Create a grid of vertices spanning the segment area.
   - Resolution: ~5m spacing between vertices (enough for mountain shapes).
   - Inner row vertices: match terrain edge height for seamless join.
   - Height generation using Perlin noise:
     Base height = terrain edge average height.
     Add mountain noise: 3 octaves, k=[0.008, 0.02, 0.06], a=[40, 15, 5].
     Height increases with distance from terrain edge (mountains get taller further out).
     Height multiplier = smoothstep(0, ringWidth, distFromEdge) — low at inner edge, tall at outer.
     Peak height: 60-120% of terrain max height.
   - Outermost row: vertices drop below the horizon (Y = -20) to hide the bottom edge.

3. RING MATERIAL
   - Use the same Ground.mat from the terrain OR create Assets/Materials/HorizonRock.mat.
   - If creating new material:
     Shader: Universal Render Pipeline/Lit
     Base Color: rock-like gray-brown (0.38, 0.35, 0.30) — matches terrain rock zones.
     Smoothness: 0.08 (rough rock).
     Metallic: 0.0.
   - The ring should NOT look out of place — it should read as distant mountains
     that are part of the same landscape.

4. COLORING (if using vertex colors or a dedicated tint texture)
   - Inner edge: match terrain edge colors (grass/rock blend).
   - Middle: transition to rock gray.
   - Outer/top: darker rock or snow on peaks if tall enough.
   - Simple approach: use a solid rock material — distant mountains reading as
     gray rock is natural and expected.

5. SHADOW AND RENDERING
   - Shadow casting: ON (ring mountains should cast shadows for realism).
   - Receive shadows: ON.
   - No MeshCollider needed (player doesn't walk on the ring).

6. HIERARCHY
   HorizonRing (root, position 0,0,0, scale 1,1,1)
   ├── Ring_North
   ├── Ring_South
   ├── Ring_East
   ├── Ring_West
   ├── Ring_Corner_NE
   ├── Ring_Corner_NW
   ├── Ring_Corner_SE
   └── Ring_Corner_SW

7. SEAMLESS BLENDING
   The most critical aspect: where the ring meets the terrain edge, there must be
   NO visible seam, gap, or height discontinuity.

   - Sample terrain mesh vertices near each edge.
   - Set ring inner-row vertex heights to match the sampled terrain heights.
   - Overlap the ring 5-10m into the terrain bounds so the meshes blend together.
   - The color/material at the inner edge should be compatible with the terrain edge appearance.

8. REPORT
   Return summary:
   - Segment count and vertex counts per segment
   - Ring width (inner to outer edge)
   - Height range of ring mountains
   - Material used
   - Confirm: seamless blend at terrain edges
   - Confirm: no previous layers modified
```

---

## Expected Result

- Looking outward from anywhere on the terrain, the player sees **mountains on the horizon** instead of empty void.
- The mountain ring is **continuous** — no gaps in any direction.
- Where the ring meets the terrain, the **transition is seamless** — no visible seam or height jump.
- Ring mountains **look like distant terrain** — same visual language as the main terrain (rock coloring, similar scale).
- The ring gives the world a sense of **enclosure and completeness**.

---

## Scoring Rubric (100 points)

### Geometry Quality (35 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Continuous ring** | 10 | No gaps in any direction. Void is fully hidden from any camera angle on the terrain. |
| **Mountain shapes** | 8 | Ring has believable mountain silhouettes with peaks and valleys, not a flat wall. |
| **Height variation** | 7 | Mountains vary in height. Some taller peaks, some lower passes. Not uniform. |
| **Corner coverage** | 5 | Corners are filled — no diagonal void visible. |
| **Outer edge hidden** | 5 | Outermost vertices drop below horizon. No visible bottom edge of the ring mesh. |

### Blending Quality (30 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Height match at seam** | 12 | Ring inner edge matches terrain edge heights exactly. No visible step/gap. |
| **No overlap artifacts** | 8 | Ring and terrain don't z-fight or create visual glitches where they meet. |
| **Visual consistency** | 10 | Ring mountains look like they belong to the same landscape. Color and scale match. |

### Technical Quality (20 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Layer isolation** | 8 | No previous layer modifications. Ring only ADDS meshes. |
| **Hierarchy** | 4 | Clean: HorizonRing → segments. |
| **Performance** | 4 | Reasonable vertex count. No editor freeze. |
| **Scale rule** | 4 | Parent scale (1,1,1). Mesh geometry defines size. |

### Visual Quality (15 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Horizon reads as mountains** | 8 | From ground level, the ring looks like distant mountain ranges. |
| **Atmosphere** | 7 | Combined with fog (if enabled), ring mountains fade naturally into distance. |

### Score Bands

| Range | Meaning |
|-------|---------|
| **0-25** | No ring or ring with large gaps |
| **26-50** | Ring exists but visible seams or flat wall appearance |
| **51-70** | Decent ring with mountain shapes, minor blending issues |
| **71-85** | Good ring with proper blending and varied mountains |
| **86-100** | Excellent: seamless, natural mountain horizon, complete enclosure |

---

## Quick Reference for Agents

| Problem | Solution |
|---------|----------|
| Gap visible in a direction | Add segment or extend existing segment to cover. Check corners. |
| Visible seam at terrain edge | Sample terrain edge heights and match ring inner vertices exactly. |
| Ring looks like a wall | Add more noise octaves for mountain variation. Vary peak heights. |
| Z-fighting at seam | Overlap ring 5-10m into terrain. Or offset ring Y by 0.1 at seam. |
| Bottom of ring visible | Drop outermost row vertices to Y = -20 or lower. |
| Ring too tall / dominates | Reduce peak height multiplier. Ring peaks should be 60-120% of terrain max. |
| Ring too short / void visible | Increase peak height. Ensure outermost row is below camera frustum. |
| Performance issues | Reduce vertex density (8-10m spacing instead of 5m). |

---

## Technical Notes

### Why 8 segments instead of one ring mesh?
A single continuous ring mesh for a 500x500 map would have a very high vertex count. Breaking it into 4 sides + 4 corners keeps each mesh manageable and allows different LOD treatment if needed.

### Seamless blending strategy
The key insight is that the ring's inner edge must be a slave to the terrain's outer edge. Sample the terrain mesh at its boundary, then set the ring's inner vertices to exactly those positions and heights. This creates a geometric weld between the two meshes.

### Height profile across the ring
The ring should NOT be tall at the inner edge (that would look like a wall). Height should increase with distance from the terrain: low at the inner edge (matching terrain), rising to mountain peaks in the middle, then dropping below the horizon at the outer edge. This creates a natural-looking mountain range backdrop.

### Why this is the hardest layer
This layer has no dedicated prompt in many benchmark implementations because it tests whether the agent can solve a spatial problem without step-by-step instructions for the algorithm. The agent must: understand the terrain bounds, generate complementary geometry, ensure seamless blending, and handle corners — all requiring spatial reasoning that many LLMs struggle with.
