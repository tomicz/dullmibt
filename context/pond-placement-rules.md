# Pond Placement Rules

Use this as the placement standard for all pond generation/update operations.

## Goals

- Place ponds so they are clearly visible.
- Keep ponds natural on uneven terrain.
- Prevent leaks/sinking and tree-on-water overlap.

## Placement workflow

1. **Resolve terrain target**
   - Use `ProceduralMeshGround` as the terrain source.
   - Require valid `MeshFilter` + `MeshCollider`.

2. **Pick candidate centers**
   - Sample XZ positions within terrain bounds (with edge padding).
   - Enforce pond-to-pond spacing so ponds do not overlap.

3. **Carve local basin (required)**
   - Carve only local circular patches under each candidate pond.
   - Use smooth falloff from center to shore.
   - Do not flatten the entire terrain.

4. **Place water surface**
   - Use smooth circular pond surface geometry.
   - Place at `groundHitY + waterLift`.
   - Start with `waterLift = 0.22`; if still sinking visually, raise to `0.30..0.45`.

5. **Material assignment**
   - Assign `Assets/Materials/WaterPond.mat`.
   - Keep water readable (blue-green tint, high smoothness, transparent-style if supported).

6. **Vegetation exclusion**
   - Build exclusion circles from pond centers/radii.
   - Exclude trees from `radius + shorelineMargin` where `shorelineMargin` is typically `4..8`.
   - Reposition any existing trees that violate exclusion.

7. **Post-carve terrain visual refresh (required)**
   - Because basin carving changes local vertex heights, refresh terrain shading assets after carving:
     - rebake `GroundHeightMap.png`
     - rebake and reassign `GroundNormalMap.png` on `ProceduralMeshGround` material
     - rebake `GroundGrassMask.png` if procedural grass is used
   - regenerate procedural grass placement so shoreline areas respect updated slope/height and water exclusion
   - enforce water-edge exclusion in grass pass:
     - no grass inside `pondRadius + shorelineMargin`
   - do not finish pond operation until refresh + grass update are complete

## Recommended defaults

- Pond count: few/medium (`4..15`) unless requested otherwise.
- Large pond radius: `12..22`.
- Extra separation between ponds: `10..18`.
- Shoreline margin for tree exclusion: `4..8`.

## Validation checklist

- Every pond is visible (not buried in ground).
- Shorelines are smooth, not rough-edged.
- Terrain remains uneven globally (only local basin edits).
- No tree roots inside pond exclusion zones.
- Ponds remain within terrain bounds.
- Terrain normal map is rebaked from post-carve mesh and reassigned.
- Grass near ponds is regenerated and excluded from pond/water zones.
