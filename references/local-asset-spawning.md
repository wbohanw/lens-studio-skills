# Local Asset Spawning and ObjectPrefab

## Two Spawning Strategies

| Strategy | API | When to use |
|---|---|---|
| **Visibility pool** | `enabled = true/false` | Static FBX models with no variants; simple show/hide |
| **ObjectPrefab instantiate** | `prefab.instantiate()` | Multiple simultaneous instances; dynamic content; prefab assets |

For most Spectacles AR projects with a fixed set of animals/props, the **visibility pool** (see `fbx-spawner-pattern.md`) is simpler and more predictable. Use `ObjectPrefab` when you genuinely need multiple independent instances of the same asset simultaneously.

## Referencing Assets via @input (Preferred)

The standard and most reliable way to reference a local asset at runtime is an `@input` field assigned in the Inspector. The Lens Studio compiler resolves the asset at build time — no runtime name lookups needed.

```typescript
@component
export class Spawner extends BaseScriptComponent {
  @input
  @hint('Drag a Prefab asset from the Asset Browser here')
  foxPrefab: ObjectPrefab

  @input
  @hint('Multiple prefabs as a list')
  animalPrefabs: ObjectPrefab[]   // Inspector shows a resizable list
}
```

> **There is no `findAssetByName()` at runtime.** The `findAsset()` function exists only in the Editor Scripting API (for Lens Studio plugin development), not in runtime lens scripts. Always pass assets in via `@input` — do not try to look them up by name at runtime.

## Synchronous Instantiation

Simple and immediate. Use for small prefabs that are already loaded:

```typescript
const parent = scene.getRootObject(0)   // or any SceneObject

const instance: SceneObject = this.foxPrefab.instantiate(parent)
instance.name = 'Fox_Instance_1'

// Position it immediately
instance.getTransform().setWorldPosition(new vec3(0, -30, -50))
instance.getTransform().setLocalScale(new vec3(20, 20, 20))
```

`instantiate(parent)` returns the root `SceneObject` of the prefab. Pass `null` as parent to instantiate at the scene root.

## Asynchronous Instantiation

Use for larger prefabs to avoid frame hitches. The async version does not block the render loop:

```typescript
this.foxPrefab.instantiateAsync(
  parent,             // parent SceneObject (null = scene root)
  (instance) => {     // onSuccess
    instance.name = 'Fox_Async'
    instance.getTransform().setWorldPosition(new vec3(0, -30, -50))
    instance.getTransform().setLocalScale(new vec3(20, 20, 20))
    print('[Spawner] Fox instantiated async')
  },
  (error) => {        // onReject / onFailure
    print('[Spawner] Async instantiate failed: ' + error)
  },
  (progress) => {     // onProgress (0–1)
    print('[Spawner] Loading: ' + Math.round(progress * 100) + '%')
  }
)
```

> **Snap recommends `instantiateAsync` for production** especially on-device, where prefab loading can cause visible frame drops if done synchronously during gameplay.

## Destroying Instances

```typescript
// Destroy a specific instance
instance.destroy()

// Destroy after a delay
const ev = this.createEvent('DelayedCallbackEvent') as DelayedCallbackEvent
ev.bind(() => instance.destroy())
ev.reset(3.0)   // destroy after 3 seconds
```

> **Do not destroy objects during `UpdateEvent`** — defer with a `DelayedCallbackEvent` set to `0` delay if you need to destroy something mid-frame. Destroying synchronously during update can cause frame errors.

## Managing Multiple Instances

Keep a registry of live instances so you can clean them up:

```typescript
@component
export class PrefabSpawner extends BaseScriptComponent {
  @input foxPrefab: ObjectPrefab

  private instances: SceneObject[] = []

  spawnFox(pos: vec3): void {
    const inst = this.foxPrefab.instantiate(scene.getRootObject(0))
    inst.getTransform().setWorldPosition(pos)
    inst.getTransform().setLocalScale(new vec3(20, 20, 20))
    this.instances.push(inst)
  }

  clearAll(): void {
    this.instances.forEach(inst => {
      if (inst) inst.destroy()
    })
    this.instances = []
  }
}
```

## Multi-Prefab Registry Pattern

For a spawner that handles multiple animal types from an array of prefabs:

```typescript
@component
export class MultiSpawner extends BaseScriptComponent {
  @input animalNames:   string[]       // e.g. ["fox","bear","rabbit"]
  @input animalPrefabs: ObjectPrefab[] // must match names array order

  private prefabMap: Map<string, ObjectPrefab> = new Map()
  private instances: Map<string, SceneObject>  = new Map()

  onAwake(): void {
    this.createEvent('OnStartEvent').bind(() => {
      // Build name → prefab map from parallel arrays
      for (let i = 0; i < this.animalNames.length; i++) {
        this.prefabMap.set(
          this.animalNames[i].toLowerCase(),
          this.animalPrefabs[i]
        )
      }
      print('[MultiSpawner] Ready with ' + this.prefabMap.size + ' types')
    })
  }

  spawn(name: string, pos: vec3): void {
    const key    = name.toLowerCase().trim()
    const prefab = this.prefabMap.get(key)
    if (!prefab) {
      print('[MultiSpawner] Unknown: "' + key + '"')
      return
    }

    // Remove existing instance of this type first
    this.remove(key)

    prefab.instantiateAsync(
      scene.getRootObject(0),
      (inst) => {
        inst.getTransform().setWorldPosition(pos)
        inst.getTransform().setLocalScale(new vec3(20, 20, 20))
        this.instances.set(key, inst)
        print('[MultiSpawner] Spawned: ' + key)
      },
      (err) => print('[MultiSpawner] Failed: ' + err)
    )
  }

  remove(name: string): void {
    const inst = this.instances.get(name.toLowerCase())
    if (inst) { inst.destroy(); this.instances.delete(name.toLowerCase()) }
  }

  clearAll(): void {
    this.instances.forEach(inst => { if (inst) inst.destroy() })
    this.instances.clear()
  }
}
```

## Finding Scene Objects at Runtime

While there is no `findAssetByName` at runtime, you can find `SceneObject` nodes by name:

```typescript
// Search from the scene root (recursive)
const obj: SceneObject | null = scene.getRootObject(0).findChild('FoxModel', true)

// Or iterate manually
const count = parentObj.getChildrenCount()
for (let i = 0; i < count; i++) {
  const child = parentObj.getChild(i)
  if (child.name === 'FoxModel') {
    // found it
  }
}
```

`findChild(name, recursive)` — the second argument `true` enables recursive search through all descendants. Returns `null` if not found.

## Prefab vs Visibility Pool — Decision Guide

```
Do you need more than one instance of the same animal at the same time?
  YES → Use ObjectPrefab.instantiateAsync()
  NO  → Use visibility pool (enabled = true/false) — simpler and faster

Does the animal need to retain independent state (health, animation phase)?
  YES → Use ObjectPrefab instances (each instance is independent)
  NO  → Visibility pool is sufficient

Is the FBX file large (> a few MB)?
  YES → instantiateAsync to avoid frame hitches
  NO  → Either approach works; synchronous instantiate is simpler
```
