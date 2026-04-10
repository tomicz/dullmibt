# Darko Unity LLM Intelligence Benchmark Test

A reproducible benchmark for testing whether AI coding agents can generate procedural 3D worlds inside the Unity Editor — by driving the editor directly through [Unity MCP](https://github.com/niceholmgren/mcp-for-unity), or by writing C# scripts the user runs in the editor.

## What this tests

Can an AI agent build a realistic landscape with terrain, water, and props by generating and executing C# code in the Unity Editor? The benchmark scores planning, API correctness, asset generation, and whether the agent actually finishes.

## Requirements

- **Unity 2022.3+** with **URP** (Universal Render Pipeline)
- An AI coding agent (Claude Code, Cursor, Windsurf, Copilot, etc.)
- **Unity MCP server** (recommended) — [mcp-for-unity](https://github.com/niceholmgren/mcp-for-unity)

## How agents should work

There are two valid execution paths:

### Path A — Unity MCP (recommended)

The agent executes C# code directly in the Unity Editor through Unity MCP's `execute_code` tool. No script files are written — code runs immediately at runtime like an editor console. This is the fastest feedback loop and the preferred path.

**Setup:** Install and connect the [Unity MCP server](https://github.com/niceholmgren/mcp-for-unity), then use an agent that supports MCP tool calling.

### Path B — C# scripts (no MCP required)

The agent writes `.cs` Editor scripts into your Assets folder. Each script uses `[MenuItem]` or runs as an Editor utility that you trigger manually from the Unity menu. The agent tells you which script to run and in what order.

**Setup:** No extra tools needed — just Unity and an AI coding agent.

## How to run

1. Open a Unity project with URP configured
2. Clone this repo into your project's `Assets/` folder
3. *(Optional)* Install and connect the Unity MCP server for Path A
4. Give your agent the benchmark rules from `darko-unity-llm-intelligence-benchmark-test.md`
5. Feed the agent each layer prompt in order (Layer 1 through Layer 7)
6. Score results using the rubrics in each prompt file

## Layer plan (execute in order)

| Layer | What it builds | Prompt file |
|-------|---------------|-------------|
| **1** | Terrain + surface PBR | `context/terrain/generate-realistic-terrain.md` |
| **2** | Terrain splat mapping | `context/terrain/generate-terrain-splat-map.md` |
| **3** | Water system (river, lakes, ponds) | `context/terrain/generate-water-system.md` |
| **4** | Props (trees, rocks) | `context/generate-props.md` |
| **5** | Lighting & post-processing | `context/generate-lighting-post-processing.md` |
| **6** | Sky & clouds | `context/generate-sky-clouds.md` |
| **7** | Horizon closure | `context/generate-horizon-closure.md` |

Each layer builds on top of the previous one. Never skip layers. Each layer only ADDS to the scene — it must not modify previous layers' output.

## Run isolation

Each run must be fully self-contained so you can compare results from multiple agents (e.g. Claude Code, Codex, Cursor) without contamination.

Pick a run ID before starting — e.g. `claude-code-2026-04-10`. All assets and the scene for that run go under:

```
Assets/BenchmarkRuns/{run-id}/
```

The agent must never write to shared folders like `Assets/Materials/` or `Assets/Textures/`. To review a run later, open `Assets/BenchmarkRuns/{run-id}/run-scene.unity`. Previous runs are untouched.

See `darko-unity-llm-intelligence-benchmark-test.md` for the full agent instructions block.

## Key rules

- **Run isolation**: All assets saved under `Assets/BenchmarkRuns/{run-id}/`. Never write to shared folders.
- **Fresh generation**: Every texture, material, and mesh must be generated from scratch by the agent. No reusing existing asset files.
- **Scale (1,1,1)**: All objects use `transform.localScale = (1,1,1)`. Map size is controlled by mesh geometry, not transform scale.
- **Layer isolation**: Each layer only adds — never modifies previous layers' terrain, textures, or water.
- **Raycast grounding**: All props are placed via downward raycast to `ProceduralMeshGround`.

## Scoring

Each layer prompt includes a detailed 100-point scoring rubric. See `darko-unity-llm-intelligence-benchmark-test.md` for the overall scoring guide.

## Community

Visit [darkounity.com](https://darkounity.com) to learn Unity with others, share benchmark results, and connect with builders.
