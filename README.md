# Darko Unity LLM Intelligence Benchmark Test

A reproducible benchmark for testing whether AI coding agents can generate procedural 3D worlds inside the Unity Editor — not by writing script files, but by driving the editor directly through [Unity MCP](https://github.com/niceholmgren/mcp-for-unity).

## What this tests

Can an AI agent build a realistic landscape with terrain, water, and props by executing C# code in the Unity Editor at runtime? The benchmark scores planning, API correctness, asset generation, and whether the agent actually finishes.

## Requirements

- **Unity 2022.3+** with **URP** (Universal Render Pipeline)
- **Unity MCP server** installed and running ([mcp-for-unity](https://github.com/niceholmgren/mcp-for-unity))
- An AI coding agent that supports MCP tool calling (Claude Code, Cursor, Windsurf, etc.)

## How agents should work

**Agents must execute C# code through Unity MCP's `execute_code` tool — NOT write `.cs` script files into the project.**

The MCP server lets agents run C# directly in the Unity Editor (like typing into an immediate-mode console). This is how terrain gets built, textures get baked, meshes get created, and materials get assigned — all at runtime through MCP calls.

If your agent starts creating `.cs` files in your Assets folder instead of calling MCP tools, it is doing it wrong. The benchmark tests runtime editor manipulation, not traditional scripting.

## How to run

1. Open a Unity project with URP configured
2. Clone this repo into your project's `Assets/` folder
3. Install and connect the Unity MCP server
4. Give your agent the benchmark rules from `darko-unity-llm-intelligence-benchmark-test.md`
5. Feed the agent each layer prompt in order (Layer 1 through Layer 4)
6. Score results using the rubrics in each prompt file

## Layer plan (execute in order)

| Layer | What it builds | Prompt file |
|-------|---------------|-------------|
| **1** | Terrain + surface PBR | `context/terrain/generate-realistic-terrain.md` |
| **2** | Terrain splat mapping | `context/terrain/generate-terrain-splat-map.md` |
| **3** | Water system (river, lakes, ponds) | `context/terrain/generate-water-system.md` |
| **4** | Props (trees, rocks) | `context/generate-props.md` |

Each layer builds on top of the previous one. Never skip layers. Each layer only ADDS to the scene — it must not modify previous layers' output.

## Key rules

- **Fresh generation**: Every texture, material, and mesh must be generated from scratch by the agent. No reusing existing asset files.
- **Scale (1,1,1)**: All objects use `transform.localScale = (1,1,1)`. Map size is controlled by mesh geometry, not transform scale.
- **Layer isolation**: Each layer only adds — never modifies previous layers' terrain, textures, or water.
- **Raycast grounding**: All props are placed via downward raycast to `ProceduralMeshGround`.

## Scoring

Each layer prompt includes a detailed 100-point scoring rubric. See `darko-unity-llm-intelligence-benchmark-test.md` for the overall scoring guide.

## Community

Visit [darkounity.com](https://darkounity.com) to learn Unity with others, share benchmark results, and connect with builders.
