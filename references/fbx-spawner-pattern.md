# FBX Import and the Visibility-Toggle Spawner Pattern

## Why There Is No Runtime Instantiate

Lens Studio does not expose a runtime `Instantiate` API for FBX assets. You cannot dynamically load a `.fbx` file at runtime the way Unity or Unreal can. The workaround is a **visibility pool**: all models are placed in the scene at design time and shown/hidden by toggling their `enabled` flag.

## Import Workflow

### Step 1 — Copy FBX files to disk

Drop your `.fbx` files directly into the project's `Assets/` folder on disk (not via the Lens Studio GUI — just Finder/Explorer):

```
MyProject/
├── Assets/
│   ├── Scripts/
│   ├── fox.fbx         ← paste here
│   ├── bear.fbx
│   └── rabbit.fbx
```

### Step 2 — Refresh Lens Studio

Open or focus Lens Studio. It detects new files in `Assets/` automatically and imports them. The models appear in the Assets panel.

### Step 3 — Build the pool hierarchy in the editor

1. Create an empty `SceneObject` at the scene root — name it `AnimalPool` (or similar).
2. Drag each imported FBX model from the Assets panel into the scene as a **child** of `AnimalPool`.
3. Rename each child to match the string your spawner will look up (e.g., `"fox"`, `"bear"` — lowercase).
4. **Disable each child** in the Inspector (`enabled = false`).
5. Assign `AnimalPool` to your spawner script's `@input`.

### Resulting hierarchy

```
SceneRoot
├── Camera
├── AnimalPool              ← @input poolRoot; scene-root child (not Camera child!)
│   ├── fox                 ← disabled; named "fox" to match spawner key
│   ├── bear
│   ├── rabbit
│   ├── deer
│   ├── squirrel
│   ├── owl
│   └── ...
└── Effects
```

> **Must be scene-root child.** If `AnimalPool` is under `Camera`, all animals follow the headset. See `world-space-anchoring.md`.

## Scale Normalisation

FBX files exported in **metres** import into Lens Studio at 1/100 scale because Lens Studio's native unit is ~1 cm. A 1-metre fox FBX becomes a 1-unit fox — invisible at normal spawn distances.

Detect this if an animal's local scale is `(0.01, 0.01, 0.01)` in the Inspector, or if it appears as a tiny speck.

Fix by applying a scale multiplier at spawn time:

```typescript
@input animalScale: number = 20   // tune per FBX pack; 20 is typical for ~1 m models

spawn(name: string, worldPos: vec3): void {
  const obj = this.registry.get(name)
  if (!obj) return
  obj.enabled = true
  obj.getTransform().setWorldPosition(worldPos)
  obj.getTransform().setLocalScale(
    new vec3(this.animalScale, this.animalScale, this.animalScale)
  )
}
```

Expose `animalScale` as an `@input` so you can tune it per FBX pack in the Inspector without recompiling.

## Material Issues After FBX Import

FBX files import with basic materials. If a model renders as solid red or flat grey:

1. In the Assets panel, find the imported material (usually named `Mat_Standard` or similar).
2. Check if `baseTex` (albedo/diffuse texture) is assigned. If null, the model has no texture data in the FBX.
3. If the FBX pack includes separate texture files, assign them manually via the material's Inspector.
4. As a fallback, assign a custom Lens Studio PBR material to the `RenderMeshVisual` component on the mesh child.

## Naming Convention

The spawner looks up objects by `child.name.toLowerCase().trim()`. Keep names:
- All lowercase in the scene hierarchy (`"fox"`, not `"Fox"` or `"FOX"`)
- Matching the strings your AI backend sends (`"fox"`, not `"red_fox"`)
- Short and unambiguous — no spaces

```typescript
// Runtime lookup — must match scene object name exactly (lowercased)
spawn("fox")     // looks for child named "fox"
spawn("bear")    // looks for child named "bear"
```

If the AI sends `"a red fox"` instead of `"fox"`, normalise at the call site:

```typescript
const key = rawName.toLowerCase().replace(/^(a|an|the)\s+/, '').trim()
this.spawner.spawn(key, position)
```

## Multi-Instance Limitation

The visibility-pool pattern supports only **one instance** of each animal at a time (because there is only one `SceneObject` per type). If you need multiple instances of the same animal simultaneously, you need either:

1. **Duplicate entries:** Add `fox2`, `fox3` to the pool as separately placed models.
2. **ObjectPrefab instantiation:** Use `ObjectPrefab.instantiate()` for true multi-instance spawning — but this requires the model to be set up as a `Prefab` asset in Lens Studio, not a raw FBX.
