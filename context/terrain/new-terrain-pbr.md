# New Terrain Setup (Procedural Mesh + PBR-like Grass)

Use this as the current reference for regenerating the terrain in this project.

## Current target result

- Terrain object: `ProceduralMeshGround`
- Terrain scale: `(5, 1, 5)`
- World footprint: `500 x 500`
- Terrain type: custom mesh (no Unity Terrain)
- Grass type: procedural mesh cards with PBR-like material response

## Generation standard

1. Recreate terrain mesh (`ProceduralMeshGround`) from noise-sampled world-space XZ.
2. Ensure components:
   - `MeshFilter`
   - `MeshRenderer`
   - `MeshCollider`
3. Bake and assign terrain maps:
   - `Assets/Textures/Procedural/GroundHeightMap.png`
   - `Assets/Textures/Procedural/GroundNormalMap.png`
4. Use terrain material:
   - `Assets/Materials/Ground.mat`
   - assign `_BaseMap`/`_MainTex` and `_BumpMap`
   - keep `_BumpScale > 0`

## Grass standard (PBR-like)

1. Create/use grass material:
   - `Assets/Materials/ProceduralGrass.mat`
   - shader: URP Lit (fallback Standard)
2. Generate/use procedural grass textures:
   - `Assets/Textures/Procedural/GrassAlbedoAlpha.png`
   - `Assets/Textures/Procedural/GrassNormalMap.png`
   - `Assets/Textures/Procedural/GrassMaskMap.png`
3. Material requirements:
   - alpha clipping enabled (cutout-style blades)
   - double-sided rendering/cull off for card visibility
   - `_BumpMap` assigned and `_BumpScale` set
   - metallic near 0, medium-low smoothness
4. Spawn grass under:
   - `ProceduralMeshGround/ProceduralGrass`
   - deterministic seed
   - slope-filtered placement

## Placement/grounding rules

- Trees, cone trees, rocks, and bushes must be grounded by downward raycast to `ProceduralMeshGround`.
- Only move root objects for placement correction (do not break child local offsets).
- Keep tree internals consistent:
  - `Tree_Trunk.localPosition.y = Tree_Trunk.localScale.y`
  - `Tree_Canopy.localPosition.y = trunkHeight * 2.6`
  - `Cone_Trunk.localPosition.y = Cone_Trunk.localScale.y`
  - `ProceduralCone.localPosition.y = trunkHeight * 2`

## Validation checklist

- `ProceduralMeshGround` exists and has `MeshCollider`.
- Terrain normal map is assigned and visible.
- Grass normal map is assigned and visible.
- `ProceduralGrass` exists and contains generated instances.
- No visible floating for trees/rocks/bushes after grounding pass.

## Related prompts

- `Prompts/generate-procedural-mesh-terrain.md`
- `Prompts/world-placement-delegator.md`
- `Prompts/generate-cone-tree.md`
- `Prompts/generate-rock-details.md`
- `Prompts/generate-bush-details.md`
