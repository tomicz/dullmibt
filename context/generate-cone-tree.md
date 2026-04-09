# Prompt: Generate Cone Tree (Procedural Mesh + Snapped Trunk Base)

Use this prompt with a Unity-capable coding agent (Unity MCP) to generate cone-style trees where each trunk base snaps to terrain surface (no penetration).

## Purpose

This prompt defines how to create the cone-tree type used in this project:
- procedural cone canopy (custom mesh, not primitive cone),
- darker green canopy material,
- cylinder trunk with brown material,
- placement grounded to `ProceduralMeshGround` via raycast.

## Copy/Paste Prompt

```text
In the active Unity scene, create cone-style trees with grounded trunk bases.

Target surface:
- Surface object name: "ProceduralMeshGround"
- Surface must have MeshCollider

Tree definition:
1. Create canopy GameObject named "ProceduralCone" (or "ConeTree_<index>" for batches).
2. Add MeshFilter + MeshRenderer + MeshCollider.
3. Procedurally build a cone mesh:
   - radius = 1.0
   - height = 2.0
   - segments = 32
   - include side triangles and base cap triangles
   - RecalculateNormals + RecalculateBounds
4. Create/load canopy material `Assets/Materials/ConeDark.mat`:
   - URP Lit fallback Standard
   - dark green color (example RGB: 0.08, 0.25, 0.08)
5. Assign `ConeDark` material to canopy renderer.

Trunk definition (match primitive-tree logic):
6. Add child cylinder named "Cone_Trunk":
   - use variable `trunkHeight` and `trunkRadius`
   - localPosition = `(0, trunkHeight, 0)`
   - localScale = `(trunkRadius, trunkHeight, trunkRadius)`
   - assign `Assets/Materials/TrunkBrown.mat` (create if missing)

Canopy placement rule (critical):
7. Add cone canopy as a sibling child under the same tree root:
   - canopy localPosition = `(0, trunkHeight * 2, 0)`
   - canopy mesh base is at local y=0
   - this guarantees cone base sits directly on trunk top (no separation)

Ground snapping (required):
8. For each tree spawn candidate `(x,z)`, raycast downward onto `ProceduralMeshGround`:
   - ray origin: `(x, bounds.max.y + 500, z)`
   - direction: down
9. Set tree root world position to `hit.point`.
10. Keep trunk/canopy offsets as defined above so:
   - trunk bottom starts at tree root
   - cone base sits on trunk top
11. Optional: slight slope alignment using hit normal (small blend only).

Batch rules (if spawning many):
12. Keep non-overlap spacing in XZ:
   - store placed points
   - reject candidates with distance below radius sum + margin
13. Retry random candidates with a capped attempt count.
14. Report created/skipped counts and confirm raycast-grounded placement.

Constraints:
- Do NOT use Unity Terrain / TerrainData.
- Do NOT place trunk below surface.
- Do NOT leave objects floating.
- Do NOT separate cone from trunk; canopy base must touch trunk top.
- Every placed tree must come from a successful surface raycast hit.
```

## Expected Result

- Cone-style tree(s) with dark cone canopy and brown trunk.
- Trunk base snapped to top of procedural mesh surface.
- Cone canopy attached to trunk (same primitive-tree-style hierarchy/offset relationship).
- Stable results for single or batch generation.
