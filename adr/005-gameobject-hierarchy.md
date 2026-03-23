# ADR-005: GameObject base class hierarchy with SOLID principles

**Status:** Proposed
**Date:** 2026-03-23
**Deciders:** Mert Ertugrul

## Context

The current codebase has `Vehicle`, `Ambulance`, and `Obstacle` as unrelated model classes extending `Equatable`. They share no common base class. This causes several problems:

1. **Collision detection is special-cased per type.** `MoveValidator` checks vehicles vs vehicles, `RoadGraph.isRangeFree` checks vehicles + ambulance + obstacles as separate lists, and shoulder vehicles are skipped in main-lane checks. This leads to bugs — a vehicle on a shoulder and a vehicle in the right lane can overlap because they're checked in isolation.

2. **No physical-space concept.** There's no universal rule that "two objects can't occupy the same space." Each collision check must be manually coded for each pair of types.

3. **Adding new object types requires modifying every collision checker.** If we add cones, garbage bins, or other environment objects, we must update `MoveValidator`, `RoadGraph.isRangeFree`, `AmbulanceAI._isLaneClearForAmbulance`, etc. This violates the Open/Closed Principle.

4. **Ambulance and Vehicle share significant structure** (segmentId, offset, lane, length, moveTo) but are completely separate classes. This violates DRY and makes polymorphic handling impossible.

Testing revealed a concrete bug: a blue car on a shoulder overlaps with a red car in the right lane. This is because shoulder vehicles are excluded from lane collision checks, but they still occupy physical space adjacent to the lane.

## Decision

Introduce a `GameObject` abstract base class and restructure the model hierarchy following SOLID principles:

```
GameObject (abstract)
├── id, segmentId, offset, lane, length
├── collisionBounds → occupancy range on segment
├── collidesWith(other) → bool
│
├── Vehicle (abstract, extends GameObject)
│   ├── speed, destination, state, currentRoute
│   ├── Car (length=1, maneuverSpeed=1.0)
│   ├── Truck (length=2, maneuverSpeed=0.6)
│   └── Ambulance (length=2, acceleration, sirenRadius)
│
└── StaticObject (abstract, extends GameObject)
    ├── Tree, Barrier, Cone, Container, GarbageBin, Building
```

Shared behaviors are expressed as mixins:
- `Collidable` — collision bounds and intersection check
- `Movable` — speed, route, route recalculation
- `Commandable` — state machine, receiving/executing player commands

Collisions are **prevented by design, not detected after the fact**. Rather than building a separate `CollisionSystem` that reacts to overlaps, the `MoveValidator` is strengthened to validate against ALL GameObjects before any move is applied. If you can't reach an invalid state, you don't need to detect it.

`MoveValidator` becomes the single source of truth for "can this object be at this position":
- Validates against all vehicles, the ambulance, and all static objects on the segment
- Checks shoulder vs lane physical overlap (fixes the overlap bug)
- Enforces segment direction constraints
- For Bézier curve maneuvers: the target position is reserved atomically in RoadGraph before the animation starts — the curve is purely visual

### Key occupancy rules
- Same lane, overlapping offset → blocked (cannot move there)
- Different lanes, same offset → allowed (lanes are separate)
- Lane vehicle at offset N, shoulder vehicle at offset N → blocked (physical overlap)
- Any vehicle vs obstacle at same lane/offset → blocked

### Immutability preserved
All model classes remain immutable (extend Equatable, return new instances on mutation). The hierarchy adds shared structure, not shared mutable state.

## Alternatives Considered

- **Keep flat structure, add collision check function:** Add a standalone `collides(a, b)` function without changing the class hierarchy. Rejected because it still requires type-checking every pair and doesn't solve the DRY problem between Vehicle and Ambulance.

- **Component-entity-system (ECS):** Use a full ECS pattern where GameObjects are entity IDs with attached components. Rejected as overengineering for our scale (~20 objects per level). The class hierarchy is simpler and more readable.

- **Protocol/interface only (no base class):** Define `GameObject` as a Dart abstract interface, no shared implementation. Rejected because Vehicle and Ambulance share too much implementation (segmentId, offset, lane, length, moveTo pattern) — an abstract class with shared fields is more practical.

## Consequences

### What becomes easier

- **Collision prevention by design** — MoveValidator checks all GameObjects, fixing the shoulder overlap bug. Invalid states are impossible, not just detected.
- **Adding new object types** — just extend `StaticObject` or `Vehicle`; MoveValidator handles them through the GameObject abstraction automatically
- **Polymorphic handling** — methods can accept `GameObject` and work with any type (rendering, serialization, graph queries)
- **Testing** — occupancy rules tested once in MoveValidator, not scattered across 5 files

### What becomes harder

- **Migration** — every file that touches Vehicle, Ambulance, or Obstacle must be updated. The refactor touches models, logic, game, and tests. Mitigated by keeping backward-compatible level JSON parsing.
- **Equatable with inheritance** — Equatable `props` must include superclass fields. Requires careful implementation to avoid breaking equality checks.
- **Serialization** — LevelLoader must map JSON to the correct subclass (Car vs Truck vs Ambulance). Currently uses a `VehicleType` enum; this becomes a factory pattern.
