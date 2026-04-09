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
| **Scale sanity**     | Honors `requestedScale` by **rebuilding mesh**, not only scaling transform.                                                  |
| **Verification**     | Reports counts, paths, and validates console / hierarchy—not “done” without evidence.                                        |


---

## How to prompt the same (agent instructions)

Give your model a **single system-style block** like this (adapt names/paths to your repo):

```text
You are a Unity Editor agent with MCP/tools that can execute editor C# and mutate the active scene.

Benchmark rules:
1) Follow the layer plan in darko-unity-llm-intelligence-benchmark-test.md strictly in order.
2) After each layer, verify: hierarchy names exist, MeshCollider on terrain, read console for errors.
3) When terrain geometry changes (ponds, carving), rebake GroundHeightMap, GroundNormalMap, GroundGrassMask (if used), reassign materials, regenerate grass.
4) Placement: use world-placement-delegator rules—downward raycast to ProceduralMeshGround, no penetration, overlap budgets.
5) World scale default: requestedScale = (10, 1, 10) unless the benchmark runner specifies otherwise.
6) Output: per-layer summary (counts, key asset paths, material slots assigned).
```

Then attach or point the model at the **Prompts** listed below—those files are the **authoritative copy/paste prompts** for each subsystem.

---

## Layer plan: generate the same world (core benchmark)

Execute **one layer at a time**. Do not skip the **visual refresh handshake** after any step that edits terrain vertices.

### Layer 1 — Terrain + surface PBR + grass on mesh

- **Primary prompt:** [Prompts/generate-procedural-mesh-terrain.md](Prompts/generate-procedural-mesh-terrain.md)  
- **Reference:** [Prompts/new-terrain-pbr.md](Prompts/new-terrain-pbr.md)  
- **Deliverables:** `ProceduralMeshGround` (mesh rebuilt at scale), `MeshCollider`, baked height/normal/mask textures, hybrid height tint, `ProceduralGrass` child system, assigned `Ground.mat` + `ProceduralGrass.mat`.

### Layer 2 — Placement contract (read before spawning anything)

- **Primary prompt:** [Prompts/world-placement-delegator.md](Prompts/world-placement-delegator.md)  
- Use for every grounded object class from here on.

### Layer 3 — Forests: mixed tree types (primitive + cone)

- **Primitive trees:** [Prompts/generate-primitive-tree.md](Prompts/generate-primitive-tree.md)  
- **Cone trees:** [Prompts/generate-cone-tree.md](Prompts/generate-cone-tree.md)  
- **Benchmark extension:** organize **chunk biomes** (separate `ConeTrees` vs `Trees`), optional interstitial mini-groves, thousands of instances with varied scale—still raycast-grounded.

### Layer 4 — Ground details: rocks

- **Primary prompt:** [Prompts/generate-rock-details.md](Prompts/generate-rock-details.md)  
- **Benchmark extension:** rocky **biome chunks** with extreme density; replace boxy cubes with irregular meshes if the model can.

### Layer 5 — Ground details: bushes

- **Primary prompt:** [Prompts/generate-bush-details.md](Prompts/generate-bush-details.md)  
- **Benchmark extension:** chunks, lines, shapes, solo; **PBR** bush textures + **height** for parallax where URP Lit supports it.

### Layer 6 — Ground details: flowers

- **Primary prompt:** [Prompts/generate-flower-details.md](Prompts/generate-flower-details.md)

### Layer 7 — Ponds (definition + rules + terrain refresh)

- **What a pond is:** [Prompts/pond-definition.md](Prompts/pond-definition.md)  
- **How to place and post-process:** [Prompts/pond-placement-rules.md](Prompts/pond-placement-rules.md)  
- **Mandatory:** local basin carve, water surface lift, exclusion zones, **rebake** terrain maps + **regenerate** grass.

### Layer 8 — Sky: volumetric-style clouds (primitive-based)

