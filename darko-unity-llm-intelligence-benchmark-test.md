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
| **Layer discipline** | Executes terrain → water → props → lighting in order; reruns visual refresh when geometry changes. |
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
3) When terrain geometry changes (water system carving), rebake ALL terrain textures (height, normal, tint, splat, composites), reassign materials.
4) Placement: use world-placement-delegator rules—downward raycast to ProceduralMeshGround, no penetration, overlap budgets.
5) Scale is ALWAYS (1,1,1). Map size controlled by localSizeX/Z. Noise uses world-space coordinates.
6) Output: per-layer summary (counts, key asset paths, material slots assigned).
7) FRESH GENERATION: Every texture, material, and mesh must be generated from scratch by the agent's own code. Do NOT reuse existing asset files from the project. If a file already exists at the target path, delete it first and regenerate. The benchmark tests whether the agent can CREATE these assets — not whether it can find and assign pre-made ones. Reusing existing files produces identical worlds across runs and invalidates the benchmark.
```

Then attach or point the model at the **Prompts** listed below—those files are the **authoritative copy/paste prompts** for each subsystem.

---

## Layer plan: generate the same world (core benchmark)

Execute **one layer at a time**. Do not skip the **visual refresh handshake** after any step that edits terrain vertices.

### Layer 1 — Terrain + surface PBR

- **Primary prompt:** [context/terrain/generate-realistic-terrain.md](context/terrain/generate-realistic-terrain.md)  
- **Deliverables:** `ProceduralMeshGround` (mesh built with scale 1,1,1), `MeshCollider`, baked height/normal textures, height-based terrain tint (snow/rock/river zones), `RiverPath` metadata object, assigned `Ground.mat`.
- **Scoring:** 100-point rubric covering structural quality (50), technical quality (30), and visual quality (20). Map size multiplier applies (0.5x–1.3x).

### Layer 2 — Terrain Splat Mapping (dynamic multi-texture blending)

- **Primary prompt:** [context/terrain/generate-terrain-splat-map.md](context/terrain/generate-terrain-splat-map.md)  
- **Requires:** Layer 1 completed (ProceduralMeshGround with varied topography + RiverPath).
- **Deliverables:** `TerrainSplatMap.png` (RGBA weight map), 12 tileable PBR textures (4 zones x albedo/normal/mask), composite baked albedo + normal + mask on `Ground.mat`.
- **Scoring:** 100-point rubric covering splat map quality (30), tileable texture quality (30), composite bake quality (25), technical quality (15).

### Layer 3 — Water System (rivers, lakes, ponds + terrain refresh)

- **Primary prompt:** [context/terrain/generate-water-system.md](context/terrain/generate-water-system.md)  
- **Requires:** Layer 1 + Layer 2 completed. RiverPath must exist.
- **Why before objects:** Water carves terrain geometry. All objects must be placed on FINAL terrain — after all carving is done. Placing objects first, then carving, would leave floating trees and buried rocks.
- **Deliverables:** `RiverWater` mesh following RiverPath, `Lake_N` irregular water bodies in low points, `Pond_N` small water bodies in plains, `WaterExclusionZones` for downstream layers.
- **Mandatory:** water surface placement, exclusion zone data.
- **Scoring:** 100-point rubric covering river (25), lakes (25), ponds (15), terrain refresh (25), technical (10).

### Layer 4 — Props: trees, rocks

- **Placement prompt:** [context/generate-props.md](context/generate-props.md)  
- **Prop definitions:**
  - [context/props/generate-tree.md](context/props/generate-tree.md) — Procedural recursive tree (seeded, unique per instance)
  - [context/props/generate-rock.md](context/props/generate-rock.md) — Rocks: singles, lines, mounds
- **Requires:** Layer 1 + Layer 3 completed. WaterExclusionZones must exist.
- **Deliverables:** `Trees`, `Rocks` parent containers. All props raycast-grounded, respecting height/slope/water exclusion zones. Baked Lit materials. Natural distribution with clustering.
- **Key rules:** Each prop uses seeded random for uniqueness. Placement respects terrain zones — no props on peaks, cliffs, or in water. Natural distribution with noise-based clustering and clearings.
- **Scoring:** 100-point rubric covering placement quality (40), prop quality (30), technical quality (20), visual quality (10).

### Layer 5 — Lighting & post-processing

- **Primary prompt:** [context/generate-lighting-post-processing.md](context/generate-lighting-post-processing.md)  
- **Requires:** Layers 1-4 completed. URP active. Camera in scene.
- **Deliverables:** Directional light (Sun) with warm color and soft shadows, gradient ambient lighting, post-processing Volume with ACES tonemapping, bloom, color grading, vignette, SSAO. Optional distance fog for atmosphere.
- **Key rules:** Does NOT modify terrain, water, or props. Lighting and rendering settings only.
- **Scoring:** 100-point rubric covering lighting quality (35), post-processing quality (35), environment quality (20), technical quality (10).

---

## Core prompt library (all `.md` files in this benchmark)

These files are the **source of truth** for copy/paste prompts. The benchmark is invalid if the runner does not keep them in sync with the repo.


| File                                                                                       | Role                                          |
| ------------------------------------------------------------------------------------------ | --------------------------------------------- |
| [context/terrain/generate-realistic-terrain.md](context/terrain/generate-realistic-terrain.md)             | Realistic terrain: mountains, hills, river    |
| [context/terrain/generate-terrain-splat-map.md](context/terrain/generate-terrain-splat-map.md)             | Terrain splat mapping: multi-texture blending |
| [context/terrain/generate-water-system.md](context/terrain/generate-water-system.md)                       | Water system: rivers, lakes, ponds            |
| [context/generate-props.md](context/generate-props.md)                                     | Props placement: trees, rocks                 |
| [context/props/generate-tree.md](context/props/generate-tree.md)                           | Procedural realistic tree (seeded, recursive branching) |
| [context/props/generate-rock.md](context/props/generate-rock.md)                           | Rocks: singles, lines, mounds                 |
| [context/generate-lighting-post-processing.md](context/generate-lighting-post-processing.md) | Lighting, shadows, post-processing            |



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