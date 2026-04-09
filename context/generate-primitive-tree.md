# Prompt: Generate a Primitive Tree in Unity

Use this prompt with a Unity-capable coding agent (with Unity MCP tools) to generate a simple tree from primitives.

## Copy/Paste Prompt

```text
In the active Unity scene, create a single tree GameObject made from Unity primitives.

Requirements:
1. Create a parent GameObject named "Tree" at position (2, 0, 2).
2. Create a child cylinder named "Tree_Trunk" to represent the trunk:
   - Primitive type: Cylinder
   - Local position: (0, 1, 0)
   - Local scale: (0.4, 1.0, 0.4)
3. Create a child sphere named "Tree_Canopy" to represent leaves:
   - Primitive type: Sphere
   - Local position: (0, 2.6, 0)
   - Local scale: (1.6, 1.6, 1.6)
4. Create materials under Assets/Materials:
   - LeavesGreen.mat
   - TrunkBrown.mat
5. Set material colors:
   - LeavesGreen: RGBA (0.2, 0.7, 0.2, 1.0)
   - TrunkBrown: RGBA (0.45, 0.28, 0.12, 1.0)
6. Assign LeavesGreen to Tree_Canopy and TrunkBrown to Tree_Trunk.
7. Verify final hierarchy and transforms, then report completion.

PBR-style upgrade (required for this project):
8. Create/use procedural texture set for trunk and canopy under `Assets/Textures/Procedural/`:
   - `TreeBarkAlbedo.png`, `TreeBarkNormal.png`, `TreeBarkMaskMap.png`
   - `TreeLeavesAlbedo.png`, `TreeLeavesNormal.png`, `TreeLeavesMaskMap.png`
9. Update tree materials for URP Lit (fallback Standard):
   - `TrunkBrown.mat` uses bark albedo + normal + mask map
   - `LeavesGreen.mat` uses leaf albedo + normal + mask map
   - keep metallic near zero and smoothness medium-low
10. Keep canopy attached to trunk top (no separation):
   - trunk local anchor rule: `Tree_Trunk.localPosition.y = Tree_Trunk.localScale.y`
   - canopy should touch/overlap trunk top slightly to avoid visible gaps

Constraints:
- Do not create extra trees or duplicate trunk/canopy objects.
- Keep naming exactly as specified.
- If Assets/Materials does not exist, create it first.
```

## Expected Result

- A single `Tree` object in the scene.
- A brown cylinder trunk and green sphere canopy with PBR-style texture response.
- Reusable materials saved in `Assets/Materials`.

