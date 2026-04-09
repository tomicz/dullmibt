# Prompt: Procedural Realistic Tree Generation

Use this prompt with a Unity-capable coding agent (Unity MCP) to generate a realistic procedural tree mesh. Each tree is **unique** — the algorithm uses a per-tree seed so no two trees look alike while remaining reproducible.

## Design Philosophy

Trees are built from code using **recursive branching** — not Unity primitives (no Sphere/Cylinder GameObjects). The algorithm produces a trunk, main branches, sub-branches, and cross-billboard leaf clusters as a single mesh per component (trunk mesh + leaf mesh). This creates trees that read as organic shapes with natural variation.

## Copy/Paste Prompt

```text
Generate a single procedural tree at the specified position in the active Unity scene.
The tree must be built entirely from generated mesh data — no Unity primitives.

IMPORTANT RULES:
- transform.localScale is ALWAYS (1, 1, 1). Size is controlled by mesh geometry.
- Each tree uses a unique integer seed for its System.Random instance. This makes every tree different but reproducible.
- Tree consists of TWO child GameObjects: "Trunk" (bark mesh) and "Leaves" (foliage mesh).
- Materials use shader "Universal Render Pipeline/Baked Lit" (NOT Lit) to avoid specular glow artifacts.
  If Baked Lit is unavailable, use URP Lit with _SpecularHighlights=0, _EnvironmentReflections=0, _Smoothness=0.

1. TREE PARAMETERS (vary per tree using seeded random)
   Base values (randomize +-20% per tree, 1 unit = 1 meter):
   - Trunk height: 5-8m
   - Trunk base radius: 0.25-0.45m
   - Trunk top radius: 0.04-0.08m
   - Recursion depth: 4-5 levels
   - Main branches from trunk: 4-6
   - Branch start height: 35-50% up trunk

   Branching ratios (these make it look natural):
   - Child branch length = parent length * 0.55-0.75
   - Child branch radius = parent end radius * 0.45-0.65
   - Branch angle from parent: 22-45 degrees
   - Taper exponent: 1.3 (exponential, not linear — thicker near base)
   - Gravity droop: depth * 0.06 (lower branches sag more)
   - Random perturbation: +-0.08 on direction vector per branch

2. TRUNK AND BRANCH MESH
   Each branch segment is a tapered cylinder:
   - 6 radial segments (cross-section vertices)
   - Trunk: 8-10 length segments for smooth curve
   - Main branches: 4-5 segments
   - Sub-branches: 2-3 segments
   - Exponential taper: radius = lerp(r0, r1, pow(t, 1.3))

   Trunk shape:
   - Slight S-curve lean (not perfectly straight)
   - Apply gentle random bend via direction perturbation at each segment

   Branch spawning:
   - Distribute children along parent at 35-95% of parent length
   - Spread evenly around parent axis (360/count + random offset +-25 degrees)
   - Lower branches more horizontal (25 deg), upper more vertical (45 deg)

   Vertex colors: bark brown with variation
   - Base: RGB(0.22, 0.13, 0.06) * random(0.8, 1.2)

3. LEAF CLUSTERS (cross-billboard approach)
   At terminal and near-terminal branch tips, add leaf clusters:
   - Each cluster = 3 intersecting quads at 0, 60, 120 degree rotations
   - This creates volume readable from any camera angle
   - Quad size: 1.0-2.0m (realistic foliage scale, 1 unit = 1 meter)
   - Slight random tilt on each quad (+-15 degrees)
   - Place clusters at branch endpoints AND along branches (50% point)
   - Depth 3+ branches get leaf clusters

   Vertex colors: green with natural variation
   - Base: RGB(0.08, 0.25, 0.03) * random(0.6, 1.0)
   - Bottom vertices slightly darker (multiply 0.8) for depth

   Double-sided rendering (both triangle windings per quad).

4. MATERIALS (create once, reuse across all trees)
   - Assets/Materials/TreeBark.mat
     Shader: Universal Render Pipeline/Baked Lit
     _BaseColor: (0.32, 0.20, 0.10, 1.0)
     _Cull: 2 (Front only)

   - Assets/Materials/TreeLeaves.mat
     Shader: Universal Render Pipeline/Baked Lit
     _BaseColor: (0.15, 0.35, 0.08, 1.0)
     _Cull: 0 (Off — double-sided)
     _AlphaClip: 1 (enabled)
     _Cutoff: 0 (threshold)

   FALLBACK if Baked Lit produces glow or is unavailable:
     Use "Universal Render Pipeline/Lit" with:
     _SpecularHighlights: 0
     _EnvironmentReflections: 0
     _Smoothness: 0
     _Metallic: 0
     Enable keywords: _SPECULARHIGHLIGHTS_OFF, _ENVIRONMENTREFLECTIONS_OFF

5. SEED SYSTEM
   - Each tree gets a unique seed: baseSeed + treeIndex
   - All random decisions (angles, lengths, radii, colors) use this seeded Random
   - Same seed = same tree shape (reproducible)
   - Different seed = visually different tree (no duplicates)

6. GAMEOBJECT HIERARCHY
   TreeName (empty parent, position at ground)
   ├── Trunk (MeshFilter + MeshRenderer, TreeBark.mat)
   └── Leaves (MeshFilter + MeshRenderer, TreeLeaves.mat)

   - All transforms: localScale = (1, 1, 1)
   - Parent position = raycast ground point
   - Children localPosition = Vector3.zero

7. MESH SETTINGS
   - IndexFormat: UInt32
   - RecalculateNormals() and RecalculateTangents() on trunk
   - RecalculateNormals() on leaves
   - RecalculateBounds() on both

8. REPORT
   Return per tree:
   - Seed used
   - Trunk: vertex count, triangle count
   - Leaves: vertex count, cluster count
   - Total height from ground
   - Position
```

