# ADR-004: Multi-lane road model with obstacles and median crossings

**Status:** Accepted
**Date:** 2026-03-20
**Deciders:** Mert Ertugrul

## Context

The original road model used single-lane segments where vehicles occupied an integer offset along a segment connecting two nodes. This worked for the initial Rush Hour-style prototype but doesn't match the game's evolved vision: a realistic highway with multi-lane roads, opposing traffic, tree-lined medians with crossing gaps, and obstacle-based puzzles.

The concept sketch shows two parallel roads (each with 2 lanes), gridlocked traffic in the ambulance's direction, and opposing traffic flowing normally. The single-lane model can't represent any of this — there's no concept of lanes, no way to have parallel opposing segments, no static obstacles, and no lateral movement between roads.

This is a fundamental pivot of the road model that touches every layer of the logic: models, graph queries, validation, pathfinding, and level loading.

## Decision

Extend the road graph model with three new concepts:

### 1. Multi-lane segments

`RoadSegment` gains a `lanes` field (default 1) and an optional `groupId` that links parallel opposing segments. `Vehicle` and `Ambulance` each gain a `lane` field (default 0).

All collision detection, range checks, and pathfinding become **lane-aware**:
- Vehicles in different lanes at the same offset do not collide
- The ambulance only blocks vehicles in the same lane
- `isRangeFree()` accepts an optional `lane` filter
- BFS pathfinding considers a segment passable if ANY lane is clear
- Lane changes are validated by `MoveValidator.validateLaneChange()`
- Crossing through a node preserves the vehicle's lane (clamped if the target segment has fewer lanes)

### 2. Static obstacles

New `Obstacle` model (`ObstacleType`: tree, barrier, container, building) with offset, length, and optional lane. Obstacles are defined in level JSON as part of segment data. They never move and are never auto-cleared.

- `lane: null` means the obstacle blocks ALL lanes at that offset
- `lane: N` means it only blocks lane N (other lanes can pass)
- Obstacles are checked in `isRangeFree()`, `validateMove()`, `validateReturnFromShoulder()`, and ambulance passability

### 3. Median crossings

New `MedianCrossing` model defines gaps in the median where vehicles can cross to the opposing segment. Each crossing has an offset on the source segment, a target segment ID, a target offset, and a width.

- Only cars can cross (trucks are too wide)
- `validateMedianCrossing()` finds the first free lane on the target segment
- `moveVehicleThroughMedian()` performs the immutable graph update
- The ambulance NEVER crosses medians

### Backward compatibility

All new fields have defaults matching prior behavior: `lanes: 1`, `lane: 0`, `groupId: null`, `obstacles: null`, `medianCrossings: null`. Existing levels parse and play identically. All prior tests pass without modification.

## Alternatives Considered

- **Grid-based lane model**: Represent each lane as a separate row in a 2D grid. Rejected because the game's core identity is graph-based topology (ADR-002), not a grid. A grid can't represent curved roads, T-junctions, or variable-capacity segments naturally.

- **Lane as a separate entity (LaneSegment)**: Each lane is its own segment in the graph, with inter-lane edges for lane changes. Rejected because it explodes the graph size (a 2-lane road becomes 2 segments + 2 inter-lane edges per position), makes BFS pathfinding much more expensive, and complicates level authoring.

- **Obstacles as special vehicles**: Reuse the Vehicle model with a `movable: false` flag. Rejected because obstacles have fundamentally different semantics — they don't have intended directions, don't auto-clear, don't have sprites in the vehicle system, and mixing them into the vehicle list complicates every query.

- **Median crossing as a special node**: Add nodes at crossing points and short segments connecting them. Rejected because it adds artificial graph complexity and the crossings are inherently a property of the road (gaps in the median), not intersection points.

## Consequences

### What becomes easier

- **Highway-style levels**: Two parallel 2-lane segments with `groupId` linking them, obstacles on the median, crossing gaps — all expressible in level JSON
- **Ambulance lane switching**: `AmbulanceAI.findBestLane()` enables the ambulance to dynamically pick the clearest lane
- **Rich puzzle design**: Lane changes, obstacle avoidance, and median crossings create a much deeper strategy space than single-lane push puzzles
- **Opposing traffic**: Parallel segments with opposite `fromNode`/`toNode` naturally model two-way roads

### What becomes harder

- **Rendering complexity**: The rendering layer (Phase 3) must now handle lane offsets, lane dividers, obstacle sprites, and median gap visuals
- **Input complexity**: Drag gestures must distinguish longitudinal movement (along segment) from lateral movement (lane change / median crossing)
- **Level design**: More parameters to tune per level (lanes, obstacles, crossings). Mitigated by sensible defaults and validation in `LevelLoader`
- **Test surface area**: Every logic class gains lane-aware and obstacle-aware code paths. Mitigated by comprehensive test coverage (427 tests, up from 310)

## Implementation

Phase 1 (multi-lane) and Phase 2 (obstacles + median crossings) shipped together as logic-only changes. Remaining phases tracked as GitHub issues:

| Phase | Issue | Description |
|-------|-------|-------------|
| 3 | [#85](https://github.com/AnunnakiCosmoCrew/clear-ahead/issues/85) | Multi-lane rendering |
| 4 | [#86](https://github.com/AnunnakiCosmoCrew/clear-ahead/issues/86) | Lane change + median crossing input |
| 5 | [#87](https://github.com/AnunnakiCosmoCrew/clear-ahead/issues/87) | Vehicle AI (cooperative yielding, traffic flow) |
| 6 | [#88](https://github.com/AnunnakiCosmoCrew/clear-ahead/issues/88) | New highway levels + migration |

### Files changed

**New models:** `obstacle.dart`, `median_crossing.dart`

**Modified models:** `road_segment.dart` (+lanes, groupId, obstacles, medianCrossings), `vehicle.dart` (+lane, moveToLane), `ambulance.dart` (+lane)

**Modified logic:** `road_graph.dart` (lane-aware queries, obstacle checks, median operations), `move_validator.dart` (lane-aware validation, validateLaneChange, validateMedianCrossing), `ambulance_ai.dart` (lane-aware passability, findBestLane), `level_loader.dart` (parse all new fields with validation)
