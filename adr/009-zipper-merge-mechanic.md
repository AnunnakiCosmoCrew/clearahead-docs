# ADR-009: Zipper merge mechanic with lateral offset

**Status:** Proposed
**Date:** 2026-03-24
**Deciders:** Mert Ertugrul

## Context

In real-life traffic jams, when an ambulance approaches from behind, vehicles on both sides of the road shift partially toward their edges — left lane cars shift left, right lane cars shift right. This opens a corridor down the middle of the road for the ambulance to drive through, straddling both lanes. This is commonly called a "zipper merge" or "Rettungsgasse" (rescue lane).

The current game model uses integer lanes — a vehicle is in lane 0 or lane 1, with no concept of being partially shifted within a lane. This prevents implementing the zipper mechanic, which is one of the most recognizable and satisfying real-world ambulance clearing behaviors.

The zipper is a core gameplay mechanic, not a cosmetic feature. It creates a unique puzzle dynamic: instead of fully clearing a lane, the player nudges vehicles to both sides, opening a corridor that's narrower than a full lane but wide enough for the ambulance.

## Decision

Add a `double lateralOffset` field to both `Vehicle` and `Ambulance`, creating a hybrid discrete+continuous position model:

### Lateral Offset

```
lateralOffset: -0.5  →  fully shifted left (toward left edge/median)
lateralOffset:  0.0  →  centered in lane (default)
lateralOffset: +0.5  →  fully shifted right (toward right edge/shoulder)
```

The integer `lane` field remains the vehicle's primary lane assignment for routing, AI, and basic queries. The `lateralOffset` adjusts the vehicle's physical position within that lane for collision detection and rendering.

### Collision with Lateral Offset

A vehicle with a large lateral offset partially occupies the adjacent lane:

```dart
/// A vehicle at lane 0 with lateralOffset +0.4 partially blocks lane 1.
/// A vehicle at lane 1 with lateralOffset -0.4 partially blocks lane 0.
/// Threshold: |lateralOffset| > 0.3 means partial occupation of adjacent lane.
bool vehicleBlocksLane(Vehicle v, int targetLane, int segmentLanes) {
  if (v.lane == targetLane) return true;
  if (v.lane == targetLane - 1 && v.lateralOffset > 0.3) return true;
  if (v.lane == targetLane + 1 && v.lateralOffset < -0.3) return true;
  return false;
}
```

### Ambulance Straddling

When both lanes have vehicles shifted to their edges (lateralOffset ≥ 0.3), the ambulance can drive through the gap by straddling:

```
ambulance.lane = 0, ambulance.lateralOffset = 0.5
→ straddling lanes 0 and 1
→ ambulance needs both lanes to be partially clear (vehicles shifted to edges)
```

The ambulance AI automatically detects straddling opportunities — if both adjacent lanes have vehicles shifted enough to create a gap, the ambulance drives through it.

### Player Input (Drag-Based, Pre-Tap+Swipe)

Using the existing drag system:
- **Small lateral drag (5-20px):** Partial shift within lane (lateralOffset changes)
- **Large lateral drag (>20px):** Full lane change (existing behavior)

This gives the player fine-grained control: a gentle nudge shifts a car to the side, a big swipe changes lanes entirely.

### Difficulty-Based Auto-Shift

| Difficulty | Behavior |
|------------|----------|
| No-brainer | Vehicles in siren range auto-shift to create zipper corridor |
| Easy | Vehicles auto-shift after ~3 seconds |
| Default | Player must manually shift each vehicle |

### Rendering

Vehicle position calculation adds lateral offset:
```dart
final laneCenterOffset = lanePerpendicularOffset(vehicle.lane, segment.lanes);
final laneWidth = worldScale * 0.7;
final totalOffset = laneCenterOffset + (vehicle.lateralOffset * laneWidth);
position += perpDir * totalOffset;
```

The vehicle visually slides within its lane, and the ambulance renders between lanes when straddling.

## Alternatives Considered

- **Half-lane model**: Split each lane into two half-lanes (left-half, right-half). Rejected because it doubles the graph complexity, complicates pathfinding, and doesn't model continuous sliding — vehicles would snap between half-lane positions.

- **Boolean "shifted" flag**: A simple `isShifted: bool` instead of continuous `lateralOffset`. Rejected because it doesn't allow variable shift amounts, doesn't support rendering smooth slide animations, and is less flexible for future mechanics.

- **Automatic-only zipper**: Skip player input entirely — vehicles auto-shift when siren reaches them. Rejected as the default because it removes player agency and puzzle depth. Kept as an option for easier difficulty levels.

- **Separate zipper lane**: Add a virtual "zipper corridor" between lanes. Rejected because it's an artificial abstraction that doesn't exist in real roads. The lateral offset model is more physically accurate and reusable.

## Consequences

### What becomes easier

- **Realistic traffic clearing** — the zipper is instantly recognizable and satisfying. Players who've seen real ambulance clearings will immediately understand the mechanic.
- **Puzzle depth** — the player must decide which vehicles to shift, how far, and in which direction. A half-shifted vehicle creates a different puzzle than a fully cleared lane.
- **Difficulty scaling** — auto-shift on easier difficulties creates a natural ramp. Hard mode requires manual precision.
- **Reusable model** — `lateralOffset` can be used for future mechanics: swerving, accident avoidance, parking maneuvers.

### What becomes harder

- **Collision complexity** — integer lane matching becomes partial lane occupation. `isRangeFree` and pathfinding must account for `lateralOffset`. Mitigated by a `vehicleBlocksLane()` helper with a clear threshold (0.3).
- **Pathfinding** — the ambulance must detect straddling opportunities and choose between full-lane and straddling paths. Mitigated by adding straddling as a secondary check after normal pathfinding.
- **Rendering precision** — vehicles at partial offsets must render smoothly without visual glitches. Mitigated by smooth animation via Bézier interpolation of `lateralOffset`.
- **Testing surface area** — every collision test gains a `lateralOffset` dimension. Mitigated by dedicated zipper test suite.