- **Primary prompt:** [Prompts/generate-volumetric-cloud.md](Prompts/generate-volumetric-cloud.md)  
- **Benchmark extension:** raise cloud height band; elongate groups; disable cloud shadow casting for ground readability (see graphics prompts).

### Layer 9 — Horizon closure (optional hard tier)

- Procedural **mountain ring** outside play bounds so the horizon is not infinite void.  
- Not in a separate prompt file—tests whether the model can invent stable large meshes + placement without breaking Layer 1 rules.

### Layer 10 — Lighting, shadows, post-processing

Pick **one** pass as specified by the benchmark runner:

- **V2 / dense worlds:** [Prompts/graphics.md](Prompts/graphics.md)  
- **Earlier polish preset:** [Prompts/visual-polish-lighting-graphics.md](Prompts/visual-polish-lighting-graphics.md)  
- **Shadow clarity subset:** [Prompts/graphi.md](Prompts/graphi.md)

### Layer 11 — Performance tier (optional)

- Add `LODGroup` on heavy chunk roots; tune `QualitySettings.lodBias`, URP `shadowDistance`, fog density vs clarity.  
- Tests trade-offs without destroying art direction.

---

## Core prompt library (all `.md` files in this benchmark)

These files are the **source of truth** for copy/paste prompts. The benchmark is invalid if the runner does not keep them in sync with the repo.


| File                                                                                       | Role                                          |
| ------------------------------------------------------------------------------------------ | --------------------------------------------- |
| [Prompts/generate-procedural-mesh-terrain.md](Prompts/generate-procedural-mesh-terrain.md) | Procedural mesh terrain, maps, grass          |
| [Prompts/new-terrain-pbr.md](Prompts/new-terrain-pbr.md)                                   | Project terrain/grass standard                |
| [Prompts/world-placement-delegator.md](Prompts/world-placement-delegator.md)               | Raycast grounding, overlap, refresh handshake |
| [Prompts/generate-primitive-tree.md](Prompts/generate-primitive-tree.md)                   | Primitive trunk + canopy tree                 |
| [Prompts/generate-cone-tree.md](Prompts/generate-cone-tree.md)                             | Procedural cone canopy + trunk                |
| [Prompts/generate-rock-details.md](Prompts/generate-rock-details.md)                       | Rocks: singles, lines, mounds                 |
| [Prompts/generate-bush-details.md](Prompts/generate-bush-details.md)                       | Bushes: singles + groups, half-buried         |
| [Prompts/generate-flower-details.md](Prompts/generate-flower-details.md)                   | Flowers: multi-color, larger tops             |
| [Prompts/pond-definition.md](Prompts/pond-definition.md)                                   | Pond semantics                                |
| [Prompts/pond-placement-rules.md](Prompts/pond-placement-rules.md)                         | Pond placement + post-carve refresh           |
| [Prompts/generate-volumetric-cloud.md](Prompts/generate-volumetric-cloud.md)               | Cloud systems from primitives                 |
| [Prompts/graphics.md](Prompts/graphics.md)                                                 | Graphics V2 pass                              |
| [Prompts/visual-polish-lighting-graphics.md](Prompts/visual-polish-lighting-graphics.md)   | Stylized polish pass                          |
| [Prompts/graphi.md](Prompts/graphi.md)                                                     | Shadow + clarity pass                         |


---

## Scoring rubric (suggested)

- **0–2:** Broken scene, compile errors, floating objects, wrong scale interpretation.  
- **3–5:** Terrain exists but incomplete maps/materials; placement mostly wrong.  
- **6–8:** Full layers complete; ponds/refresh mostly correct; minor art/perf issues.  
- **9–10:** All layers + verification + performance tier; reproducible instructions and asset paths.

---

## Repo location

Place this file next to your `Prompts/` folder (as in this project) so relative links resolve:

- Benchmark doc: `darko-unity-llm-intelligence-benchmark-test.md`  
- Prompts: `Prompts/*.md`

---

*Benchmark authored for repeatable Unity LLM evaluation. For ongoing learning and community, visit [darkounity.com](https://darkounity.com).*