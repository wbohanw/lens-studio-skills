# Spectacles Coordinate System

## World Origin

When a Lens launches on Spectacles, the **world origin (0, 0, 0) is placed at the user's head position at that moment**. All world-space coordinates are measured from that point. As the user physically moves through the room, their position shifts relative to this origin — but the origin itself does not move.

## Axis Orientation

| Axis | Direction | Mnemonic |
|---|---|---|
| +X | Right of user's initial facing | |
| −X | Left | |
| +Y | Up (sky) | |
| −Y | Down (floor) | |
| **−Z** | **Forward** (into the scene) | Content the user should see |
| +Z | Behind the user | Avoid for visible content |

> **Critical:** Z is inverted from what many 3D artists expect. Positive Z is *behind* the user. Always use **negative Z** for content that should appear in front.

## Units

Lens Studio world units are **approximately centimetres** on Spectacles. This means:
- `50` units ≈ 50 cm (arm's length in front)
- `100` units ≈ 1 metre
- `30` units below eye level ≈ roughly chest/ground level for a standing user

> **FBX import caveat:** FBX files exported in metres will import at 1/100 scale (because 1 FBX metre = 100 Lens units). See `fbx-spawner-pattern.md` for the scale correction.

## Safe Spawn Positions

These positions are relative to the world origin (initial head position). All are confirmed visible on-device.

| Intent | `vec3` | Notes |
|---|---|---|
| Ground level, 50 cm in front | `(0, -30, -50)` | Default character spawn — confirmed visible |
| Left of user at ground | `(-100, -30, -50)` | ~1 m to the left |
| Right of user at ground | `(100, -30, -50)` | ~1 m to the right |
| Overhead / floating | `(0, 30, -50)` | Above eye level |
| Eye level, arm's length | `(0, 0, -70)` | Good for floating objects or close interactions |
| Wide scene spread | `(-200, -30, -50)` / `(200, -30, -50)` | ~2 m spread for multiple characters |

```typescript
// Safe defaults
private readonly POS_CENTER: vec3 = new vec3(   0, -30, -50)
private readonly POS_LEFT:   vec3 = new vec3(-100, -30, -50)
private readonly POS_RIGHT:  vec3 = new vec3( 100, -30, -50)
private readonly POS_ABOVE:  vec3 = new vec3(   0,  30, -50)

spawnAt(name: string, position: string): void {
  const pos = {
    center: this.POS_CENTER,
    front:  this.POS_CENTER,
    left:   this.POS_LEFT,
    right:  this.POS_RIGHT,
    above:  this.POS_ABOVE,
  }[position.toLowerCase()] ?? this.POS_CENTER

  this.registry.get(name)?.getTransform().setWorldPosition(pos)
}
```

## Camera-Relative vs World-Space Spawning

Sometimes you want objects to spawn near where the user currently is (not relative to the launch origin). In that case, read the camera's current world position and offset from it:

```typescript
private spawnNearCamera(name: string): void {
  const cam = this.cameraObject.getTransform()
  const camPos = cam.getWorldPosition()

  // 50 cm in front of the user's current position, at eye-level
  const spawnPos = new vec3(camPos.x, camPos.y - 30, camPos.z - 50)

  this.registry.get(name)?.getTransform().setWorldPosition(spawnPos)
}
```

> **Warning:** If `cameraObject` is not assigned or its parent is misconfigured, `getWorldPosition()` may return (0, 0, 0). Always verify the camera reference in the Inspector.

## Fixed vs Camera-Relative: When to Use Each

| Mode | Use case | How to set |
|---|---|---|
| Fixed world position | Demo mode, story beats, anchored characters | Hardcode coordinates; see safe spawn table |
| Camera-relative | "Spawn in front of me" commands, agent mode | Read `cameraObject.getTransform().getWorldPosition()` + offset |

Switching between modes at runtime:
```typescript
public setFixedMode(fixed: boolean): void {
  this.useFixedPositions = fixed
}

private computeSpawnPosition(slot: string): vec3 {
  if (this.useFixedPositions || !this.cameraObject) {
    return FIXED_POSITIONS[slot] ?? this.POS_CENTER
  }
  const cam = this.cameraObject.getTransform().getWorldPosition()
  return new vec3(cam.x + OFFSETS[slot].x, cam.y - 30, -50)
}
```
