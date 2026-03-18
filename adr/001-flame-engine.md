# ADR-001: Use Flame engine for game rendering

**Status:** Accepted
**Date:** 2026-03-16
**Deciders:** Mert Ertugrul

## Context

ClearAhead! is a real-time traffic puzzle game that needs smooth rendering of road networks, vehicles, and animations. The game needs to run on iOS and Android from a single codebase.

## Decision

Use the Flame game engine on top of Flutter for all game rendering and input handling.

## Alternatives Considered

- **Pure Flutter (CustomPainter)**: Works for static/simple UIs but lacks game loop, component system, collision detection, and sprite management. Would require building a game engine from scratch.
- **Unity**: Industry standard for games but requires C#, separate codebase from any Flutter app shell, and large binary size. Overkill for a 2D puzzle game.
- **Godot**: Powerful 2D engine but GDScript/C# tooling, separate from Flutter ecosystem. Would need platform channels for any Flutter integration.
- **Raw Canvas/Skia**: Maximum control but no abstractions. Would spend months building what Flame provides out of the box.

## Consequences

- Flame provides game loop, component tree, input handling (DragCallbacks), and camera system
- Stays within the Flutter/Dart ecosystem — shared tooling, testing, and deployment
- Flame's component system maps naturally to game entities (roads, vehicles, ambulance)
- Smaller community than Unity/Godot but sufficient for a 2D puzzle game
- FlameGame subclass is the single entry point for the game
