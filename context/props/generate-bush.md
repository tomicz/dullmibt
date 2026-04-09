# Prompt: Generate Bush Details (Half-Buried Singles + Groups)

Use this prompt with a Unity-capable coding agent (Unity MCP) to generate procedural bushes that look rooted in terrain.

## Purpose

This prompt defines bush generation standards for this world:

- half-buried bush look (top-half visible),
- single bushes and grouped bushes,
- terrain-snapped placement with pond exclusion.

## Copy/Paste Prompt

```text
In the active Unity scene, generate procedural bushes on `ProceduralMeshGround`.

Requirements:
1. Ensure target terrain:
   - object: `ProceduralMeshGround`
   - components: MeshFilter + MeshCollider
2. Use/refresh detail container:
   - parent: `GroundDetails/Bushes`
3. Use material:
   - `Assets/Materials/BushGreen.mat` (create if missing)
   - matte green vegetation look

Placement constraints:
4. Snap bush roots via downward raycast to terrain.
5. Exclude pond zones:
   - read pond circles from `Ponds/Pond_*`
   - apply shoreline safety margin
6. Avoid overlap with existing details in XZ.
7. Avoid very steep slopes unless requested.

Bush style rules:
8. Generate both styles:
   - single bushes (one main puff)
   - grouped bushes (2-4 puffs)
9. Primitive shape:
   - sphere-based puffs
10. Half-buried requirement:
   - each bush core should be buried ~45%-60% into terrain
   - only upper half (or slightly more) remains visible
11. Group behavior:
   - secondary puffs offset around core with variation in size/height
12. Remove colliders from bush puffs (visual detail only)
13. Report counts:
   - singles count
   - grouped bushes count
   - skipped attempts

Constraints:
- Do not place bushes in ponds.
- Do not leave bushes floating above terrain.
- Preserve half-buried rooted appearance.
```

## Expected Result

- Bushes feel integrated into terrain, not sitting on top.
- Visible variety from singles and grouped clumps.
- No bushes on pond surfaces.

