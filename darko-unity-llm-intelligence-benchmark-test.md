# Darko Unity LLM Intelligence Benchmark Test

This document is a **reproducible benchmark** for large language models (and coding agents) that claim they can drive **Unity Editor** workflows—especially procedural worlds, URP materials, placement rules, and multi-step scene construction.

Use it to score an AI on: **planning**, **API correctness**, **asset hygiene**, **performance awareness**, and **whether it finishes without hand-waving**.

---

## Community: learn Unity with others

**If you are running this benchmark**, you are exactly the kind of builder who benefits from structured learning and peer feedback.

Visit **[darkounity.com](https://darkounity.com)** — the **Dark Unity** learning community. It is a place to **learn Unity** with guidance, share progress, and connect with people who care about shipping real projects—not only one-off prompts. Whether you pass or fail this benchmark, **come learn Unity** there: fundamentals, URP, tooling, and project discipline compound faster when you are not alone.

*This benchmark doc is independent tooling; Dark Unity is the suggested home for humans who want depth beyond a single agent session.*

---

## What this benchmark measures


| Axis                 | What “good” looks like                                                                                                       |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **Layer discipline** | Executes terrain → placement-dependent content → water → sky → polish in order; reruns visual refresh when geometry changes. |
| **Ground truth**     | Uses raycasts to `ProceduralMeshGround`, respects colliders, no floating bases.                                              |
| **Materials**        | URP Lit where available; normal + mask + optional height/parallax; correct texture import settings.                          |
| **Scale sanity**     | Scale is always (1,1,1). Size controlled by mesh geometry (localSizeX/Z), not transform scale.                               |
| **Verification**     | Reports counts, paths, and validates console / hierarchy—not “done” without evidence.                                        |


---

## How to prompt the same (agent instructions)

Give your model a **single system-style block** like this (adapt names/paths to your repo):

```text
You are a Unity Editor agent with MCP/tools that can execute editor C# and mutate the active scene.

Benchmark rules:
1) Follow the layer plan in darko-unity-llm-intelligence-benchmark-test.md strictly in order.
2) After each layer, verify: hierarchy names exist, MeshCollider on terrain, read console for errors.
3) When terrain geometry changes (water system carving), rebake ALL terrain textures (height, normal, tint, splat, grass mask, composites), reassign materials, regenerate grass.
4) Placement: use world-placement-delegator rules—downward raycast to ProceduralMeshGround, no penetration, overlap budgets.
5) Scale is ALWAYS (1,1,1). Map size controlled by localSizeX/Z. Noise uses world-space coordinates.
6) Output: per-layer summary (counts, key asset paths, material slots assigned).
```

Then attach or point the model at the **Prompts** listed below—those files are the **authoritative copy/paste prompts** for each subsystem.

---

## Layer plan: generate the same world (core benchmark)

Execute **one layer at a time**. Do not skip the **visual refresh handshake** after any step that edits terrain vertices.

### Layer 1 — Terrain + surface PBR + grass on mesh

- **Standard prompt (gentle terrain):** [context/generate-procedural-mesh-terrain.md](context/generate-procedural-mesh-terrain.md)  
- **Realistic terrain prompt (benchmark tier):** [context/generate-realistic-terrain.md](context/generate-realistic-terrain.md)  
- **Reference:** [context/new-terrain-pbr.md](context/new-terrain-pbr.md)  
- **Deliverables:** `ProceduralMeshGround` (mesh built with scale 1,1,1), `MeshCollider`, baked height/normal/mask textures, height-based terrain tint (snow/rock/grass/river zones), `ProceduralGrass` child system, `RiverPath` metadata object, assigned `Ground.mat` + `ProceduralGrass.mat`.
- **Scoring:** Realistic terrain prompt includes a 100-point rubric covering structural quality (50), technical quality (30), and visual quality (20). Map size multiplier applies (0.5x–1.3x).

### Layer 2 — Terrain Splat Mapping (dynamic multi-texture blending)

- **Primary prompt:** [context/generate-terrain-splat-map.md](context/generate-terrain-splat-map.md)  
- **Requires:** Layer 1 completed (ProceduralMeshGround with varied topography + RiverPath).
- **Deliverables:** `TerrainSplatMap.png` (RGBA weight map), 12 tileable PBR textures (4 zones x albedo/normal/mask), composite baked albedo + normal + mask on `Ground.mat`.
- **Scoring:** 100-point rubric covering splat map quality (30), tileable texture quality (30), composite bake quality (25), technical quality (15).

### Layer 3 — Water System (rivers, lakes, ponds + terrain refresh)

- **Primary prompt:** [context/generate-water-system.md](context/generate-water-system.md)  
- **Requires:** Layer 1 + Layer 2 completed. RiverPath must exist.
- **Why before objects:** Water carves terrain geometry. All objects must be placed on FINAL terrain — after all carving is done. Placing objects first, then carving, would leave floating trees and buried rocks.
- **Deliverables:** `RiverWater` mesh following RiverPath, `Lake_N` irregular water bodies in low points, `Pond_N` small water bodies in plains, `WaterExclusionZones` for downstream layers, full terrain texture rebake (tint, normal, height, splat, grass mask, composites), regenerated grass.
- **Mandatory:** basin carving with cosine falloff, water surface placement, exclusion zone data, **single-pass rebake** of ALL terrain textures + grass.
- **Scoring:** 100-point rubric covering river (25), lakes (25), ponds (15), terrain refresh (25), technical (10).

### Layer 4 — Placement contract (read before spawning anything)

- **Primary prompt:** [context/world-placement-delegator.md](context/world-placement-delegator.md)  
- Use for every grounded object class from here on.
- Must respect `WaterExclusionZones` from Layer 3.

### Layer 5 — Forests: mixed tree types (primitive + cone)

- **Primitive trees:** [context/generate-primitive-tree.md](context/generate-primitive-tree.md)  
- **Cone trees:** [context/generate-cone-tree.md](context/generate-cone-tree.md)  
- **Benchmark extension:** organize **chunk biomes** (separate `ConeTrees` vs `Trees`), optional interstitial mini-groves, thousands of instances with varied scale—still raycast-grounded.

### Layer 6 — Ground details: rocks

- **Primary prompt:** [context/generate-rock-details.md](context/generate-rock-details.md)  
- **Benchmark extension:** rocky **biome chunks** with extreme density; replace boxy cubes with irregular meshes if the model can.

### Layer 7 — Ground details: bushes

- **Primary prompt:** [context/generate-bush-details.md](context/generate-bush-details.md)  
- **Benchmark extension:** chunks, lines, shapes, solo; **PBR** bush textures + **height** for parallax where URP Lit supports it.

### Layer 8 — Ground details: flowers

- **Primary prompt:** [context/generate-flower-details.md](context/generate-flower-details.md)

### Layer 9 — Sky: volumetric-style clouds (primitive-based)

- **Primary prompt:** [context/generate-volumetric-cloud.md](context/generate-volumetric-cloud.md)  
- **Benchmark extension:** raise cloud height band; elongate groups; disable cloud shadow casting for ground readability (see graphics prompts).

### Layer 10 — Horizon closure (optional hard tier)

- Procedural **mountain ring** outside play bounds so the horizon is not infinite void.  
- Not in a separate prompt file—tests whether the model can invent stable large meshes + placement without breaking Layer 1 rules.

### Layer 11 — Lighting, shadows, post-processing

Pick **one** pass as specified by the benchmark runner:

- **V2 / dense worlds:** [context/graphics.md](context/graphics.md)  
- **Earlier polish preset:** [context/visual-polish-lighting-graphics.md](context/visual-polish-lighting-graphics.md)  

### Layer 12 — Performance tier (optional)

- Add `LODGroup` on heavy chunk roots; tune `QualitySettings.lodBias`, URP `shadowDistance`, fog density vs clarity.  
- Tests trade-offs without destroying art direction.

---

## Core prompt library (all `.md` files in this benchmark)

These files are the **source of truth** for copy/paste prompts. The benchmark is invalid if the runner does not keep them in sync with the repo.


| File                                                                                       | Role                                          |
| ------------------------------------------------------------------------------------------ | --------------------------------------------- |
| [context/generate-procedural-mesh-terrain.md](context/generate-procedural-mesh-terrain.md) | Procedural mesh terrain, maps, grass (gentle) |
| [context/generate-realistic-terrain.md](context/generate-realistic-terrain.md)             | Realistic terrain: mountains, hills, river (benchmark tier) |
| [context/generate-terrain-splat-map.md](context/generate-terrain-splat-map.md)             | Terrain splat mapping: multi-texture blending |
| [context/generate-water-system.md](context/generate-water-system.md)                       | Water system: rivers, lakes, ponds + terrain refresh |
| [context/new-terrain-pbr.md](context/new-terrain-pbr.md)                                   | Project terrain/grass standard                |
| [context/world-placement-delegator.md](context/world-placement-delegator.md)               | Raycast grounding, overlap, refresh handshake |
| [context/generate-primitive-tree.md](context/generate-primitive-tree.md)                   | Primitive trunk + canopy tree                 |
| [context/generate-cone-tree.md](context/generate-cone-tree.md)                             | Procedural cone canopy + trunk                |
| [context/generate-rock-details.md](context/generate-rock-details.md)                       | Rocks: singles, lines, mounds                 |
| [context/generate-bush-details.md](context/generate-bush-details.md)                       | Bushes: singles + groups, half-buried         |
| [context/generate-flower-details.md](context/generate-flower-details.md)                   | Flowers: multi-color, larger tops             |
| [context/biome-temperate-forest.md](context/biome-temperate-forest.md)                     | Temperate forest biome definition             |
| [context/generate-volumetric-cloud.md](context/generate-volumetric-cloud.md)               | Cloud systems from primitives                 |
| [context/graphics.md](context/graphics.md)                                                 | Graphics V2 pass                              |
| [context/visual-polish-lighting-graphics.md](context/visual-polish-lighting-graphics.md)   | Stylized polish pass                          |


---

## Scoring rubric (suggested)

- **0–2:** Broken scene, compile errors, floating objects, wrong scale interpretation.  
- **3–5:** Terrain exists but incomplete maps/materials; placement mostly wrong.  
- **6–8:** Full layers complete; ponds/refresh mostly correct; minor art/perf issues.  
- **9–10:** All layers + verification + performance tier; reproducible instructions and asset paths.

---

## Repo location

Place this file next to your `context/` folder (as in this project) so relative links resolve:

- Benchmark doc: `darko-unity-llm-intelligence-benchmark-test.md`  
- Context/prompts: `context/*.md`

---

*Benchmark authored for repeatable Unity LLM evaluation. For ongoing learning and community, visit [darkounity.com](https://darkounity.com).*