---

## Expected Result

- A single tree that reads as an organic shape — branching trunk with foliage canopy.
- Brown bark with natural color variation, green leaves with depth shading.
- No specular glow or mirror reflections on leaves.
- Each tree generated with a different seed looks visually distinct.
- Reproducible: same seed always produces the same tree.

---

## Algorithm Reference

### Recursive Branching (L-System style)

```
function growBranch(start, direction, length, radius, depth):
    if depth > maxDepth or radius < 0.02: return

    // Apply gravity droop
    direction += down * (depth * 0.06)
    direction += randomPerturbation(+-0.08)
    direction = normalize(direction)

    end = start + direction * length
    endRadius = radius * 0.6

    addTaperedCylinder(start, end, radius, endRadius)

    // Add leaves at depth >= maxDepth - 2
    if depth >= maxDepth - 2:
        addLeafCluster(end, size=2.5-5.5)
        addLeafCluster(midpoint, size * 0.7)

    // Spawn children
    childCount = 5 (trunk), 3-4 (branches), 2 (twigs)
    for each child:
        spawnPoint = lerp(start, end, 0.35 + i/count * 0.6)
        childDir = rotate direction by spreadAngle(22-45) around parentAxis
        childLength = length * 0.55-0.75
        childRadius = endRadius * 0.45-0.65
        growBranch(spawnPoint, childDir, childLength, childRadius, depth+1)
```

### Cross-Billboard Leaf Cluster

Three quads intersecting at 60-degree intervals create a volumetric appearance from any viewing angle. Each quad is a simple 4-vertex rectangle with both front and back face triangles.

```
function addLeafCluster(center, size):
    for angle in [0, 60, 120]:
        rotation = Euler(random(+-15), angle + random(+-10), random(+-10))
        right = rotation * Vector3.right * size
        up = rotation * Vector3.up * size
        emit 4 vertices: center +- right, center +- up
        emit front triangles + back triangles (double-sided)
```

### Key Ratios That Make Trees Look Natural

| Parameter | Value | Why |
|-----------|-------|-----|
| Length ratio (child/parent) | 0.55-0.75 | Prevents unrealistically long twigs |
| Radius ratio (child/parent) | 0.45-0.65 | Maintains structural taper |
| Taper exponent | 1.3 | Exponential taper looks more organic than linear |
| Branch angle | 22-45 degrees | >50 looks unnatural |
| Gravity droop | depth * 0.06 | Lower branches sag realistically |
| Random perturbation | +-0.08 | Breaks perfect symmetry |

---

## Scoring Rubric (100 points)

### Tree Structure (40 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Recursive branching** | 10 | Visible trunk → branches → sub-branches hierarchy. Not random tubes. |
| **Natural proportions** | 8 | Correct length/radius ratios. Branches taper. No unnaturally thick twigs. |
| **Trunk quality** | 7 | Slight curve, proper taper, smooth cylinder mesh. |
| **Branch distribution** | 8 | Branches spread around trunk. Lower ones more horizontal. Natural angles. |
| **Gravity influence** | 4 | Lower/outer branches droop slightly. |
| **Uniqueness per seed** | 3 | Different seeds produce visibly different trees. Same seed = same tree. |

### Foliage (25 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Leaf cluster coverage** | 8 | Canopy reads as full foliage, not sparse dots. |
| **Cross-billboard technique** | 6 | 3+ intersecting quads per cluster. Reads as volume from any angle. |
| **Cluster placement** | 6 | At branch tips and along branches. Natural distribution. |
| **Color variation** | 5 | Green with visible variation. Darker undersides. Not uniform flat color. |

### Materials & Technical (20 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **No glow/reflection** | 6 | Baked Lit or Lit with specular OFF. No white bloom on leaves. |
| **Correct shader setup** | 4 | Baked Lit preferred. Proper _Cull settings (leaves double-sided, bark front). |
| **Mesh quality** | 4 | Normals recalculated. UInt32 index format. No degenerate triangles. |
| **Scale rule** | 3 | All localScale = (1,1,1). Size from geometry only. |
| **Performance** | 3 | Reasonable vertex count. Single tree < 10K verts total. |

### Visual Quality (15 points)

| Criterion | Points | Full marks |
|-----------|--------|------------|
| **Reads as a tree** | 8 | From 50m away, clearly identifiable as a tree. Not abstract shape. |
| **Bark vs leaf distinction** | 4 | Two materials, visible difference between wood and foliage. |
| **Casts proper shadow** | 3 | Tree shadow on ground matches tree shape. |

### Score Bands

| Range | Meaning |
|-------|---------|
| **0-25** | No tree or unrecognizable shape |
| **26-50** | Basic tree shape but wrong materials, glow, or no branching |
| **51-70** | Recognizable tree, branching works, minor issues |
| **71-85** | Good tree with proper materials, branching, and foliage |
| **86-100** | Natural-looking tree, seed variation, no glow, clean hierarchy |

---

## Quick Reference for Agents

| Problem | Solution |
|---------|----------|
| White glow on leaves | Switch to Baked Lit shader. Or disable _SpecularHighlights and _EnvironmentReflections. |
| Leaves invisible from some angles | Use double-sided rendering (_Cull = 0) and emit both triangle windings. |
| Tree looks like random tubes | Check branching ratios. Child length should be 55-75% of parent. |
| All trees look identical | Each tree needs a unique seed for its Random instance. |
| Tree too small on terrain | Base trunk height should be 5-8m (1 unit = 1 meter). |
| Leaves are tiny dots | Leaf quad size should be 1.0-2.0m. Cross-billboard clusters, not individual leaves. |
| Flat/symmetric look | Add random perturbation to branch directions. Apply gravity droop. |
