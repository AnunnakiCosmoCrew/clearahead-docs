# ADR-002: Immutable road graph with pure logic layer

**Status:** Accepted
**Date:** 2026-03-16
**Deciders:** Mert Ertugrul

## Context

The game needs a road network data structure that supports pathfinding (BFS), vehicle placement/movement validation, and collision detection. The logic must be testable without any rendering dependencies.

## Decision

Use an immutable `RoadGraph` data model in a pure Dart logic layer (`lib/logic/`). All state changes produce new graph instances. The rendering layer reads from the graph but never mutates it.

## Alternatives Considered

- **Mutable graph with in-place updates**: Simpler initially but harder to debug, can't easily undo moves, and mutation bugs are subtle in game loops.
- **ECS (Entity Component System)**: Standard in game engines but adds significant architectural complexity for a puzzle game with ~20 entities.
- **Graph library (e.g., `graphs` package)**: Adds a dependency for something simple enough to model with adjacency lists.

## Consequences

- Logic layer is 100% testable without Flame/Flutter dependencies
- `RoadGraph`, `MoveValidator`, `AmbulanceAI`, `HealthBar` are all pure Dart classes
- Graph immutability prevents accidental state corruption in the game loop
- Slight overhead from creating new graph instances on each move (negligible for puzzle-scale graphs)
- Clear separation: `lib/logic/` (pure) vs `lib/game/` (Flame rendering)
