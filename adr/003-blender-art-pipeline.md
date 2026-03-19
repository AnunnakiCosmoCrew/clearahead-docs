# ADR-003: Blender 3D render pipeline for art assets

**Status:** Accepted
**Date:** 2026-03-20
**Deciders:** Mert Ertugrul

## Context

ClearAhead! needs isometric sprite assets (vehicles, roads, environment) that precisely match the game's 2:1 dimetric projection. The visual style is "jelly" — chunky, rounded, toy-like objects with glossy subsurface-scattering materials. The sole developer has no prior art experience.

All sprites must render at a consistent isometric angle across 50+ assets with 4 directional variants each. The game's projection math is:

```
screenX = (worldX - worldY) * 80
screenY = (worldX + worldY) * 80 * 0.5
```

## Decision

Use Blender with an orthographic camera (X=60°, Y=0°, Z=45°) to model and render all game art. Source `.blend` files live in the `clear-ahead-assets` repo. Final PNGs are exported at 2x resolution (160px per world unit) and manually synced to the game repo.

Style references: Choro-Q / Penny Racers (chibi vehicle proportions), Theme Hospital (isometric world charm), iRacing (vehicle variety).

## Alternatives Considered

- **AI-generated art (Midjourney/DALL-E/Stable Diffusion) + manual cleanup**: Fastest for initial concepts but cannot reliably produce exact 2:1 isometric angles. Style/lighting/scale inconsistency across 50+ sprites would require extensive per-sprite rework. Transparency cleanup is tedious.
- **Hand-drawn (Procreate/Aseprite)**: Full creative control but requires drawing skill the developer doesn't have. Maintaining precise isometric angles freehand is error-prone.
- **Hybrid (AI concepts → Blender for consistency)**: Adds an extra step without clear benefit — Blender alone handles both ideation (viewport modeling) and final render.

## Consequences

- Isometric angle is guaranteed consistent: set the orthographic camera once, all renders match the game's projection
- 4 directional variants are trivial: rotate the model 90° and re-render
- Jelly material look is achievable with Principled BSDF (subsurface scattering + clear coat)
- Color variants are instant: duplicate material, change base color
- Blender Python scripting enables batch rendering automation
- Steepest learning curve of the alternatives, but the jelly style requires only simple geometry (rounded boxes, cylinders) — not photorealistic modeling
- The parametric/technical Blender workflow suits a developer's mindset better than freehand illustration
