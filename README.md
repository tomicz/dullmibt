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

### Step 1 — Set up Unity

- Create a Unity 2022.3+ project with **URP** (Universal Render Pipeline) configured
- Clone this repo into your project's `Assets/` folder

### Step 2 — Choose your execution path

- **Path A (MCP):** Install and connect the [Unity MCP server](https://github.com/niceholmgren/mcp-for-unity)
- **Path B (scripts):** No setup needed — the agent will write `.cs` files you run manually

### Step 3 — Pick a run ID

Choose a unique name for this run, e.g. `claude-code-2026-04-10`. You'll use this in the next step. Every run gets its own isolated folder so you can compare agents later.

### Step 4 — Give your agent the system prompt

Open `darko-unity-llm-intelligence-benchmark-test.md` and copy the agent instructions block. Fill in your **run ID** and **execution path**, then paste it to your agent as the first message (or system prompt).

### Step 5 — Run each layer in order

For each layer (1 through 7), copy the full contents of the layer's prompt file and send it to your agent. Wait for the layer to complete and verify before moving to the next.

| Layer | Prompt file |
|-------|-------------|
| 1 — Terrain | `context/terrain/generate-realistic-terrain.md` |
| 2 — Splat map | `context/terrain/generate-terrain-splat-map.md` |
| 3 — Water | `context/terrain/generate-water-system.md` |
| 4 — Props | `context/generate-props.md` |
| 5 — Lighting | `context/generate-lighting-post-processing.md` |
| 6 — Sky & clouds | `context/generate-sky-clouds.md` |
| 7 — Horizon | `context/generate-horizon-closure.md` |

### Step 6 — Score the result

Each layer prompt includes a 100-point rubric at the bottom. Score each layer and total the results.

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
