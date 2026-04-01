# Inspiration

## Alexander Ameye — Holographic Card Shader

> Source: https://ameye.dev/notes/holographic-card-shader/

A holographic Pokémon card shader built in Unity URP. The goal was to do as much as possible in the shader itself without relying on textures. Uses a single 3D model for the Pokémon and a single texture for text. Card design inspired by Rob Fichman. Demo model: Cosmog by AlmondFeather on Sketchfab.

### Techniques Used

**Card Frame** — Outer edge, horizontal divider, window, and title bar all drawn with rectangle and polygon SDFs. 65 exposed shader properties for full layout control.

**Icons** — Pokémon type icons made from signed distance functions. Stars = 2 vesica SDFs blended. Moon = 2 spheres. Planet = several spheres and ellipses. Parallax effect fakes depth between icons on the same plane. Reference: [Inigo Quilez 2D SDFs](https://iquilezles.org/articles/distfunctions2d/).

**Raymarched Spheres** — Used for type and attack indicators. Blinn-Phong lighting makes them look 3D without actual geometry.

**Text** — Initially attempted in-shader with bezier curves and SDFs (got close) but switched to a simple texture for the exact font needed.

**Holographic Sparkles** — Voronoi patterns where colors are converted to directions, then dot product with view direction makes different sparkles appear at different viewing angles. Multiple voronoi layers overlaid for richer patterns.

**Parallax Effect** — UV offset based on view direction. Fakes depth on a flat card. Combined with raymarched spheres for a convincing 3D look.

**Stencil Shader** — 3D Pokémon scene only visible through the card window. Reference: [Cyanilux — Stencils in URP](https://www.cyanilux.com/tutorials/holofoil-card-shader-breakdown/).

### Other References

- [Cyanilux — Holofoil Card Shader Breakdown](https://www.cyanilux.com/tutorials/holofoil-card-shader-breakdown/)
- [Daniel Ilett — Holofoil Cards in Shader Graph](https://danielilett.com/2025-10-13-tut11-01-holofoil-cards/)
