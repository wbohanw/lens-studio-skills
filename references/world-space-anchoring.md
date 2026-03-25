# World-Space Anchoring vs Camera-Relative Positioning

## The Bug

Any `SceneObject` that is a child (or descendant) of the `Camera` scene object has its transforms resolved in **camera-local space**. This means:

- `setWorldPosition(0, -30, -50)` does not place the object 50 units in front of the world origin — it places it 50 units in front of the camera, every frame.
- As the user moves their head, the object moves with it.
- The object appears "stuck" in space relative to the viewer, not the room.

This is not a bug in Lens Studio — it is how scene graph parenting is supposed to work. It becomes a bug when you *intend* the object to be anchored to the physical room.

## Symptom

> "The fox is always in front of me no matter where I walk."

Objects that should be fixed in physical space move when the user turns or walks. Logs show `setWorldPosition` is being called with constant coordinates — yet the object follows the headset.

## Diagnostic

Before anything else, ask: **Is this SceneObject (or any ancestor) a child of the Camera scene object?**

Check the scene hierarchy in Lens Studio. If you see:

```
Camera
└── AnimalPool        ← this is the problem
    ├── fox
    └── bear
```

everything under `AnimalPool` will follow the camera.

## Fix in Code

```typescript
// ❌ WRONG — AnimalPool is a Camera child
const pool = scene.createSceneObject('AnimalPool')
pool.setParent(this.cameraObject)         // Camera as parent
pool.getTransform().setWorldPosition(     // "world" = camera-local
  new vec3(0, -30, -50)
)

// ✅ CORRECT — AnimalPool is at scene root
const sceneRoot = scene.getRootObject(0)
const pool = scene.createSceneObject('AnimalPool')
pool.setParent(sceneRoot)                 // scene root as parent
pool.getTransform().setWorldPosition(     // genuinely world-space
  new vec3(0, -30, -50)
)
```

## Fix via Lens Studio MCP

If you have Lens Studio MCP access, reparent the object without touching any code:

```
SetLensStudioParent(objectUUID: "a1b2c3d4-...")   // no parentUUID = move to scene root
```

Passing no `parentUUID` argument moves the object to the scene root.

## When Camera-Parenting Is Intentional

Not all objects should be at the scene root. **Head-locked UI** — panels, HUDs, menus — should be children of `Camera` so they follow the user. The distinction is:

| Object type | Correct parent | Reason |
|---|---|---|
| Characters, props, environment | Scene root | Anchored to physical room |
| Heads-up HUD panels | Camera | Always in view |
| Floating subtitles | Camera | Always readable |
| World labels on real objects | Scene root | Attached to real-world position |

## `setWorldPosition` After `setParent`

Always call `setWorldPosition` **after** `setParent`. Setting position before reparenting places the object in the old coordinate space, then the parent relationship takes over and the position is re-interpreted. The result is an unexpected offset.

```typescript
// ✅ Correct order
const obj = scene.createSceneObject('Fox')
obj.setParent(scene.getRootObject(0))    // 1. set parent
obj.getTransform().setWorldPosition(     // 2. set position in new space
  new vec3(0, -30, -50)
)
```

## Keeping Fixed Position with Per-Frame Updates

If you are updating position every frame (e.g., for idle animation — see `idle-animation.md`), the object will stay at the correct world position as long as its parent is the scene root. The per-frame `setWorldPosition` call re-anchors it each frame, effectively overriding any drift.

```typescript
// This works correctly when obj is a scene-root child
private onUpdate(): void {
  obj.getTransform().setWorldPosition(
    new vec3(baseX + sway, baseY + bob, baseZ)  // world-space coords stay fixed
  )
}
```
