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
- *(Optional)* Install [Unity MCP](https://github.com/niceholmgren/mcp-for-unity) for Path A

### Step 2 — Start the benchmark

**Option A — with repo cloned:**
Clone this repo into `Assets/` and point your agent at `START-BENCHMARK.md`.

**Option B — no repo:**
Copy the entire contents of `START-BENCHMARK.md` and paste it to your agent as the first message.

`START-BENCHMARK.md` contains everything: global rules, run ID setup, and all 7 layer prompts in order. The agent runs all layers autonomously.

### Step 3 — Score the result

Each layer section in `START-BENCHMARK.md` includes a 100-point rubric. Total score is out of 700.

## Run isolation

Each run is fully self-contained so you can compare results from multiple agents (e.g. Claude, Codex, Cursor) without contamination.

The agent **auto-generates** a run ID from its model name and today's date — e.g. `claude-opus-4-6-2026-04-10/`, `gpt-5-2026-04-10/`, `codex-2026-04-10/`. You can also specify a custom run ID in the prompt if you want.

All assets and the scene for a run live under:

```
Assets/BenchmarkRuns/{run-id}/
```

To review a run later, open `Assets/BenchmarkRuns/{run-id}/run-scene.unity`. Previous runs are untouched.

## Key rules

- **Run isolation**: All assets saved under `Assets/BenchmarkRuns/{run-id}/`. Never write to shared folders.
- **Fresh generation**: Every texture, material, and mesh must be generated from scratch by the agent. No reusing existing asset files.
- **Scale (1,1,1)**: All objects use `transform.localScale = (1,1,1)`. Map size is controlled by mesh geometry, not transform scale.
- **Layer isolation**: Each layer only adds — never modifies previous layers' terrain, textures, or water.
- **Raycast grounding**: All props are placed via downward raycast to `ProceduralMeshGround`.

## Scoring

Each layer in `START-BENCHMARK.md` includes a 100-point scoring rubric. Total score is out of 700 across all 7 layers. Layer 1 has a map size multiplier (0.5x–1.3x) that applies to its score.

## Community

Visit [darkounity.com](https://darkounity.com) to learn Unity with others, share benchmark results, and connect with builders.
