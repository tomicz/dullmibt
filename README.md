# One Shot Prompt — World Generation in Unity

A single prompt that drives an AI coding agent to generate a complete procedural 3D landscape inside the Unity Editor — by driving the editor directly through [Unity MCP](https://darkounity.com/blog/how-do-i-use-ai-in-unity-for-free-2026-tutorial), or by writing C# scripts the user runs in the editor.

## What it generates

A realistic procedural landscape built in 6 layers:

1. **Terrain mesh** — FBM heightfield with momentum-based hydraulic erosion, two-phase hydrology (rivers + lakes), and seed-driven feature probability (ocean edges, lakes, 1–5 rivers)
2. **Terrain textures** — unified 4K context-aware bake with heightlerp blending (flow accumulation and carve depth as regular splatting inputs, wetness as continuous post-blend modifier)
3. **River water** — mask-based transparent water mesh following every carved channel
4. **Pine forest** — Poisson-disk clumped trees with power-law scale and slope orientation
5. **Lighting & post-processing** — sun, shadows, ambient, ACES tonemapping, bloom, color grading
6. **Sky & clouds** — procedural sphere-cluster clouds at altitude

## Requirements

- **Unity 2022.3+** with **URP** (Universal Render Pipeline)
- An AI coding agent (Claude Code, Cursor, Windsurf, Copilot, etc.)
- **Unity MCP server** (recommended) — [setup guide](https://darkounity.com/blog/how-do-i-use-ai-in-unity-for-free-2026-tutorial)

## How agents work

There are two valid execution paths:

### Path A — Unity MCP (recommended)

The agent executes C# code directly in the Unity Editor through Unity MCP's `execute_code` tool. No script files are written — code runs immediately at runtime like an editor console. This is the fastest feedback loop and the preferred path.

**Setup:** Follow the [Unity MCP setup guide](https://darkounity.com/blog/how-do-i-use-ai-in-unity-for-free-2026-tutorial) to install and connect the server, then use an agent that supports MCP tool calling.

### Path B — C# scripts (no MCP required)

The agent writes `.cs` Editor scripts into your Assets folder. Each script uses `[MenuItem]` or runs as an Editor utility that you trigger manually from the Unity menu. The agent tells you which script to run and in what order.

**Setup:** No extra tools needed — just Unity and an AI coding agent.

## How to run

### Step 1 — Set up Unity

- Create a Unity 2022.3+ project with **URP** (Universal Render Pipeline) configured
- *(Optional)* Install [Unity MCP](https://darkounity.com/blog/how-do-i-use-ai-in-unity-for-free-2026-tutorial) for Path A

### Step 2 — Start the prompt

**Option A — with repo cloned:**
Clone this repo into `Assets/` and point your agent at `WORLD-GENERATION-PROMPT.md`.

**Option B — no repo:**
Copy the entire contents of `WORLD-GENERATION-PROMPT.md` and paste it to your agent as the first message.

`WORLD-GENERATION-PROMPT.md` contains everything: global rules, run ID setup, and all 6 layer prompts in order. The agent runs all layers autonomously.

## Run isolation

Each run is self-contained so you can compare results from multiple agents without contamination.

The agent **auto-generates** a run ID from its model name and today's date — e.g. `claude-opus-4-6-2026-04-10/`, `gpt-5-2026-04-10/`. You can also specify a custom run ID in the prompt.

All assets and the scene for a run live under:

```
Assets/WorldGenRuns/{run-id}/
```

To review a run later, open `Assets/WorldGenRuns/{run-id}/run-scene.unity`. Previous runs are untouched.

## Key rules

- **Run isolation**: All assets saved under `Assets/WorldGenRuns/{run-id}/`. Never write to shared folders.
- **Fresh generation**: Every texture, material, and mesh must be generated from scratch by the agent. No reusing existing asset files.
- **Scale (1,1,1)**: All objects use `transform.localScale = (1,1,1)`. Map size is controlled by mesh geometry, not transform scale.
- **Layer isolation**: Each layer only adds — never modifies previous layers' terrain, textures, or water.
- **Raycast grounding**: All props are placed via downward raycast to `ProceduralMeshGround`.

## Community

Visit [darkounity.com](https://darkounity.com) to learn Unity with others, share results, and connect with builders.
