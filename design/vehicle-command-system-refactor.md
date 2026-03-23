# Vehicle Command System & Realistic Movement Refactor

**Status:** Proposed
**Date:** 2026-03-23
**Authors:** Mert Ertugrul, Claude
**Related ADRs:** ADR-005, ADR-006, ADR-007, ADR-008
**Related Issues:** [Test findings — clearahead-docs#1](https://github.com/AnunnakiCosmoCrew/clearahead-docs/issues/1)

---

## 1. Vision

The current drag-and-snap movement model feels like sliding toy blocks — it's not fun and doesn't match the game's theme of being a helicopter dispatcher coordinating real traffic. This refactor transforms the game from a **sliding block puzzle** into a **real-time traffic command game** where:

- Every vehicle is AI-driven with its own route and destination
- The player's role is **dispatcher**, not driver
- The ambulance siren activates nearby vehicles into a cooperative state
- The player issues **directional commands** (tap + swipe) to cooperative vehicles
- Vehicles execute **realistic curved maneuvers** (Bézier curves)
- After the ambulance passes, vehicles resume their routes autonomously

This is the single biggest gameplay pivot since the move from grid to graph topology (ADR-001).

---

## 2. Current State vs Target

| Aspect | Current | Target |
|--------|---------|--------|
| Movement model | Drag-and-snap (instant teleport) | AI-driven with Bézier curve maneuvers |
| Player input | Drag vehicle from A to B | Tap vehicle + swipe direction |
| Vehicle control | 3 modes (playerDraggable, aiCooperative, aiTraffic) | All AI-driven; state machine (autonomous → siren-aware → commanded → returning) |
| Object hierarchy | Vehicle and Obstacle are unrelated classes | All inherit from GameObject base |
| Collision | Lane-aware, special-cased per type | Universal: all GameObjects have collision bounds |
| Truck restrictions | Can't use shoulders or cross median | Can do everything cars can, just slower |
| Shoulder vehicles | Locked (can't move longitudinally) | Can move longitudinally, slower (off-road penalty) |
| Route system | Fixed VehicleRoute (segment list) | Destination-based with dynamic pathfinding |
| Siren | Visual/audio only | Core game mechanic with activation radius |
| No-input behavior | N/A | Difficulty-dependent (easy: auto-yield; hard: deadlock) |

---

## 3. GameObject Hierarchy (ADR-005)

### Design Principles

Follow SOLID principles:
- **S**ingle Responsibility: Each class has one reason to change
- **O**pen/Closed: Extend via new subclasses, not modifying existing ones
- **L**iskov Substitution: Any GameObject can be used where GameObject is expected
- **I**nterface Segregation: Movable, Collidable, Commandable as separate interfaces/mixins
- **D**ependency Inversion: Logic depends on abstractions (GameObject), not concrete types

### Class Hierarchy

```
GameObject (abstract)
├── position: Vector2
├── size: Vector2
├── collisionBounds: Rect
├── collidesWidth(other: GameObject): bool
│
├── Vehicle (abstract)
│   ├── id: String
│   ├── segmentId: String
│   ├── offset: int
│   ├── lane: int
│   ├── length: int
│   ├── speed: double
│   ├── destination: String (nodeId)
│   ├── state: VehicleState
│   ├── currentRoute: List<String>?
│   ├── moveTo(...) → Vehicle
│   │
│   ├── Car
│   │   ├── length = 1
│   │   └── maneuverSpeed = 1.0 (base)
│   │
│   ├── Truck
│   │   ├── length = 2
│   │   └── maneuverSpeed = 0.6 (slower)
│   │
│   └── Ambulance
│       ├── length = 2
│       ├── acceleration = 0.3
│       ├── sirenRadius: SirenRadius
│       └── exitNodeId: String
│
└── StaticObject (abstract)
    ├── Tree
    ├── Barrier
    ├── Cone
    ├── Container
    ├── GarbageBin
    └── Building
```

### Interfaces (Mixins)

```dart
mixin Movable {
  double get speed;
  List<String>? get currentRoute;
  void recalculateRoute(RoadGraph graph);
}

mixin Collidable {
  Rect get collisionBounds;
  bool collidesWith(Collidable other);
}

mixin Commandable {
  VehicleState get state;
  void receiveCommand(Direction direction, double magnitude);
  void executeManeuver();
}
```

### Key Rule

**No two GameObjects can occupy the same physical space** — this is the universal collision rule that fixes the shoulder overlap bug. Collision detection runs against ALL GameObjects, not type-specific checks.

---

## 4. Vehicle State Machine (ADR-006)

Every vehicle (except the ambulance) follows this state machine:

```
                    siren reaches vehicle
    ┌──────────┐  ─────────────────────>  ┌──────────────┐
    │AUTONOMOUS│                          │ SIREN-AWARE  │
    │          │  <─────────────────────   │              │
    │ Driving  │   siren out of range     │ Slows down   │
    │ own route│   (no command given)     │ Honks/flashes│
    └──────────┘                          └──────┬───────┘
                                                 │
                                          player gives
                                          direction command
                                                 │
                                                 ▼
    ┌──────────┐   maneuver complete       ┌──────────────┐
    │ WAITING  │  <─────────────────────   │  EXECUTING   │
    │          │                           │              │
    │ Holds    │                           │ Bézier curve │
    │ position │                           │ maneuver     │
    └────┬─────┘                           └──────────────┘
         │
         │ ambulance passes
         │ (2+ segments behind)
         ▼
    ┌──────────┐
    │RETURNING │
    │          │
    │ Recalc   │
    │ route to │
    │ dest     │
    └────┬─────┘
         │
         │ back on valid route
         ▼
    ┌──────────┐
    │AUTONOMOUS│
    └──────────┘
```

### States

| State | Behavior | Player can command? |
|-------|----------|---------------------|
| `autonomous` | Drives own route at normal speed | No |
| `sirenAware` | Slows down, honks/flashes, waits for command | Yes |
| `executing` | Performing Bézier curve maneuver | No (in progress) |
| `waiting` | Holding position after maneuver | Yes (can re-command) |
| `returning` | Recalculating route, merging back | No |

### Difficulty-Based Fallback (No Player Input)

When a vehicle is in `sirenAware` state and the player gives no command:

| Difficulty | Behavior |
|------------|----------|
| No-brainer | After ~3 seconds, vehicle auto-yields (attempts basic lane change or forward move) |
| Easy | After ~5 seconds, vehicle auto-yields |
| Default | Vehicle stays in `sirenAware` indefinitely — **deadlock**. Player MUST handle every vehicle. |

The auto-yield fallback reuses the existing cooperative AI logic (try lane change away from ambulance path, else move forward).

---

## 5. Siren Radius Mechanic (ADR-007)

### Concept

The ambulance siren has a **cone-shaped activation radius** that moves with it. Only vehicles inside this cone can be commanded by the player. This creates the core tension: you can only interact with vehicles the siren has reached.

### Cone Shape

```
                    ╱─────────╲
                  ╱             ╲
                ╱    FORWARD     ╲
              ╱     CONE RANGE    ╲
            ╱      (6 segments)     ╲
          ╱                           ╲
        ╱               ↑               ╲
      ╱─────────────────┼─────────────────╲
                    AMBULANCE
      ╲─────────────────┼─────────────────╱
        ╲               ↓               ╱
          ╲  REAR (2 segments)        ╱
            ╲                       ╱
              ╲───────────────────╱
```

### Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| `forwardRange` | 6 segments | Distance ahead of ambulance |
| `rearRange` | 2 segments | Distance behind ambulance |
| `lateralSpread` | Adjacent grouped segments | Covers parallel lanes |
| `forwardAngle` | 90° | Cone opening angle |

### Detection Algorithm

```
For each vehicle:
  1. Compute graph distance (BFS hops) from ambulance segment to vehicle segment
  2. Determine if vehicle is ahead or behind ambulance (based on path direction)
  3. If ahead AND distance ≤ forwardRange → in cone
  4. If behind AND distance ≤ rearRange → in cone
  5. If on grouped segment (same groupId) of an in-cone segment → in cone
  6. Otherwise → out of cone
```

### Performance

Siren detection runs on a **0.5s timer** (not every frame). It maintains a set of `sirenAwareVehicleIds`. When a vehicle enters the set, its state transitions to `sirenAware`. When it leaves (and wasn't commanded), it reverts to `autonomous`.

---

## 6. Command Input System (ADR-006)

### Replacing Drag with Tap+Swipe

The current drag system (DragCallbacks on ClearAheadGame) is replaced with a two-step command:

1. **Tap** a `sirenAware` or `waiting` vehicle → selects it (visual highlight, glow effect)
2. **Swipe** in any direction → issues directional command

### Swipe Direction Mapping

The swipe vector is decomposed relative to the vehicle's current segment direction. **Pure perpendicular (left/right) commands are excluded** — real vehicles can only change lateral position by steering while moving forward or backward (only front wheels turn). All lateral movement is a diagonal arc.

| Swipe | Command | Meaning |
|-------|---------|---------|
| Along segment (forward) | `forward` | Move forward on segment |
| Along segment (backward) | `backward` | Reverse on segment |
| Diagonal (forward-right) | `forwardRight` | Lane change right, shoulder pull-off, median crossing (forward arc) |
| Diagonal (forward-left) | `forwardLeft` | Lane change left, shoulder pull-off, median crossing (forward arc) |
| Diagonal (backward-right) | `backwardRight` | Back into shoulder/lane (reverse arc) |
| Diagonal (backward-left) | `backwardLeft` | Back into shoulder/lane (reverse arc) |

This gives **6 directions** with 60° angular zones each, providing generous hit targets on phone screens.

### Swipe Magnitude

- **Short swipe** (< 1 world unit) → minimal movement (1 offset + lane change)
- **Medium swipe** (1-3 world units) → moderate movement
- **Long swipe** (> 3 world units) → maximum valid movement in that direction

The vehicle finds the **nearest valid position** in the commanded direction, clamped by obstacles, other vehicles, and segment boundaries.

### Command Resolution

When a command is issued:

```
1. Determine target position:
   - Direction → candidate positions along that vector
   - Find furthest valid position (no collision, within bounds)
   - Clamp to segment/lane boundaries

2. Plan maneuver:
   - Calculate Bézier curve from current position to target
   - Estimate duration based on distance and vehicle type (trucks slower)

3. Execute:
   - Vehicle state → EXECUTING
   - Animate along Bézier curve
   - On completion → state = WAITING
```

---

## 7. Realistic Vehicle Movement — Bézier Curves

### The Problem

Current movement instantly teleports vehicles from one position to another. Real cars don't slide sideways — they turn.

### The Solution

All vehicle maneuvers follow a **cubic Bézier curve** from start to end position. The control points create a natural turning arc.

### Curve Construction

```
Start: (x₀, y₀) — current world position
End:   (x₃, y₃) — target world position

All lateral movement is diagonal (forward+lateral or backward+lateral),
since real vehicles can only steer while moving. There is no pure
sideways maneuver — even a "lane change" requires forward motion.

For a lane change (forward + lateral arc):
  P0 = (x₀, y₀)                      — start
  P1 = (x₀ + dx*0.4, y₀ + dy*0.1)   — front wheels turn, car begins arc
  P2 = (x₃ - dx*0.4, y₃ - dy*0.1)   — car aligns with target lane
  P3 = (x₃, y₃)                      — end

For a shoulder pull-off (forward + lateral arc, tighter curve):
  P0 = (x₀, y₀)
  P1 = (x₀ + dx*0.3, y₀ + dy*0.15)  — sharper initial turn toward shoulder
  P2 = (x₃ - dx*0.3, y₃ - dy*0.15)  — aligning with shoulder
  P3 = (x₃, y₃)

For a pure forward/backward move (straight line):
  P0 = (x₀, y₀)
  P1 = lerp(P0, P3, 0.33)            — straight interpolation
  P2 = lerp(P0, P3, 0.66)
  P3 = (x₃, y₃)
```

### Architecture Separation

| Layer | Responsibility |
|-------|---------------|
| **Logic** (pure Dart) | Validates target position, returns `ManeuverPlan(startOffset, endOffset, startLane, endLane, startSegment, endSegment)` |
| **Game** (Flame) | Converts ManeuverPlan to Bézier curve using world coordinates, animates vehicle along curve, updates visual rotation |

The logic layer **never** deals with Bézier curves or pixel positions. It only validates whether a target (segment, offset, lane) is reachable and collision-free.

### Maneuver Duration

| Vehicle | Base duration | Notes |
|---------|--------------|-------|
| Car | 0.4s per unit distance | Nimble |
| Truck | 0.7s per unit distance | Heavier, wider turning radius |

During execution, the vehicle's collision bounds follow the curve — intermediate collision detection prevents clipping through other vehicles mid-maneuver.

### Vehicle Rotation During Maneuver

The vehicle sprite rotates to follow the Bézier curve tangent:

```
At parameter t along the curve:
  tangent = B'(t)  — first derivative of Bézier
  angle = atan2(tangent.y, tangent.x)
  vehicle.rotation = angle
```

This makes the car visually "turn" through the maneuver — front swings first, body follows.

---

## 8. Dynamic Route Calculation (ADR-008)

### Current System

`VehicleRoute` stores a fixed list of segment IDs. AI vehicles follow this sequence rigidly.

### New System

Each vehicle has a `destination: String` (a node ID). Routes are calculated dynamically using BFS pathfinding — the same algorithm the ambulance uses.

### When Routes Recalculate

| Event | Action |
|-------|--------|
| Level load | Initial route calculated for all vehicles |
| Vehicle state → `returning` | Recalculate from current position to destination |
| Segment becomes blocked | Affected vehicles recalculate (dirty flag) |
| Vehicle crosses segment boundary | Check if still on route; recalculate if deviated |

### Route Calculation

```dart
class VehicleRouter {
  /// Calculates shortest path from vehicle's current position to destination.
  /// Reuses the same BFS logic as AmbulanceAI.findPath().
  static List<String>? findRoute(
    String fromSegmentId,
    String towardsNodeId,
    String destinationNodeId,
    RoadGraph graph,
  );
}
```

### Vehicle Movement Along Route

Autonomous vehicles advance along their route on a timer:

| Vehicle state | Tick rate | Behavior |
|---------------|-----------|----------|
| `autonomous` | 0.5s | Advance 1 offset; at segment end, transition to next segment in route |
| `returning` | 0.3s | Faster return to route (driver eager to resume) |
| `sirenAware` | 1.0s | Slows down while waiting for command |

### Vehicle Exit

When a vehicle reaches its destination node, it exits the graph:
1. Play a drive-off animation (cosmetic)
2. Remove from RoadGraph
3. Optionally spawn a new vehicle at an entry node (traffic flow — future feature)

---

## 9. Universal Collision Detection

### Current Problem

Collision is special-cased:
- `MoveValidator` checks vehicles vs vehicles (lane-aware)
- `RoadGraph.isRangeFree` checks vehicles + ambulance + obstacles separately
- Shoulder vehicles are skipped in main-lane checks
- No collision between shoulder vehicles and lane vehicles (the overlap bug)

### New System

All GameObjects participate in a unified collision system:

```dart
class CollisionSystem {
  /// Check if a target rect collides with any existing GameObject.
  /// Respects segment topology (only checks objects on same/adjacent segments).
  bool isPositionFree(
    String segmentId,
    int offset,
    int length,
    int lane,
    bool onShoulder,
    ShoulderSide? shoulderSide,
    RoadGraph graph, {
    String? excludeId,
  });

  /// Get all objects that would collide at a given position.
  List<GameObject> findCollisions(
    String segmentId,
    int offset,
    int length,
    int lane,
    bool onShoulder,
    ShoulderSide? shoulderSide,
    RoadGraph graph, {
    String? excludeId,
  });
}
```

### Collision Rules

| Object A | Object B | Collision? |
|----------|----------|------------|
| Vehicle (lane) | Vehicle (same lane) | Yes — blocked |
| Vehicle (lane) | Vehicle (different lane) | No — lanes are separate |
| Vehicle (lane) | Vehicle (shoulder, same side & overlapping offset) | Yes — physical overlap |
| Vehicle (any) | Obstacle (same lane or all-lane) | Yes — blocked |
| Vehicle (shoulder) | Obstacle (shoulder) | Yes — blocked |
| Ambulance | Vehicle (same lane) | Yes — blocked |
| Ambulance | Obstacle (same lane) | Yes — blocked |

### Intermediate Collision (During Maneuvers)

While a vehicle is executing a Bézier curve maneuver, its collision bounds are checked at discrete intervals along the curve (every 0.1 parameter step). If a collision is detected mid-maneuver:

1. Vehicle stops at last collision-free position
2. State remains `executing` but target is shortened
3. Player can re-command after the shortened maneuver completes

---

## 10. Rule Changes

### Trucks on Shoulders

**Before:** Trucks cannot use shoulders.
**After:** Trucks CAN use shoulders, but:
- Maneuver takes 1.75x longer (0.7s vs 0.4s per unit)
- Truck occupies 2 units of shoulder space
- Shoulder must have 2 consecutive unblocked units

### Trucks Crossing Median

**Before:** Trucks cannot cross median (too wide).
**After:** Trucks CAN cross median, but:
- Crossing width must be ≥ 2 (truck length)
- Maneuver takes 1.75x longer
- Crossing animation shows truck carefully navigating the gap

### Shoulder Movement

**Before:** Shouldered vehicles are locked (cannot move longitudinally).
**After:** Shouldered vehicles CAN move longitudinally, but:
- Speed reduced to 0.5x (off-road penalty, rough terrain)
- Still subject to shoulder obstacles blocking the path
- Can return to main road if lane is clear

### Direction Enforcement

**Before:** Vehicles can be dragged against segment direction.
**After:** Vehicles can only move in the segment's allowed direction. Backward movement only on bidirectional segments.

---

## 11. Implementation Phases

Each phase is independently shippable and testable.

### Phase 1: Foundation — GameObject Hierarchy & Universal Collision

**Scope:** Refactor models to use inheritance. Fix collision bugs.
**ADR:** ADR-005

- [ ] Create `GameObject` abstract base class with position, size, collision bounds
- [ ] Create `Vehicle` abstract class extending GameObject
- [ ] Refactor `Car`, `Truck` as Vehicle subclasses (extract from VehicleType enum)
- [ ] Refactor `Ambulance` to extend Vehicle
- [ ] Create `StaticObject` abstract class extending GameObject
- [ ] Refactor `Obstacle` to extend StaticObject
- [ ] Implement `CollisionSystem` with universal collision checks
- [ ] Replace all special-cased collision code in MoveValidator and RoadGraph
- [ ] **Fix:** Shoulder overlap bug (vehicles on shoulder collide with lane vehicles at same offset)
- [ ] **Fix:** Opposite direction movement (enforce segment direction in MoveValidator)
- [ ] Update RoadGraph to work with new type hierarchy
- [ ] Update all tests

**Files changed:** `lib/models/` (new base classes), `lib/logic/move_validator.dart`, `lib/logic/road_graph.dart`, `lib/game/components/`

### Phase 2: Dynamic Routes & Vehicle Destination

**Scope:** Replace fixed VehicleRoute with destination-based pathfinding.
**ADR:** ADR-008

- [ ] Add `destination: String` field to Vehicle
- [ ] Create `VehicleRouter` class (reuses BFS from AmbulanceAI)
- [ ] Replace `VehicleRoute.segments` with dynamically calculated routes
- [ ] Update TrafficAI to use VehicleRouter for route following
- [ ] Implement route recalculation on position change
- [ ] Update LevelLoader to parse `destination` from JSON
- [ ] Update level_001.json with vehicle destinations
- [ ] Test route recalculation scenarios

**Files changed:** `lib/models/vehicle.dart`, `lib/logic/vehicle_router.dart` (new), `lib/logic/traffic_ai.dart`

### Phase 3: Siren Radius & Vehicle State Machine

**Scope:** Siren as game mechanic. Vehicle states replace control modes.
**ADR:** ADR-007

- [ ] Create `SirenRadius` class with cone-shaped detection
- [ ] Create `VehicleState` enum/sealed class with state machine transitions
- [ ] Replace `VehicleControl` enum with state-based system
- [ ] Implement siren detection on 0.5s timer
- [ ] Vehicle visual feedback: honking, flashing when sirenAware
- [ ] Vehicle slows down in sirenAware state
- [ ] Implement difficulty-based auto-yield fallback
- [ ] Update ClearAheadGame to manage siren radius
- [ ] Render siren radius cone visually (subtle overlay)
- [ ] Update tests

**Files changed:** `lib/models/vehicle_state.dart` (new), `lib/logic/siren_radius.dart` (new), `lib/logic/traffic_ai.dart`, `lib/game/clear_ahead_game.dart`

### Phase 4: Command Input System (Tap+Swipe)

**Scope:** Replace drag with tap+swipe directional commands.
**ADR:** ADR-006

- [ ] Remove DragCallbacks from ClearAheadGame
- [ ] Implement tap detection on vehicles (only sirenAware/waiting vehicles respond)
- [ ] Implement swipe gesture recognition with direction decomposition
- [ ] Swipe direction relative to segment angle (not screen)
- [ ] Short/medium/long swipe → movement magnitude
- [ ] Command resolution: find nearest valid target position in direction
- [ ] Create `ManeuverPlan` model (start, end, duration)
- [ ] Vehicle state transition: sirenAware → executing on command
- [ ] Selection visual (glow, highlight)
- [ ] Deselection on tap-away or second tap
- [ ] Update tests

**Files changed:** `lib/game/clear_ahead_game.dart`, `lib/game/components/vehicle_component.dart`, `lib/logic/command_resolver.dart` (new), `lib/models/maneuver_plan.dart` (new)

### Phase 5: Realistic Movement — Bézier Curves

**Scope:** Vehicles execute commands with curved, animated maneuvers.

- [ ] Create `BezierManeuver` class that generates curve from ManeuverPlan
- [ ] Control point calculation for different maneuver types (lane change, shoulder, forward, diagonal)
- [ ] Vehicle sprite follows curve tangent (rotation)
- [ ] Duration varies by vehicle type (truck 1.75x slower)
- [ ] Intermediate collision detection along curve
- [ ] Vehicle stops at last safe position if collision detected mid-curve
- [ ] State transition: executing → waiting on completion
- [ ] Camera follows maneuver smoothly
- [ ] Update tests

**Files changed:** `lib/game/effects/bezier_maneuver.dart` (new), `lib/game/components/vehicle_component.dart`, `lib/game/rendering_utils.dart`

### Phase 6: Rule Relaxation & Polish

**Scope:** Enable trucks on shoulders/median. Shoulder longitudinal movement. Direction enforcement.

- [ ] Remove truck restrictions from MoveValidator (shoulders + median)
- [ ] Add maneuver speed multiplier for trucks (1.75x duration)
- [ ] Enable longitudinal movement on shoulders (0.5x speed)
- [ ] Enforce segment direction in command resolver (no backward on forward-only)
- [ ] Update level_001.json to showcase new capabilities
- [ ] Design 2-3 new levels that use the new mechanics
- [ ] Update tests for relaxed rules
- [ ] Polish animations, particle effects, sound

---

## 12. Migration Strategy

### Backward Compatibility

Each phase maintains backward compatibility with existing level JSON:
- Phase 1: `VehicleType` enum still works, mapped to Car/Truck subclasses internally
- Phase 2: `route` field in JSON still parsed, but `destination` takes precedence if present
- Phase 3: `control` field mapped to initial vehicle state
- Phase 4-6: Input changes don't affect level format

### Testing Strategy

- **Logic layer:** Unit tests for every phase (pure Dart, fast)
- **Collision system:** Dedicated test suite covering all collision matrix combinations
- **Siren radius:** Unit tests with known graph topologies
- **Command resolution:** Unit tests for direction → target mapping
- **Integration:** Manual playtesting after each phase (use clearahead-docs testing checklists)

### Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Bézier maneuvers feel sluggish | Tune duration constants; allow cancellation |
| Siren radius too small/large | Make it a tuning constant; adjust per difficulty |
| Tap+swipe less intuitive than drag | Phase 4 is self-contained; can A/B test or revert |
| Universal collision too expensive | Spatial partitioning by segment (already natural) |
| Route recalculation cascade | Dirty flag + timer batching (same pattern as ambulance BFS) |

---

## 13. Open Questions

1. **Siren radius visual:** Should the player see the siren cone on screen? A subtle overlay could help them understand which vehicles they can command.
2. **Multiple selections:** Can the player command multiple vehicles simultaneously, or one at a time?
3. **Command queue:** If a vehicle is executing, can the player queue the next command?
4. **Traffic spawning:** Should new vehicles spawn at entry nodes to maintain traffic flow? (Future feature, not in this refactor.)
5. **Undo:** Should the player be able to cancel a command mid-execution? (Swipe back?)
