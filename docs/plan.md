# Holographic Card Shader — Plan

> Reference: [Alexander Ameye — Holographic Card Shader](https://ameye.dev/notes/holographic-card-shader/)
> Goal: Recreate this effect for **every Pokémon** with an automated pipeline

---

## What We're Building

A holographic Pokémon card shader (Unity URP / Shader Graph) applied to every Pokémon, with a batch automation pipeline to rip all models, process them in Blender, and render cards at scale.

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

## Pokémon Model Ripping

### Sources
- **Scarlet/Violet (Switch):** GFBMDL/TRMDL MaxScript by Random Talking Bush → extract .gfbmdl models
- **Sword/Shield (Switch):** Switch Toolbox → extract models
- **Sun/Moon/USUM (3DS):** SPICA by gdkchan → batch export to .dae/.gltf
- **Existing dumps:** Random Talking Bush's model repository has most Pokémon already ripped

### Target Format
- Export all models as **GLTF/GLB** for Blender import
- Need: mesh, textures, rigging (for idle pose or T-pose)

---

## Manual Workflow (per Pokémon)

This is the manual process from the tutorial. Our goal is to automate as much of this as possible.

### Step 1 — Export Skeleton as FBX
- Click on model's skeleton in Blender, export as .fbx
- Export options: **only Armature selected**

### Step 2 — Switch Toolbox
- Download [Switch Toolbox](https://github.com/KillzXGaming/Switch-Toolbox)
- Used for viewing models/animations ripped from Switch games

### Step 3 — Prep Animation Files
- Collect all `.tranm` files (animation files per Pokémon)
- Put them in a separate folder
- Rename all extensions to `.gfbanm` (required for Switch Toolbox to read them)
- **Status: Already downloaded** — animations are local

### Step 4 — Get a Bulbasaur .gfpak
- Need a .gfpak file from Pokémon Sword/Shield RomFS dump
- This acts as a "container" to load the skeleton + animations into Switch Toolbox

### Step 5 — Open .gfpak in Switch Toolbox
- Open Bulbasaur .gfpak, double-click `pm0001_00.gfbmdl` to view the model

### Step 6 — Replace Model with Target Skeleton
- Replace Bulbasaur model with target Pokémon's skeleton .fbx
- Import settings: **disable "Use Original Bones"**, click OK

### Step 7 — Load Animations
- Drag and drop all renamed `.gfbanm` animation files into Switch Toolbox
- Double-click each to load them in

### Step 8 — Export Animations as .smd
- Play each animation, export as `.smd` file
- Repeat for all animations, save all .smd files in one folder

### Step 9 — Blender SMD Import Add-on
- Install Blender add-on for .smd import (install the **zipped** folder, don't unzip)
- Enable it and hit Refresh

### Step 10 — Import Animations into Blender
- With Pokémon model open in Blender, import .smd animations
- Adjust FPS if animations seem slow

---

## Automation Pipeline

### What Can Be Automated

| Step | Manual Action | Automation Approach |
|------|--------------|-------------------|
| 1 — Export skeleton | Select armature, export fbx | **Blender Python** — script to select armature + export with correct settings |
| 3 — Rename .tranm → .gfbanm | Manually rename files | **PowerShell/Bash** — `ren *.tranm *.gfbanm` one-liner per folder |
| 6 — Replace model in Switch Toolbox | GUI click import | **AutoHotKey** — macro the click sequence (Switch Toolbox has no CLI) |
| 7 — Load animations | Drag and drop into GUI | **AutoHotKey** — macro the drag/drop or use file dialog |
| 8 — Export .smd per animation | Click play, click export, repeat | **AutoHotKey** — loop: select animation → play → export .smd → next |
| 10 — Import .smd into Blender | Import one at a time | **Blender Python** — batch import all .smd files in a folder |

### The Hard Part: Switch Toolbox (Steps 5-8)

Switch Toolbox is a **GUI-only** Windows app. No CLI, no scripting API. This is the bottleneck. Two plans for tackling it:

#### Plan A — AutoHotKey (available now)
- Script the exact click coordinates + keystrokes
- Fragile (breaks if window moves/resizes) but works today
- Lock Switch Toolbox to a fixed window position/size
- Loop: open .gfpak → replace model → load animations → export each .smd → close → next
- Build in delays/waits for file load times
- **Pros:** Can build this now, no dependencies
- **Cons:** Brittle, needs babysitting, breaks on UI changes

#### Plan B — Claude Desktop with Computer Use (future)
- Claude Desktop can see the screen and click through GUI apps
- Give it the manual steps as instructions, let it drive Switch Toolbox
- More resilient — it can adapt if buttons move, read error dialogs, retry
- Could handle the full loop: open file → replace → load anims → export → next Pokémon
- **Pros:** More robust than pixel-perfect AHK macros, can handle edge cases
- **Cons:** Don't have Claude Desktop set up yet, would need computer use enabled

#### Stretch — Skip Switch Toolbox entirely
- The .gfbanm format has been reverse-engineered
- Research if .gfbanm → .smd conversion can be done with existing Python libraries
- If a standalone converter exists, the entire pipeline becomes Blender Python + Bash with zero GUI

### What's Fully Scriptable (No GUI)

Everything in Blender can be headless:

```bash
# For each Pokémon folder:
for pokemon_dir in /models/*/; do
    # Step 1: Export skeleton
    blender --background "${pokemon_dir}/model.blend" --python export_skeleton.py

    # Step 3: Rename animation files
    for f in "${pokemon_dir}/anims/"*.tranm; do
        mv "$f" "${f%.tranm}.gfbanm"
    done

    # [Steps 5-8: Switch Toolbox — AutoHotKey or manual]

    # Step 10: Import all .smd animations into Blender model
    blender --background "${pokemon_dir}/model.blend" --python import_animations.py -- "${pokemon_dir}/smd/"

    # Bonus: Normalize, pose, export for Unity
    blender --background "${pokemon_dir}/model.blend" --python process_for_unity.py -- "${pokemon_dir}/output.fbx"
done
```

### Full Pipeline Overview

```
Per Pokémon folder:
  /raw/
    model.blend          ← already ripped 3D model
    anims/*.tranm        ← raw animation files

  Script Phase 1 (Blender Python):
    → export skeleton.fbx (armature only)

  Script Phase 2 (Bash/PowerShell):
    → rename *.tranm → *.gfbanm

  Script Phase 3 (AutoHotKey OR custom CLI):
    → Switch Toolbox: load gfpak → replace model → load anims → export *.smd

  Script Phase 4 (Blender Python):
    → import all .smd animations
    → normalize scale, center, orient
    → set best idle animation
    → export final .fbx for Unity

  Script Phase 5 (Unity C# Editor Script):
    → import .fbx → create prefab
    → assign holographic card material
    → pull PokéAPI data (type, name, stats) → configure card properties
    → render card from multiple angles → export PNG/video
```

---

## Data Pipeline (per Pokémon)

| Step | Input | Output | Tool | Automatable? |
|------|-------|--------|------|-------------|
| Export skeleton | .blend model | .fbx (armature only) | Blender Python | Yes |
| Rename anims | .tranm files | .gfbanm files | Bash/PowerShell | Yes |
| Convert anims | .gfbanm + skeleton | .smd per animation | Switch Toolbox | Partial (AHK/custom CLI) |
| Import anims | .smd files + model | .blend with animations | Blender Python | Yes |
| Process model | .blend | .fbx (normalized, posed) | Blender Python | Yes |
| Unity import | .fbx | Unity prefab | Unity Editor script | Yes |
| Card config | PokéAPI data | Card properties | Python/C# script | Yes |
| Render | Prefab + shader | PNG/video per card | Unity Editor script | Yes |

---

## Open Questions

1. Which game generation has the best/most complete model set?
2. Do we want animated idle poses or static?
3. How to handle alternate forms / shiny / mega / regional variants?
4. Card text — generate textures per Pokémon or do it in-shader with SDF font?
5. How many Pokémon total? (~1000+ including forms)
6. Output format — static images, looping video, or interactive WebGL?

---

## References

- [Alexander Ameye — Holographic Card Shader](https://ameye.dev/notes/holographic-card-shader/)
- [Cyanilux — Holofoil Card Shader Breakdown](https://www.cyanilux.com/tutorials/holofoil-card-shader-breakdown/)
- [Daniel Ilett — Holofoil Cards in Shader Graph](https://danielilett.com/2025-10-13-tut11-01-holofoil-cards/)
- [Rob Fichman — Card design inspiration](https://ameye.dev/notes/holographic-card-shader/)
- [Inigo Quilez — 2D SDF functions](https://iquilezles.org/articles/distfunctions2d/)
- [SPICA — 3DS model extractor](https://github.com/gdkchan/SPICA)
- [Switch Toolbox](https://github.com/KillzXGaming/Switch-Toolbox)
- [PokéAPI](https://pokeapi.co/)
