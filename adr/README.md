# Architecture Decision Records

This directory captures significant technical decisions for ClearAhead!

| ADR | Decision | Status | Date |
|-----|----------|--------|------|
| [001](001-flame-engine.md) | Use Flame engine for game rendering | Accepted | 2026-03-16 |
| [002](002-immutable-road-graph.md) | Immutable road graph with pure logic layer | Accepted | 2026-03-16 |
| [003](003-blender-art-pipeline.md) | Blender 3D render pipeline for art assets | Accepted | 2026-03-20 |
| [004](004-multi-lane-road-model.md) | Multi-lane road model with obstacles and median crossings | Accepted | 2026-03-20 |
| [005](005-gameobject-hierarchy.md) | GameObject hierarchy with universal collision | Proposed | 2026-03-23 |
| [006](006-vehicle-command-system.md) | Directive vehicle command system (tap+swipe) | Proposed | 2026-03-23 |
| [007](007-siren-radius-mechanic.md) | Siren radius as core game mechanic | Proposed | 2026-03-23 |
| [008](008-dynamic-vehicle-routing.md) | Dynamic vehicle routing with destination-based pathfinding | Proposed | 2026-03-23 |
| [009](009-zipper-merge-mechanic.md) | Zipper merge with lateral offset and ambulance straddling | Proposed | 2026-03-23 |
| [010](010-vehicle-animation-system.md) | Physics-based vehicle animation system | Accepted | 2026-03-27 |

## How to add a new ADR

1. Copy [template.md](template.md) to `NNN-short-title.md`
2. Fill in the fields
3. Open a PR
4. Update this index
