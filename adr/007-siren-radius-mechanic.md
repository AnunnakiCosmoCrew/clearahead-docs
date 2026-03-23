# ADR-007: Siren radius as core game mechanic

**Status:** Proposed
**Date:** 2026-03-23
**Deciders:** Mert Ertugrul

## Context

Currently the ambulance siren is purely cosmetic — a visual glow and audio effect. Vehicle control modes are assigned statically in the level JSON (`playerDraggable`, `aiCooperative`, `aiTraffic`). The player can interact with any `playerDraggable` vehicle at any time, regardless of distance from the ambulance.

This creates two problems:

1. **No spatial tension.** The player can clear vehicles 10 segments ahead while the ambulance is stuck at the start. There's no urgency to prioritize nearby vehicles.

2. **Static control assignment doesn't match reality.** In real life, drivers yield for an ambulance because they hear the siren. A driver 2 km ahead doesn't know the ambulance is coming. The siren's reach determines who cooperates.

The siren should be the **core game mechanic** that determines which vehicles the player can interact with, creating a moving window of influence that travels with the ambulance.

## Decision

The ambulance siren has a **cone-shaped activation radius** that moves with it in real-time. Only vehicles inside this cone can be commanded by the player.

### Cone Geometry

The siren projects further ahead (where the ambulance is going) than behind:
- **Forward range:** 6 segments (graph hops from ambulance's current segment)
- **Rear range:** 2 segments
- **Lateral spread:** Vehicles on grouped segments (same `groupId`) of any in-cone segment are also in cone
- **Forward angle:** 90° (future tuning parameter)

### Detection

A `SirenRadius` class in the logic layer (pure Dart, no Flame) computes which vehicles are in the cone:

```dart
class SirenRadius {
  final int forwardRange;
  final int rearRange;

  Set<String> computeAffectedVehicles(RoadGraph graph);
}
```

Detection runs on a **0.5s timer** (not every frame) to keep it cheap. It performs a bounded BFS from the ambulance's segment, tagging segments as "ahead" or "behind" based on the ambulance's path direction.

### Vehicle State Transitions

When a vehicle enters the siren cone:
- State transitions from `autonomous` → `sirenAware`
- Vehicle slows down and begins honking/flashing (visual feedback for the player)
- Vehicle becomes commandable (tap+swipe)

When a vehicle leaves the siren cone (without being commanded):
- State transitions from `sirenAware` → `autonomous`
- Vehicle resumes normal speed and route

### Difficulty-Based Auto-Yield Fallback

If a vehicle is `sirenAware` but receives no player command:

| Difficulty | Behavior |
|------------|----------|
| Easy | Auto-yield after ~3 seconds (try lane change away from ambulance path, else move forward) |
| Medium | Auto-yield after ~5 seconds |
| Hard | **Never auto-yield.** Vehicle stays `sirenAware` indefinitely — deadlock. Player must handle every vehicle. |

This creates meaningful difficulty scaling from a single mechanic.

### Siren Radius as Tuning Lever

The siren parameters can vary by difficulty or level:

| Parameter | Easy | Medium | Hard |
|-----------|------|--------|------|
| Forward range | 8 segments | 6 segments | 4 segments |
| Rear range | 3 segments | 2 segments | 1 segment |
| Auto-yield delay | 3s | 5s | ∞ (never) |

### Visual Feedback

The siren cone is rendered as a subtle visual overlay on the road — a gradient that fades from the ambulance outward. This helps the player understand which vehicles they can currently command.

## Alternatives Considered

- **Circular radius (not cone):** Equal range in all directions. Rejected because the player needs more look-ahead (forward) than look-behind (auto-clear handles rear vehicles anyway). A cone matches real siren behavior — sound carries further in the direction of travel.

- **Static zones (pre-defined per level):** Mark specific segments as "siren zone" in the level JSON. Rejected because it removes the dynamic, ambulance-following nature of the mechanic. The tension comes from the cone moving with the ambulance.

- **Distance-based (Euclidean):** Use world-coordinate distance instead of graph hops. Rejected because graph hops better represent road topology — two segments might be close in world space but far apart in the road network (e.g., separated by a median with no crossing).

- **Always-on (current system):** Keep the status quo where all playerDraggable vehicles are always commandable. Rejected as the primary feedback from playtesting — no spatial tension, no urgency.

## Consequences

### What becomes easier

- **Difficulty scaling** — siren parameters + auto-yield delay create a natural difficulty curve without changing level layouts
- **Gameplay tension** — the moving cone creates urgency; player must prioritize vehicles about to enter/leave the cone
- **Thematic consistency** — the siren driving cooperation matches real-world ambulance behavior
- **Future mechanics** — siren range could be upgraded (power-ups), blocked by tunnels (environmental challenge), or amplified at intersections

### What becomes harder

- **Level design** — designers must consider siren range when placing vehicles; a vehicle outside the max siren range is never commandable. Mitigated by generous default ranges and the auto-yield fallback on easier difficulties.
- **Player understanding** — players must learn that they can only command nearby vehicles. Mitigated by visual siren cone overlay and tutorial levels.
- **Performance** — BFS from ambulance segment every 0.5s. With typical level sizes (< 20 segments), this is < 0.1ms — negligible.
