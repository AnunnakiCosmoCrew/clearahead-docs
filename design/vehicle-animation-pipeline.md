# Vehicle Animation Pipeline

**Status:** Implemented
**Date:** 2026-03-27
**Authors:** Mert Ertugrul, Claude
**Related ADR:** [ADR-010](../adr/010-vehicle-animation-system.md)
**Related Issues:** #146-#157 in clear-ahead

---

## 1. Overview

The vehicle animation pipeline layers physics-based visual effects on top of the BezierManeuver curve system. All effects are purely cosmetic — the game logic layer (RoadGraph, MoveValidator) operates on discrete integer positions and is unaware of animation.

```
Player swipe
    │
    ▼
CommandResolver (pure Dart)
    │ Returns ManeuverPlan(startSeg, endSeg, startOffset, endOffset, duration)
    │
    ▼
ClearAheadGame
    │ 1. Reserves target position atomically in RoadGraph
    │ 2. Captures old visual position/angle
    │ 3. Creates BezierManeuver effect
    │
    ▼
BezierManeuver (Flame Component)
    │ Each frame:
    │   1. Compute eased t (back-ease with anticipation/overshoot)
    │   2. Evaluate cubic Bezier position B(t)
    │   3. Add suspension bounce offset
    │   4. Record previous position for trail
    │   5. Update vehicle position
    │   6. Compute tangent for direction (guarded: only when t in [0,1])
    │   7. Update sprite direction from tangent
    │   8. Set body tilt from lateral tangent component
    │   9. Set motion stretch from actual speed + tangent direction
    │
    ▼
VehicleComponent.update()
    │   1. Smooth drag interpolation (lerp toward target during drag)
    │   2. Selection glow animation
    │   3. Jelly spring physics (damped spring toward scale targets)
    │
    ▼
VehicleComponent.render()
    │   1. Motion trail (ghost sprites at previous positions)
    │   2. Canvas transform (jelly scale * tilt)
    │   3. Drop shadow (blurred oval)
    │   4. Selection glow
    │   5. Sprite (pre-rendered isometric, 32 directions)
    │   6. Fade-out (auto-clear animation)
```

---

## 2. Easing Curve: Back-Ease-In-Out

The raw interpolation parameter `tLinear = elapsed / duration` is transformed through a back-ease-in-out function that provides anticipation and overshoot:

```dart
static double _easeInOut(double t) {
  const overshoot = 1.0;
  const scaled = overshoot * 1.525;
  if (t < 0.5) {
    final p = 2 * t;
    return (p * p * ((scaled + 1) * p - scaled)) / 2;
  } else {
    final p = 2 * t - 2;
    return (p * p * ((scaled + 1) * p + scaled) + 2) / 2;
  }
}
```

**Behavior:**
- `t = 0.0` → `eased = 0.0` (exact start)
- `t ≈ 0.05` → `eased ≈ -0.04` (anticipation: brief backward pull)
- `t = 0.5` → `eased = 0.5` (midpoint)
- `t ≈ 0.95` → `eased ≈ 1.04` (overshoot: past target)
- `t = 1.0` → `eased = 1.0` (exact end)

**Important:** Sprite direction is only updated when `eased t` is in `[0, 1]`. During anticipation and overshoot, the tangent points backward, which would flip the sprite to the wrong facing direction.

---

## 3. Effect Details

### 3.1 Suspension Bounce

A sinusoidal Y-offset simulating vehicle suspension. The amplitude is modulated by a `sin(pi * tLinear)` envelope so bounce starts and ends at zero.

```
bounce = sin(2π * frequency * elapsed) * amplitude * sin(π * tLinear)
```

Frequency scales with movement speed: `freq = base_8Hz + speed * 0.04`, clamped to 6-20Hz. Faster maneuvers produce higher-pitch bouncing.

### 3.2 Directional Squash/Stretch

The existing jelly spring system (`_scaleX`, `_scaleY` with damped spring physics) is driven by BezierManeuver based on **actual movement speed** (position delta / dt, not raw Bezier parameterization).

