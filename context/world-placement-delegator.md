# Prompt: World Placement Delegator

Use this prompt when you want the agent to place generated objects correctly in the world (especially on procedural mesh terrain) without floating or clipping.

## Purpose

This prompt is the single source of truth for **where and how objects are positioned** in the scene.

It ensures:

- objects are placed on top of terrain/mesh surfaces,
- object bases do not penetrate the ground,
- spacing rules are respected (no overlap),
- placement remains correct even when terrain is uneven.

Object-type behavior:

- **Cone trees** are ground-anchored objects and must use raycast grounding + base snap.
- **Clouds** are sky objects and must be positioned relative to ground height (above ground), not snapped to surface.

## Copy/Paste Prompt

```text
Act as the World Placement Delegator for this Unity scene.

Goal:
- Place objects so they sit correctly on the environment surface.
- Never let object bases spawn inside the terrain/mesh.

Placement Rules:
1. Always resolve surface first
   - Target surface object name: "ProceduralMeshGround"
   - Require MeshCollider on the target surface.
   - If collider is missing, add/fix it before placing anything.

2. Use raycast-based grounding
   - For each object candidate `(x,z)`, cast a ray downward from above bounds:
     - origin: `(x, bounds.max.y + margin, z)` where margin is large enough (e.g. +500)
     - direction: `Vector3.down`
   - Use the ray hit point as the world root position.

3. Base-on-surface rule (no penetration)
   - Place object root at ray hit point.
   - Build child geometry with local offsets so base touches the root:
     - Example cylinder trunk of Unity primitive:
       - localPosition.y = trunkHeight
       - localScale.y = trunkHeight
       - This makes cylinder bottom align with root at y=0 in local space.
   - Never subtract extra Y unless explicitly asked.

4. Optional slope alignment
   - If terrain is uneven, align object up-vector partly to hit normal:
     - `align = FromToRotation(Vector3.up, hit.normal)`
     - Apply partial blend (e.g. Slerp identity->align with 0.2-0.4) for natural tilt.
   - Keep extreme tilts avoided unless user asks.

5. Non-overlap rule
   - Track placed XZ positions and placement radius per object.
   - Reject candidates where distance < (radiusA + radiusB + safetyMargin).
   - Retry random candidate sampling up to a max attempt budget.

6. Bounds-aware sampling
   - Sample candidate XZ inside the target surface collider bounds.
   - Do not place outside terrain extents unless requested.

7. Deterministic reporting
   - Return counts:
     - created
     - skipped (if no valid placement found)
   - Confirm grounding method used:
     - "roots placed via surface raycast hits"

8. Type-specific placement policies
   - Cone trees:
     - root position = surface raycast hit point
     - trunk base must sit at root (no penetration, no floating)
     - canopy must remain attached to trunk (no visible separation)
   - Clouds:
     - sample XZ within terrain bounds
     - raycast to get ground Y at each cloud XZ
     - final cloud Y = groundY + baseCloudHeight + randomVariation
     - do not snap cloud root to terrain (clouds stay in sky layer)
     - default recommendation: baseCloudHeight = 10, variation = +/-2 (or as user specifies)

9. Count and adjustment behavior
   - If user asks to change count, add/remove instances to match exact target count.
   - If user asks to move objects up/down, apply delta consistently to all targeted instances.
   - Prefer deterministic naming and stable parent containers:
     - cone trees under `ConeTrees`
     - clouds under `VolumetricClouds`

10. Visual-regeneration handshake (required after terrain-affecting operations)
   - If any workflow step changed terrain geometry/vertex heights (including basin carving, smoothing, or mesh regeneration), force a visual refresh in the same run:
     - rebake terrain textures from current mesh data (`GroundHeightMap`, `GroundNormalMap`, and `GroundGrassMask` if used)
     - reassign updated normal map to `ProceduralMeshGround` material (`_BumpMap`, `_BumpScale > 0`)
     - regenerate/reposition procedural grass instances to match updated terrain and masks
     - ensure grass material still has its own normal map assigned for visible blade depth
   - Do not report success until this refresh is complete.

Validation Checklist (must pass):
- Every placed object has a successful surface raycast hit.
- No object base starts below surface at its placement point.
- MeshCollider is assigned to generated surface mesh.
- Overlap constraint respected for all placed instances.
- Cone trees: cone canopy is attached to trunk top.
- Clouds: inside terrain XZ bounds and at requested above-ground height band.
- If terrain changed: terrain normal map rebaked and reassigned.
- If grass exists: grass regenerated for updated terrain and grass material normal map remains assigned.
```

## Suggested Usage

- Use this prompt before spawning forests, rocks, props, NPC markers, or pickups.
- Combine with generation prompts:
  - First generate terrain/mesh.
  - Then run World Placement Delegator for spawn and grounding.
  - If terrain geometry changed at any point, run the visual-regeneration handshake before final confirmation.

## Recommended Name

`world-placement-delegator.md` is recommended because it is explicit and easy to remember.