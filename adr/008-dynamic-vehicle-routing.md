# ADR-008: Dynamic vehicle routing with destination-based pathfinding

**Status:** Proposed
**Date:** 2026-03-23
**Deciders:** Mert Ertugrul

## Context

The current `VehicleRoute` model stores a **fixed, ordered list of segment IDs** that an AI vehicle follows. This was designed for `aiTraffic` vehicles that follow a predetermined path through the level.

With the new vehicle command system (ADR-006), ALL vehicles become AI-driven — they all have routes. The fixed route model breaks down because:

1. **After being commanded**, a vehicle is at a new position (different segment, lane, or shoulder). Its original fixed route is now invalid — it may not even be on any segment in the route list.

2. **Route recalculation is needed** when a vehicle returns to autonomous driving after the ambulance passes. It needs to find a new path from its current (potentially unexpected) position to its destination.

3. **Fixed routes are fragile** — if any segment in the route becomes blocked (by obstacles, other vehicles, or road changes), the vehicle has no fallback.

4. **Level authoring burden** — every vehicle needs a manually specified segment-by-segment route. With dynamic routing, level designers only specify origin and destination.

## Decision

Replace fixed `VehicleRoute` with **destination-based dynamic pathfinding**:

### Vehicle Model Change

```dart
// Before
class Vehicle {
  final VehicleRoute? route;  // fixed segment list + exit node
}

// After
class Vehicle {
  final String destination;  // target node ID
  List<String>? currentRoute;  // dynamically calculated, cached
}
```

### VehicleRouter

A new `VehicleRouter` class (pure Dart, logic layer) calculates routes using the same BFS algorithm as `AmbulanceAI.findPath()`:

```dart
class VehicleRouter {
  static List<String>? findRoute(
    String fromSegmentId,
    String towardsNodeId,
    String destinationNodeId,
    RoadGraph graph,
  );
}
```

This reuses the existing BFS infrastructure — no new pathfinding algorithm needed.

### Route Recalculation Triggers

| Event | Action |
|-------|--------|
| Level load | Calculate initial route for every vehicle |
| Vehicle state → `returning` | Recalculate from current position to destination |
| Vehicle crosses segment boundary | Verify still on cached route; recalculate if deviated |
| Segment blockage change | Mark affected vehicles' routes as dirty |

### Dirty Flag Pattern

Same pattern as ambulance pathfinding (ADR-002): a `_routeDirty` flag per vehicle prevents unnecessary recalculation. Only vehicles whose routes are invalidated (segment they need becomes blocked) recalculate.

### Route recalculation is batched on a timer (not per-frame) to prevent cascading recalculations when multiple vehicles move simultaneously.

### Level JSON Change

```json
// Before
{
  "id": "car_red",
  "route": { "segments": ["seg_a", "seg_b", "seg_c"], "exitNode": "E" }
}

// After
{
  "id": "car_red",
  "destination": "E"
}
```

The `route` field is still parsed for backward compatibility but `destination` takes precedence.

### Vehicle Exit

When a vehicle reaches its destination node:
1. Play drive-off animation (cosmetic, detached sprite)
2. Remove from RoadGraph immediately (same pattern as auto-clear, ADR-003 in clear-ahead repo / ADR-007 original)
3. Future: optionally spawn replacement vehicle at entry node (traffic flow feature, out of scope)

## Alternatives Considered

- **Keep fixed routes, add route patching:** When a vehicle is displaced by a player command, find the nearest segment on its original route and resume from there. Rejected because it doesn't handle cases where the vehicle is moved to a segment that's not adjacent to any route segment (e.g., across a median crossing). Full recalculation is simpler and more robust.

- **Waypoint-based routing:** Vehicle has a list of waypoint nodes (not segments) and pathfinds between consecutive waypoints. Rejected as unnecessary complexity — destination-based BFS produces the same result without manual waypoints.

- **A* instead of BFS:** Use A* with heuristic for potentially faster pathfinding. Rejected because our graph sizes are tiny (< 20 segments) and BFS is already < 0.1ms. A* adds complexity without measurable benefit.

- **Pre-computed route tables:** At level load, compute all-pairs shortest paths and look up routes instantly. Rejected as premature optimization — with < 20 segments, BFS is effectively instant. Pre-computation also doesn't help when the graph changes dynamically (vehicles blocking segments).

## Consequences

### What becomes easier

- **Level authoring** — just specify origin segment and destination node per vehicle; no manual route crafting
- **Post-command behavior** — vehicle automatically finds its way back to destination from any position
- **Dynamic traffic** — vehicles can reroute around blockages, creating more realistic traffic behavior
- **Ambulance/vehicle parity** — both use the same BFS pathfinding, reducing code duplication

### What becomes harder

- **Route prediction** — with fixed routes, you could predict exactly where a vehicle would go. Dynamic routing means vehicles might take unexpected paths after being displaced. Mitigated by showing route indicators (future UX feature).
- **Performance with many vehicles** — N vehicles each doing BFS on route dirty = O(N × graph_size). With N < 15 and graph < 20 segments, this is still < 1ms total. Mitigated further by dirty flags and timer batching.
- **Testing** — fixed routes were deterministic; dynamic routes depend on graph state. Mitigated by testing with known graph topologies where the optimal route is predictable.
