# Pond Definition (World Standard)

This document defines what a **pond** is in this project.

## What a pond is

A pond is a **small-to-medium inland water body** represented as:
- a smooth, mostly circular water surface mesh/object,
- placed on a localized terrain basin,
- visually readable from gameplay camera distance.

In this project, a pond is implemented as:
- a dedicated water object under parent `Ponds` (e.g. `Pond_1`, `Pond_2`, ...),
- typically a flat circular primitive surface (currently cylinder-top style),
- material `Assets/Materials/WaterPond.mat`,
- no collider required (visual-only) unless explicitly requested.

## Non-negotiable pond characteristics

1. **Visible water surface**
   - Must not be buried in terrain.
   - Must sit above sampled ground by a configured lift offset.

2. **Readable pond shape**
   - Avoid rough or jagged shoreline silhouettes.
   - Prefer smooth circular/oval forms.

3. **Terrain compatibility**
   - Pond area should be prepared by local basin carving/flattening.
   - Do not globally flatten the world terrain to make ponds work.

4. **Separation from vegetation**
   - Trees must not spawn inside pond zones (with shoreline safety margin).

## Typical sizing

- Preferred large pond radius range: `12..22` world units.
- Avoid tiny puddle-like ponds unless user explicitly asks for small ponds.

## Visual defaults

- Material: `WaterPond.mat`
- Water color: blue-green tint with some transparency
- High smoothness for reflective water look
- Rendered as transparent-style surface where shader supports it
