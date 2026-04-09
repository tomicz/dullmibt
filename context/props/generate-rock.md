# Prompt: Generate Rock Details (Singles + Formations)

Use this prompt with a Unity-capable coding agent (Unity MCP) to generate procedural ground rocks with mixed formation styles.

## Purpose

This prompt defines rock detail generation for this world:
- single rocks,
- linear formations,
- small mound/hill clusters,
- terrain-snapped placement with pond exclusion.

## Copy/Paste Prompt

```text
In the active Unity scene, generate procedural rocks on `ProceduralMeshGround`.

Requirements:
1. Ensure target terrain:
   - object: `ProceduralMeshGround`
   - components: MeshFilter + MeshCollider
2. Use/refresh detail container:
   - parent: `GroundDetails/Rocks`
3. Use material:
   - `Assets/Materials/RockGray.mat` (create if missing)
   - low smoothness, neutral gray stone look

Placement constraints:
4. Snap all rocks via downward raycast to terrain.
4b. Embed rocks into terrain by percentage (required):
   - after raycast grounding, sink each rock into ground by ~12%-26% of its world-space height
   - slight deterministic variation per rock is recommended (avoid uniform look)
   - this prevents floating visuals and improves natural placement
5. Exclude pond zones:
   - read pond circles from `Ponds/Pond_*`
   - apply safety margin around shoreline
6. Avoid overlap with existing details using XZ spacing radius checks.
7. Avoid very steep slopes unless explicitly requested.

Generation modes (all required):
8. Singles:
   - spawn many standalone rocks with random scale/rotation
9. Linear formations:
   - spawn groups arranged along a direction vector
   - include slight jitter so lines feel natural, not perfect
10. Mound formations:
   - spawn compact groups in circular clusters
   - vary scale to suggest small natural piles

Shape/style rules:
11. Use primitive shapes for rocks:
   - mostly spheres and some cubes
   - flatten Y scale for grounded stone silhouettes
12. Keep variation:
   - random size, yaw, slight tilt
13. Report counts:
   - singles count
   - linear groups count
   - mound groups count
   - total grouped rock pieces
   - skipped attempts

Constraints:
- Do not place rocks in ponds.
- Do not float rocks above terrain.
- Rocks should be partially buried (embedded), not merely touching terrain.
- Keep formations readable from gameplay camera distance.
```

## Expected Result

- Rocks distributed as a mix of singles, lines, and mounds.
- Natural-looking clutter and composition support.
- No rocks on pond surfaces.
