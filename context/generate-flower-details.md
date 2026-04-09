# Prompt: Generate Flower Details (Color Mix + Larger Flower Tops)

Use this prompt with a Unity-capable coding agent (Unity MCP) to generate colorful flowers across `ProceduralMeshGround`.

## Purpose

This prompt defines flower generation standards for this world:

- color variety (blue, red, yellow, orange),
- broad map distribution,
- grounded placement on terrain,
- slightly larger flower top for better readability at camera distance.

## Copy/Paste Prompt

```text
In the active Unity scene, generate procedural flowers on `ProceduralMeshGround`.

Requirements:
1. Ensure target terrain:
   - object: `ProceduralMeshGround`
   - components: MeshFilter + MeshCollider
2. Use/refresh detail container:
   - parent: `GroundDetails/Flowers`
3. Create/use materials:
   - `Assets/Materials/FlowerBlue.mat`
   - `Assets/Materials/FlowerRed.mat`
   - `Assets/Materials/FlowerYellow.mat`
   - `Assets/Materials/FlowerOrange.mat`
   - `Assets/Materials/FlowerStemGreen.mat`
4. Place flowers via downward raycast to terrain.
5. Avoid very steep slopes.

Flower construction:
6. Build each flower from primitives:
   - stem: 1 cylinder
   - top: 4 petals + 1 center (sphere-based)
7. Color assignment:
   - randomly distribute blue, red, yellow, orange petals across map
8. Top-size rule (required):
   - make top parts slightly bigger than default:
     - petals: ~15%-25% larger than baseline
     - center: ~20%-30% larger than baseline
   - raise top parts slightly in local Y so they still read cleanly above stem
9. Remove colliders from flower parts (visual detail only).
10. Report:
   - total flower count
   - per-color counts

Constraints:
- Do not place flowers floating above terrain.
- Keep flowers spread across the map footprint.
- Keep naming deterministic (`Flower_<index>`, `Petal_<index>`, `Center`, `Stem`).
```

## Expected Result

- Flowers distributed across map with all requested colors.
- No floating flowers.
- Slightly larger flower tops improve visibility and style consistency.