Stretch is applied along the dominant screen axis:
- Mostly horizontal movement → stretch X, squash Y
- Mostly vertical movement → stretch Y, squash X

This is a pragmatic compromise for pre-rendered isometric sprites. True tangent-aligned stretch would require a rotated scale transform, which would distort the pre-rendered perspective.

### 3.3 Body Tilt

During lateral movement (lane changes, shoulder moves), the vehicle compresses horizontally by up to 6% to simulate weight transfer lean.

```dart
lateralIntensity = tangent.dot(perpendicular).abs() / tangent.length
tiltScaleX = 1.0 - maxTilt * lateralIntensity
```

Applied as a separate `_tiltScaleX` multiplier in `render()`, independent of the jelly spring scale.

### 3.4 Motion Trail

A ring buffer of 3 recent world positions. Ghost sprites are rendered at these positions with decreasing opacity before the main sprite.

```
Ghost 0 (most recent): 12% opacity
Ghost 1: 7% opacity
Ghost 2 (oldest): 3% opacity
```

Trail paints are cached statically to avoid per-frame allocation. Previous position (before update) is recorded, not current position, so ghosts trail behind rather than overlaying the sprite.

### 3.5 Drop Shadow

A blurred oval drawn beneath the vehicle, offset slightly down-right (2px, 1.5px) to simulate isometric light from the upper-left.

```dart
shadowRect = Rect.fromCenter(
  center: Offset(size.x / 2 + 2.0, size.y * 0.85 + 1.5),
  width: size.x * 0.7,
  height: size.y * 0.3,
)
```

Shadow opacity tracks vehicle opacity during auto-clear fade-out. Uses a cached static paint; only temporarily swaps the color during fade-out.

### 3.6 Smooth Drag Interpolation

During drag, the vehicle's visual position lerps toward the logical target each frame using exponential decay:

```dart
lerpFactor = (1 - exp(-18.0 * dt)).clamp(0.0, 1.0)
position.lerp(dragTargetPos, lerpFactor)
```

Draw-order priority is updated from the interpolated position (not the target) to prevent depth-sorting artifacts during the lerp.

---

## 4. Cleanup and Early Cancellation

BezierManeuver overrides `onRemove()` to ensure all vehicle state resets even when the effect is cancelled early (e.g., player starts a new swipe while a maneuver is in progress):

```dart
@override
void onRemove() {
  vehicleComponent.clearTrail();
  vehicleComponent.setBodyTilt(0);
  vehicleComponent.resetMotionStretch();
  super.onRemove();
}
```

Without this, cancelled maneuvers would leave vehicles stuck in a tilted, stretched, or trailing state.

---

## 5. Performance Considerations

| Concern | Mitigation |
|---------|------------|
| Trail: 3 extra drawImage per moving vehicle per frame | Cached paints; only active during BezierManeuver (not stationary) |
| Shadow: blur filter per vehicle per frame | Cached static MaskFilter; single drawOval call |
| Spring physics: per-vehicle per-frame math | 6 multiplications + 2 exp() calls — negligible |
| Position recording for trail: clone per frame | Only during active animation; ring buffer of 3 |

If frame budget is tight on low-end devices, disable effects in this priority order:
1. Motion trail (most expensive: extra draw calls)
2. Suspension bounce (subtle, won't be missed)
3. Drop shadow (blur filter)
4. Squash/stretch and tilt (cheapest: just scale constants)

---

## 6. Future: Animated Sprite Sheets (#157)

The current system uses single static sprites per direction. Issue #157 proposes 2-4 frame sprite sheets with wheel rotation per direction. The animation pipeline is ready for this:

- `_loadSprites()` already loads by direction name — extend to load frame sequences
- `updateSpriteDirection()` already swaps sprites per frame — extend to cycle frames based on distance traveled
- Frame cycling should be distance-based (not time-based) so wheel spin matches ground speed
- Static when not moving (frame 0)
- Fallback to single sprite if frames not available

This is the single biggest remaining improvement and requires new Blender renders, not code architecture changes.
