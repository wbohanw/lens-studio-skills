# Scene-Graph Hierarchy and `enabled` Cascading

Two linked bugs that together produce the same symptom: **"object spawns, logs say it is enabled, but nothing appears on screen."**

---

## Bug A — The Recursive `scanChildren` Registration Trap

### What happens

A spawner iterates the scene hierarchy recursively to build a `name → SceneObject` registry:

```typescript
// ❌ BAD — registers ALL descendants
private scanChildren(root: SceneObject): void {
  const count = root.getChildrenCount()
  for (let i = 0; i < count; i++) {
    const child = root.getChild(i)
    this.registry.set(child.name, child)
    this.scanChildren(child)              // ← recurse into inner nodes
  }
}
```

For a fox model with the hierarchy `Fox > Armature > Body_Mesh`, this registers three entries: `Fox`, `Armature`, and `Body_Mesh`.

Later, `hideAll()` iterates the full registry and sets `enabled = false` on every entry — including `Armature` and `Body_Mesh`. When `spawn("Fox")` re-enables only the `Fox` top-level node, the inner nodes remain explicitly disabled. The parent is visible but its mesh geometry is not.

### Fix

Only register and toggle **direct children** of the pool root. Never recurse:

```typescript
// ✅ GOOD — registers only direct children
private scanChildren(root: SceneObject): void {
  const count = root.getChildrenCount()
  for (let i = 0; i < count; i++) {
    const child = root.getChild(i)
    const key = child.name.toLowerCase().trim()
    if (key.length > 1) {
      this.registry.set(key, child)
      print('[Spawner] Registered: ' + key)
    }
    // Do NOT recurse — inner nodes are managed automatically by the parent
  }
}
```

---

## Bug B — Parent `enabled` Cascade

### What happens

Disabling a parent hides all descendants regardless of their own `enabled` state. **Re-enabling the parent does NOT automatically restore children that were individually disabled** — their `enabled` flag remains `false` even though the parent is now `true`.

This is why the two bugs compound: Bug A individually disables inner mesh nodes via `hideAll()`. Bug B means re-enabling the parent does not bring them back.

### The golden rule

> Toggle only the **parent's** `enabled` flag. Never set `enabled = false` on any node deeper than the top-level entry in your registry.

```typescript
hideAll(): void {
  // ✅ Toggle only registry entries (direct children)
  this.registry.forEach(obj => { obj.enabled = false })
  // Inner mesh nodes were never touched — they will come back when parent re-enables
}

spawn(name: string): void {
  this.hideAll()
  const target = this.registry.get(name.toLowerCase().trim())
  if (!target) { print('[Spawner] Unknown: ' + name); return }
  target.enabled = true   // ✅ Re-enable the top-level node only
  // Inner nodes: never disabled → still enabled → now visible via parent
}
```

---

## Complete Visibility-Toggle Spawner

```typescript
@component
export class ObjectSpawner extends BaseScriptComponent {
  @input
  @hint('Root SceneObject whose direct children are spawnable prefabs — all start disabled')
  poolRoot: SceneObject

  @input animalScale: number = 20

  private registry: Map<string, SceneObject> = new Map()
  private active: Set<string> = new Set()

  onAwake(): void {
    this.createEvent('OnStartEvent').bind(() => {
      this.registerDirectChildren()
    })
  }

  private registerDirectChildren(): void {
    const count = this.poolRoot.getChildrenCount()
    for (let i = 0; i < count; i++) {
      const child = this.poolRoot.getChild(i)
      const key = child.name.toLowerCase().trim()
      if (!key || key.length < 2) continue
      this.registry.set(key, child)
      child.enabled = false    // start hidden
    }
    print('[Spawner] Registered ' + this.registry.size + ' objects')
  }

  hideAll(): void {
    this.registry.forEach(obj => { obj.enabled = false })
    this.active.clear()
  }

  spawn(name: string, worldPos: vec3): boolean {
    const key = name.toLowerCase().trim()
    const obj = this.registry.get(key)
    if (!obj) {
      print('[Spawner] Not found: "' + key + '". Known: ' + [...this.registry.keys()].join(', '))
      return false
    }
    obj.enabled = true
    obj.getTransform().setWorldPosition(worldPos)
    obj.getTransform().setLocalScale(new vec3(this.animalScale, this.animalScale, this.animalScale))
    this.active.add(key)
    print('[Spawner] Spawned: ' + key + ' at ' + worldPos)
    return true
  }

  remove(name: string): void {
    const obj = this.registry.get(name.toLowerCase().trim())
    if (obj) obj.enabled = false
    this.active.delete(name.toLowerCase().trim())
  }

  clearAll(): void {
    this.hideAll()
  }

  getActiveObjects(): Set<string> {
    return new Set(this.active)
  }
}
```

---

## Debugging Invisible Objects

Add this diagnostic method to your spawner to see exactly what the registry contains at runtime:

```typescript
debugRegistry(): void {
  print('[Spawner] Registry dump:')
  this.registry.forEach((obj, key) => {
    print(`  "${key}" → enabled=${obj.enabled} worldPos=${JSON.stringify(obj.getTransform().getWorldPosition())}`)
  })
  print('[Spawner] Pool root enabled: ' + this.poolRoot.enabled)
}
```

Call this from a button press or delayed event to inspect state without stopping the lens.

---

## Scene Hierarchy Layout

```
SceneRoot
├── Camera
├── AnimalPool          ← poolRoot @input; must be scene-root child (see world-space-anchoring.md)
│   ├── fox             ← registered as "fox"; starts disabled
│   ├── bear            ← registered as "bear"; starts disabled
│   ├── rabbit
│   └── ...
└── Effects
```

> **Key constraint:** `AnimalPool` must be a child of the scene root (not Camera) or the animals will follow the headset. See `world-space-anchoring.md`.
