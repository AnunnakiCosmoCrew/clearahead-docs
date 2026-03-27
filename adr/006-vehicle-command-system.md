# ADR-006: Directive vehicle command system (tap+swipe replacing drag)

**Status:** Proposed
**Date:** 2026-03-23
**Deciders:** Mert Ertugrul

## Context

The current input system uses **drag callbacks** on `ClearAheadGame`: the player touches a vehicle and drags it along the segment. The vehicle teleports instantly to the new position. This was inherited from the Rush Hour sliding-block puzzle model.

Testing feedback (clearahead-docs#1) identified this as the biggest problem with the game feel:

> "I think this car dragging logic is off, with this logic it feels like user moving cars like toy blocks. I think user would like to see a realistic movement, like when a car makes a place for the ambulance by moving to the right shoulder of the road it first turns the wheel to right then its right front part then the whole front then the whole car enters the shoulder."

> "I think the new logic would be: user tells the direction for the car to move, then the car itself moves like a real car to the side of the road."

The player's role is a helicopter dispatcher — they should be **commanding** vehicles, not physically pushing them. The input system should match this fantasy.

## Decision

Replace drag-and-snap with a **two-step tap+swipe command system**:

### Step 1: Tap to Select
- Player taps a vehicle that is in `sirenAware` or `waiting` state
- Vehicle highlights (selection glow)
- Vehicles in other states (autonomous, executing, returning) don't respond to taps

### Step 2: Swipe to Command
- Player swipes in a direction from the selected vehicle
- Swipe vector is decomposed relative to the segment direction into **6 directions**: forward, backward, and 4 diagonals (forward-left, forward-right, backward-left, backward-right)
- **Pure perpendicular (left/right) is excluded** — real vehicles cannot move sideways; only front wheels steer, so any lateral movement requires forward or backward motion (a diagonal arc)
- Swipe magnitude maps to movement distance: short (1 unit), medium (2-3 units), long (max valid)
- The game resolves the command to the furthest valid target position in that direction (collision-free, within bounds)
- Vehicle transitions to `executing` state and performs a Bézier curve maneuver to the target

### Deselection
- Tap on empty space → deselect
- Tap same vehicle again → deselect
- Tap different valid vehicle → switch selection

### Direction Decomposition
The swipe vector is rotated by the negative of the segment angle so that "forward along segment" maps to the forward command regardless of screen orientation. This ensures intuitive controls on curved roads.

With 6 directions (instead of 8), each angular zone is 60° wide, providing generous hit targets on small phone screens. The zones are:
- Forward: -30° to +30° relative to segment direction
- Forward-right: +30° to +90°
- Backward-right: +90° to +150°
- Backward: +150° to -150° (wraps)
- Backward-left: -150° to -90°
- Forward-left: -90° to -30°

### Bézier Curve Execution
Once a command resolves to a target position, the vehicle follows a cubic Bézier curve:
- Control points create a natural arc (car turns into the maneuver)
- Vehicle sprite rotates to follow the curve tangent
- Duration: cars 0.55s/unit (tuned from 0.4s), trucks 0.7s/unit
- Intermediate collision detection prevents clipping through objects mid-maneuver

### Architecture Split
- **Logic layer** (`CommandResolver`): pure Dart, validates the target position, returns `ManeuverPlan(startSegment, startOffset, startLane, endSegment, endOffset, endLane)`
- **Game layer** (`BezierManeuver`): Flame-specific, converts ManeuverPlan to Bézier curve, animates the vehicle, handles rotation

The logic layer never touches pixel coordinates or curves.

## Alternatives Considered

- **Improved drag (smoother, with animation):** Keep drag but animate the vehicle along its path instead of instant teleport. Rejected because it still has the "pushing toy blocks" feel — the player is physically moving the vehicle, not commanding it. It also doesn't fit the dispatcher fantasy.

- **Tap destination (tap where vehicle should go):** Player taps a vehicle, then taps the target position. Rejected because it's hard to tap precise lane/shoulder positions on a small phone screen, and it doesn't convey directionality (the vehicle might take an unexpected path).

- **Gesture draw (draw a path):** Player draws the vehicle's path with their finger. Rejected as too complex for the quick, reactive gameplay needed during the active phase. Drawing paths while the ambulance is racing is too slow.

- **Tap-only with auto-resolve:** Tap a vehicle and it automatically moves to the best position. Rejected because it removes player agency entirely — the core puzzle is deciding WHERE each vehicle should go.

## Consequences

### What becomes easier

- **Dispatcher feel** — player commands, vehicles execute. Matches the helicopter dispatcher fantasy.
- **Realistic movement** — Bézier curves look natural; vehicles turn rather than slide
- **Realistic 6-direction commands** — all lateral movement is diagonal (forward+lateral arc), matching real vehicle physics. No unrealistic pure-sideways slides.
- **Difficulty scaling** — the command system works identically across difficulties; only the auto-yield fallback changes

### What becomes harder

- **Discoverability** — new players need to learn tap+swipe. Mitigated by tutorial levels that introduce the mechanic step-by-step.
- **Precision on small screens** — swipe direction must be clearly distinguished (6 directions). Mitigated by generous 60° angle zones per direction.
- **Testing** — gesture recognition needs careful testing with varied swipe angles and speeds. Mitigated by unit-testable CommandResolver (pure Dart, no gesture dependency).
- **A/B testing risk** — if tap+swipe doesn't feel good, we need to revert or iterate. Mitigated by implementing this as a self-contained phase; drag code can be feature-flagged during testing.
