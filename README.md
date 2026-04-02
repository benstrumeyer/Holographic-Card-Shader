# Holographic Card Shader — Plan

<video src="https://github.com/benstrumeyer/Holographic-Card-Shader/raw/master/holographic-card-shader.mp4" autoplay loop muted playsinline></video>

> Reference: [Alexander Ameye — Holographic Card Shader](https://ameye.dev/notes/holographic-card-shader/)
> Goal: Recreate this effect for **every Pokémon** with an automated pipeline

---

## What We're Building

A holographic Pokémon card shader (Unity URP / Shader Graph) applied to every Pokémon — **fully animated** — with a batch automation pipeline to process all models and animations in Blender, and render animated holographic cards at scale.

---

## The Shader (from Ameye's breakdown)

All done in-shader with minimal textures (one 3D model per Pokémon, one text texture):

- **Card frame** — rectangles + polygon SDFs, 65 exposed properties for full layout control
- **Icons** — Pokémon type icons built from signed distance functions (vesica, sphere, ellipse combos)
- **Raymarched spheres** — fake 3D spheres in the shader for type/attack indicators (Blinn-Phong lit)
- **Text** — texture-based (bezier SDF approach was too labor-intensive)
- **Holographic sparkles** — voronoi patterns, color→direction conversion, dot product with view direction so different sparkles appear at different angles, multiple layers overlaid
- **Parallax effect** — UV offset based on view direction to fake depth on a flat card
- **Stencil shader** — 3D Pokémon scene only visible through the card window

Full details: https://ameye.dev/notes/holographic-card-shader/

---

## 3D Models

### Target Format
- Models as **GLTF/GLB/FBX** for Blender import
- Need: mesh, textures, rigging, animations

### Sourcing
Model and animation sourcing details kept offline — see local notes.

---

## Blender Processing Workflow

Assumes models + animations are already available locally as importable files.

### Per-Pokémon Steps
1. Import model into Blender
2. Import animations (e.g. .smd files via Blender add-on)
3. Normalize scale, center, orient for card window framing
4. Set idle/showcase animation
5. Adjust FPS if animations seem slow
6. Export as .fbx for Unity import

---

## Automation Pipeline

Assumes models + animations are already local. Goal: as little clicking as possible.

### Blender Batch Processing (fully scriptable, headless)

```bash
for pokemon_dir in /models/*/; do
    blender --background --python process_pokemon.py -- \
        --input "${pokemon_dir}" \
        --output "${pokemon_dir}/output.fbx"
done
```

`process_pokemon.py` handles per-Pokémon:
1. Import model
2. Import all animation files
3. Normalize scale, center, orient
4. Set best idle animation
5. Export final .fbx for Unity

### GUI Automation (for intermediate tools without CLI)

Some intermediate format conversion tools are GUI-only. Two approaches:

#### Plan A — AutoHotKey (available now)
- Script click coordinates + keystrokes for GUI tools
- Lock window to fixed position/size for consistency
- Loop through all Pokémon folders
- **Pros:** Can build now, no dependencies
- **Cons:** Brittle, needs babysitting

#### Plan B — Claude Desktop with Computer Use (future)
- Claude Desktop drives the GUI visually
- More resilient — adapts to UI changes, reads errors, retries
- **Pros:** Robust, handles edge cases
- **Cons:** Not set up yet

### Unity Batch Pipeline (C# Editor Script)

```
For each .fbx:
  → import → create prefab
  → assign holographic card material
  → pull PokéAPI data (type, name, stats) → configure card properties
  → render card from multiple angles → export PNG/video
```

### Data Pipeline Summary

| Step | Input | Output | Tool | Automatable? |
|------|-------|--------|------|-------------|
| Process model + anims | Raw model + anims | .fbx (normalized, animated) | Blender Python | Yes |
| Unity import | .fbx | Unity prefab | Unity Editor script | Yes |
| Card config | PokéAPI data | Card properties | Python/C# script | Yes |
| Render | Prefab + shader | PNG/video per card | Unity Editor script | Yes |

---

## Open Questions

1. How to handle alternate forms / shiny / mega / regional variants?
2. Card text — generate textures per Pokémon or do it in-shader with SDF font?
3. How many Pokémon total? (~1000+ including forms)
4. Output format — looping video or interactive WebGL? (animated is the goal)

---

## References

- [Alexander Ameye — Holographic Card Shader](https://ameye.dev/notes/holographic-card-shader/)
- [Cyanilux — Holofoil Card Shader Breakdown](https://www.cyanilux.com/tutorials/holofoil-card-shader-breakdown/)
- [Daniel Ilett — Holofoil Cards in Shader Graph](https://danielilett.com/2025-10-13-tut11-01-holofoil-cards/)
- [Inigo Quilez — 2D SDF functions](https://iquilezles.org/articles/distfunctions2d/)
- [PokéAPI](https://pokeapi.co/)
