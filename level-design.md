# Level Design Guide

## Current State (2026-03-20)

The game has pivoted from single-lane graph puzzles to multi-lane highway gameplay ([ADR-004](adr/004-multi-lane-road-model.md)). All 15 legacy single-lane levels have been deleted. Level 1 ("Highway Rush") is the sole level and the iteration target.

## Level JSON Schema

Levels are defined in `assets/levels/level_NNN.json`. The schema supports:

### Segments
```json
{
  "id": "seg_east",
  "from": "W",
  "to": "E",
  "capacity": 10,
  "lanes": 2,
  "groupId": "highway",
  "direction": "forward",
  "shoulders": [...],
  "obstacles": [...],
  "medianCrossings": [...]
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `lanes` | int | 1 | Number of lanes |
| `groupId` | string? | null | Links parallel opposing segments |
| `direction` | string? | null | `forward`, `backward`, `bidirectional`, or null (= bidirectional) |
| `obstacles` | array? | null | Static obstacles on the road |
| `medianCrossings` | array? | null | Gaps where vehicles can cross to opposing road |
| `shoulders` | array? | null | Pull-over zones with optional obstacles |

### Obstacles
```json
{ "id": "tree1", "type": "tree", "offset": 2, "length": 1, "lane": 0 }
```
Types: `tree`, `barrier`, `container`, `building`. If `lane` is omitted, blocks all lanes.

### Median Crossings
```json
{ "id": "mc1", "targetSegment": "seg_west", "offset": 4, "targetOffset": 5, "width": 2 }
```
Only cars can cross (trucks too wide).

### Vehicles
```json
{
  "id": "car_red",
  "type": "car",
  "sprite": "car_red",
  "segment": "seg_east",
  "offset": 2,
  "lane": 1,
  "control": "aiTraffic",
  "intendedDirection": { "segment": "seg_east", "towards": "E" },
  "route": { "segments": ["seg_east"], "exitNode": "E" }
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `lane` | int | 0 | Lane index (0-based) |
| `control` | string | `playerDraggable` | `playerDraggable`, `aiCooperative`, or `aiTraffic` |
| `route` | object? | null | For AI vehicles: ordered segment list + exit node |

### Ambulance
```json
{ "segment": "seg_east", "offset": 0, "exitNode": "E", "lane": 1 }
```

## Design Principles

1. **The ambulance's direction is completely jammed** — this IS the puzzle
2. **Opposing traffic flows normally** on the other road (aiTraffic)
3. **Cooperative vehicles yield** when the ambulance is nearby but are constrained by traffic
4. **Obstacles create fixed constraints** — trees in median, containers/buildings on shoulders
5. **Median crossings are strategic shortcuts** — only cars, and only at gaps
6. **Lane changes add lateral strategy** — move vehicles sideways to clear a path

## Iteration Plan

1. Refine level 1 until core gameplay loop feels right
2. Design tutorial levels (1-3) introducing mechanics one at a time
3. Design difficulty ramp with all mechanics combined
4. All levels built on multi-lane foundation
