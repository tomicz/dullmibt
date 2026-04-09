# Biome: Temperate Deciduous Forest

Use this prompt to test whether an LLM can generate a procedural world that reads as a real temperate deciduous forest — not just scattered objects, but a coherent ecosystem with spatial rules.

## Ecological profile

A mid-latitude forest dominated by broadleaf deciduous trees with a layered vertical structure. Characterized by high canopy coverage, a distinct understory layer, dense ground vegetation, and seasonal variation in light penetration.

## Vertical structure (layers the LLM must reproduce)

| Layer | What belongs | Height band | Coverage |
|-------|-------------|-------------|----------|
| **Canopy** | Large deciduous trees (sphere-canopy type), occasional conifers mixed in | Tallest objects, trunk 2-4m equivalent | 50-70% of tree population is deciduous |
| **Sub-canopy** | Smaller/younger trees, scaled-down versions | 60-75% height of canopy trees | 15-25% of tree population |
| **Shrub** | Dense bush clusters, grouped more than solo | Ground level, half-buried | High density near tree gaps and edges |
| **Ground cover** | Grass (moderate-tall density), wildflowers in clearings and gaps | Surface | Grass thins under dense canopy, flowers favor sunlit patches |
| **Floor** | Rocks — moss-covered feel, partially buried, scattered not clustered | Surface, embedded | Sparse, individual boulders |

## Spatial rules (what makes it feel like a real forest)

### Tree clustering
Trees must NOT be uniformly distributed. Real forests have **dense groves** with tight spacing and **natural clearings** with open ground. The LLM must create both. Implementation hint: generate grove center points first, then place trees with probability weighted by distance to nearest grove center.

### Edge behavior
Bushes and flowers concentrate at the edges of tree clusters and around clearings, not uniformly across the map. The transition between grove and clearing is where the most understory diversity exists.

### Canopy gaps
Where there are no tall trees, grass should be denser and flowers should appear. Under dense canopy, grass thins out and flowers are rare. The LLM should estimate local canopy density (e.g. count trees within 15-20 world units) and modulate ground cover accordingly.

### Conifer placement
The minority conifer trees (cone trees) should appear in small groups of 3-7, not evenly mixed with deciduous trees. In a real temperate forest, conifers cluster on ridges, slopes, or poorer soil patches. Place conifer groups at grove EDGES (low-moderate grove influence), not deep inside deciduous groves.

### Rock placement
Rocks are solitary or in pairs, not in formations. They feel old, embedded, like they have been there longer than the trees. Deeply buried (20-35%), with wide spacing between individual boulders.

### Water
Ponds sit in natural low points. Vegetation is lush near water but no trees directly in the shoreline zone. Bushes can approach closer to water than trees.

## What does NOT belong

- Uniform grid-like tree spacing
- Trees and conifers perfectly mixed 50/50
- Bare ground with no understory
- Rocks in organized lines or dense mounds (that is highland/rocky biome)
- Flowers in deep shade under canopy
- Identical tree scales across the map
- Tropical flower colors (orange, red) in this biome

## Color palette

- **Canopy**: Rich greens, slight variation (some yellower, some bluer-green)
- **Conifers**: Distinctly darker green than deciduous
- **Understory/bushes**: Medium green, slightly muted
- **Grass**: Varied — brighter in clearings, darker/sparser under trees
- **Flowers**: Blues, whites, yellows — woodland wildflower palette
- **Rocks**: Gray with slight green/brown tint (mossy)
- **Terrain**: Rich dark earth tones, not bright green

## Density guidelines (per 1000x1000 world unit footprint)

| Element | Range | Distribution |
|---------|-------|-------------|
| Deciduous trees | 80-150 | Clustered in groves |
| Sub-canopy trees | 30-60 | Near/between canopy groups |
| Conifer trees | 15-40 | Small groups of 3-7 on grove edges |
| Bush clusters | 60-100 | Concentrated at grove edges and clearings |
| Flowers | 80-200 | Concentrated in clearings and gaps |
| Rocks | 20-40 | Sparse, solitary, well-buried |
| Ponds | 2-8 | Natural low points only |

## Scoring criteria (biome-specific)

| Criterion | Weight | What "good" looks like |
|-----------|--------|----------------------|
| **Clustering** | High | Trees form visible groves with clearings between them, not uniform distribution |
| **Layer coherence** | High | Understory reacts to canopy — gaps have more flowers/grass, dense areas have less |
| **Species ratio** | Medium | Deciduous clearly dominant, conifers minority and grouped separately |
| **Scale variation** | Medium | Mix of large mature trees and smaller sub-canopy trees, not all same size |
| **Edge ecology** | Medium | Bushes and flowers cluster at transitions between forest and clearing |
| **Rock character** | Low | Solitary, deeply buried, not in formations |
| **Color coherence** | Low | Palette reads as temperate woodland, not tropical or arid |

## Technical requirements

- All grounded objects placed via downward raycast to `ProceduralMeshGround`
- Pond exclusion zones respected for all vegetation
- Grove structure must be generated BEFORE individual tree placement
- Canopy density map must inform understory (bushes, flowers, grass) placement
- Conifer groups must be spatially separate from deciduous groves
