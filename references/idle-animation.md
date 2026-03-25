# Idle Animation for Static FBX Meshes

## Why Procedural Animation

FBX packs downloaded from asset stores often have no skeletal rig or baked animation tracks. Static meshes in AR look lifeless — users may not even notice the object is there if nothing moves. A simple procedural bob and sway adds enough motion that objects read as "alive" at a glance.

## The Pattern

Drive position offsets using `Math.sin(getTime())` in `UpdateEvent`. Apply separate frequencies and amplitudes to Y (bob) and X (sway) to avoid a mechanical straight-line oscillation.

```typescript
@component
export class ObjectSpawner extends BaseScriptComponent {
  @input bobSpeed:     number = 1.5   // cycles per second — vertical
  @input bobAmplitude: number = 1.2   // world units
  @input swaySpeed:    number = 0.8   // cycles per second — horizontal
  @input swayAmplitude:number = 0.6   // world units

  // Keyed by scene object's uniqueIdentifier
  private basePositions: Map<string, vec3>  = new Map()
  private phaseOffsets:  Map<string, number> = new Map()
  private activeObjects: Set<string>         = new Set()

  onAwake(): void {
    this.createEvent('UpdateEvent').bind(() => this.onUpdate())
  }

  // Call this after enabling an object and setting its world position
  registerForAnimation(obj: SceneObject): void {
    const id = obj.uniqueIdentifier
    this.basePositions.set(id, obj.getTransform().getWorldPosition())
    this.phaseOffsets.set(id,  Math.random() * Math.PI * 2)   // random phase
    this.activeObjects.add(id)
  }

  unregisterFromAnimation(obj: SceneObject): void {
    const id = obj.uniqueIdentifier
    this.basePositions.delete(id)
    this.phaseOffsets.delete(id)
    this.activeObjects.delete(id)
  }

  private onUpdate(): void {
    if (this.activeObjects.size === 0) return
    const t = getTime()

    this.activeObjects.forEach(id => {
      const obj   = this.findObjectById(id)
      const base  = this.basePositions.get(id)
      const phase = this.phaseOffsets.get(id) ?? 0
      if (!obj || !base || !obj.enabled) return

      const bob  = Math.sin(t * this.bobSpeed  * Math.PI * 2 + phase) * this.bobAmplitude
      const sway = Math.sin(t * this.swaySpeed * Math.PI * 2 + phase + 1.0) * this.swayAmplitude
      //                                                                ^^^
      // +1.0 shifts sway phase relative to bob → elliptical motion, not straight line

      obj.getTransform().setWorldPosition(
        new vec3(base.x + sway, base.y + bob, base.z)
      )
    })
  }
}
```

## The Phase Offset — Why It Matters

Without a per-object phase offset, all animals bob and sway in **perfect synchrony**. Ten foxes all moving up at the same time looks more mechanical and eerie than no animation at all.

A random phase in `[0, 2π]` staggers each object's cycle independently:

```typescript
// Same time t, different phases = different positions
const fox_y  = Math.sin(t * speed + 0.0)   // peaks at t = 0
const bear_y = Math.sin(t * speed + 2.1)   // peaks at a different time
const owl_y  = Math.sin(t * speed + 4.7)   // peaks at yet another time
```

Assign phase once at registration — do not randomise each frame.

## Base Position Update

When an animal's world position changes (e.g., after `setWorldPosition` in `spawn()`), update its base position:

```typescript
spawn(name: string, worldPos: vec3): void {
  const obj = this.registry.get(name)
  if (!obj) return
  obj.enabled = true
  obj.getTransform().setWorldPosition(worldPos)
  obj.getTransform().setLocalScale(...)

  // Update base — the animation loop references this
  if (this.basePositions.has(obj.uniqueIdentifier)) {
    this.basePositions.set(obj.uniqueIdentifier, worldPos)
  } else {
    this.registerForAnimation(obj)
  }
}
```

## Tuning Parameters

| Parameter | Low value effect | High value effect | Recommended start |
|---|---|---|---|
| `bobSpeed` | Slow, dreamy float | Fast nervous jitter | 1.0 – 1.5 |
| `bobAmplitude` | Barely moves | Exaggerated bounce | 1.0 – 2.0 |
| `swaySpeed` | Slow side lean | Fast side-to-side | 0.6 – 1.0 |
| `swayAmplitude` | Barely sways | Large side oscillation | 0.5 – 1.0 |

Set all four as `@input` fields so you can tune them in the Inspector without recompiling.

## Disabling Animation for Specific Objects

If some objects in the pool should not animate (e.g., a static tree vs an animal), simply do not call `registerForAnimation()` for them. The animation loop only processes objects in `activeObjects`.

```typescript
spawn(name: string, worldPos: vec3): void {
  ...
  obj.enabled = true
  if (this.ANIMATED_TYPES.has(name)) {
    this.registerForAnimation(obj)
  }
}

private readonly ANIMATED_TYPES = new Set(['fox', 'bear', 'rabbit', 'deer', 'squirrel', 'owl'])
```

## Performance Note

`setWorldPosition` is called every frame for every active animated object. For typical Spectacles scenes with fewer than 20 simultaneous objects, this is negligible. If you have many more, consider batching into a single update or using a lower-frequency delayed event for non-critical animation.
