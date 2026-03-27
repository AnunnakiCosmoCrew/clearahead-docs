# ADR-010: Physics-based vehicle animation system

**Status:** Accepted
**Date:** 2026-03-27
**Deciders:** Mert Ertugrul

## Context

After implementing the Bezier curve maneuver system (ADR-006, Phase 6), vehicle movement looked mechanical despite following curved paths. Pre-rendered 32-direction isometric sprites snap between directions, and the constant-speed interpolation along curves made vehicles look like static images sliding across the screen rather than physical objects with weight and momentum.

Device testing feedback (clearahead-docs#1) confirmed this:

> "This car dragging logic is off, with this logic it feels like user moving cars like toy blocks."

The Bezier curves solved the *path* problem (vehicles turn instead of teleporting), but not the *feel* problem. We needed physics-based animation effects layered on top of the curve system to sell the illusion of mass, momentum, and ground contact.

## Decision

Layer **8 independent animation effects** onto the existing BezierManeuver system, all purely visual with zero impact on game logic. Each effect is a small, self-contained change to either `BezierManeuver` (curve timing/position) or `VehicleComponent` (rendering).

### Effects implemented

| Effect | Location | Mechanism |
|--------|----------|-----------|
| **Ease-in-out timing** | BezierManeuver | Back-ease curve replaces linear `t` — vehicles accelerate/decelerate with ~4% anticipation and overshoot |
| **Drop shadow** | VehicleComponent.render() | Semi-transparent blurred oval beneath sprite, offset for isometric light direction |
| **Smooth drag interpolation** | VehicleComponent.update() | During drag, visual position lerps toward logical target (18/sec exponential decay) instead of snapping |
| **Directional squash/stretch** | VehicleComponent + BezierManeuver | Stretch along dominant screen axis proportional to actual movement speed, via existing jelly spring system |
| **Suspension bounce** | BezierManeuver | Sinusoidal Y-offset (1.5px amplitude) with sin(pi*t) envelope, frequency scales with movement speed (8-20Hz) |
| **Anticipation/overshoot** | BezierManeuver easing | Back-ease-in-out curve: brief backward pull before moving, slight overshoot before settling |
| **Body tilt on turns** | VehicleComponent + BezierManeuver | Horizontal scale compression (up to 6%) during lateral movement, proportional to perpendicular tangent component |
| **Motion trail** | VehicleComponent + BezierManeuver | Ring buffer of 3 previous positions rendered as ghost sprites with decreasing opacity (12%, 7%, 3%) |

### Architecture principles

1. **All effects are cosmetic** — game logic (RoadGraph, MoveValidator, AmbulanceAI) is completely unaware of them
2. **Atomic state reservation unchanged** — target position reserved in graph before animation starts; visual effects are layered on top
3. **Each effect is independent** — can be disabled/tuned individually without affecting others
4. **BezierManeuver owns cleanup** — `onRemove()` override ensures all vehicle state (tilt, stretch, trail) resets even when animation is cancelled early by a new swipe
5. **No per-frame allocation in render path** — shadow paint, trail paints, glow paint are all cached as static fields
6. **Sprite direction guarded** — during anticipation (eased t < 0) and overshoot (t > 1), sprite direction is not updated to prevent wrong-way flipping

### Tuning constants

| Constant | Value | Location |
|----------|-------|----------|
| Car maneuver speed | 0.55 sec/unit | CommandResolver |
| Truck maneuver speed | 0.7 sec/unit | CommandResolver |
| Overshoot parameter | 1.0 (subtle) | BezierManeuver._overshoot |
| Bounce amplitude | 1.5 px | BezierManeuver._bounceAmplitude |
| Bounce base frequency | 8 Hz | BezierManeuver._baseBounceFrequency |
| Max motion stretch | 10% | VehicleComponent._maxMotionStretch |
| Max body tilt | 6% | VehicleComponent._maxTilt |
| Drag lerp speed | 18/sec | VehicleComponent._dragLerpSpeed |
| Shadow opacity | 25% | VehicleComponent._shadowPaint |
| Trail opacities | 12%, 7%, 3% | VehicleComponent._trailPaints |

## Alternatives Considered

- **Animated sprite sheets (2-4 frames per direction with spinning wheels):** Would be the single biggest visual improvement, but requires creating 32 directions x 4 frames x N vehicle variants = hundreds of new art assets. Deferred to a future phase (issue #157). The physics-based effects provide significant improvement without any new art.

- **Full skeletal animation (spine/rive):** Considered for wheel turn and body lean. Rejected because our sprites are pre-rendered isometric 3D — skeletal rigs would need to match the isometric perspective exactly, which is harder than just rendering more frames in Blender.

- **Particle-based effects only (dust, tire marks):** Dust is already implemented (`spawnVehicleDust`), but particles alone don't address the core problem of the vehicle sprite itself looking static. The physics effects (squash/stretch, bounce, tilt) modify the sprite directly.

- **Camera shake / screen effects:** Considered for impact feel but rejected — in a puzzle game, camera stability is important for the player to plan moves. Per-vehicle effects are better than global screen effects.

## Consequences

### What becomes easier

- **Tuning feel** — each effect has 1-2 constants that can be adjusted independently. The back-ease overshoot parameter alone controls how "cartoony" vs "realistic" the movement feels.
- **Adding future effects** — the pattern is established: BezierManeuver computes per-frame data, calls setter on VehicleComponent, VehicleComponent applies in render(). New effects follow the same pattern.
- **Animated sprite sheets** — when #157 is implemented, the existing direction-update system (which already handles 32 directions and tangent-based rotation) will automatically cycle frames during movement.

### What becomes harder

- **Debugging visual glitches** — with 8 layered effects, identifying which one causes a visual issue requires toggling them individually. Each effect should have a simple on/off mechanism for debugging.
- **Performance on low-end devices** — trail rendering adds 3 extra drawImage calls per moving vehicle per frame. Motion trail should be the first effect disabled if frame budget is tight.
- **Merge conflicts** — BezierManeuver.update() and VehicleComponent.render() are now hotspots that multiple effects modify. Changes to these methods will frequently conflict across feature branches.
