# Hand and Finger Interaction

## Two Layers to Understand

Lens Studio exposes hand interaction at two levels. Use the right one for your use case:

| Layer | Class | Use for |
|---|---|---|
| **Raw hand state** | `TrackedHand` (via `SIK.HandInputData`) | Gesture detection, fingertip positions, proximity |
| **Interaction layer** | `HandInteractor` | Hover detection, interactable targeting, event-driven activation |

For simple gestures (pinch to advance, fingertip position), use `TrackedHand`. For hover-over-object → pinch-to-activate patterns, use `HandInteractor` + `Interactable` components.

## Accessing TrackedHand

```typescript
import { SIK } from 'SpectaclesInteractionKit.lspkg/SIK'

@component
export class GestureHandler extends BaseScriptComponent {
  private rightHand: TrackedHand
  private leftHand:  TrackedHand

  onAwake(): void {
    this.rightHand = SIK.HandInputData.getHand('right')
    this.leftHand  = SIK.HandInputData.getHand('left')
    this.createEvent('UpdateEvent').bind(() => this.onUpdate())
  }

  private onUpdate(): void {
    if (this.rightHand.isPinching() || this.leftHand.isPinching()) {
      // Either hand is pinching
    }
  }
}
```

## Pinch Detection

```typescript
// Raw pinch state — fires every frame while pinching
if (rightHand.isPinching()) { ... }

// Targeting pose — hand held up with index finger extended (pointing)
if (rightHand.isInTargetingPose()) { ... }
```

> **Naming gotcha:** `TrackedHand` uses `isPinching()` and `isInTargetingPose()`. `HandInteractor` uses `isTargeting()`. These are different APIs at different layers — do not mix them up.

## Pinch Debounce Pattern

Checking `isPinching()` raw in `UpdateEvent` fires every frame the pinch is held. For a one-shot "pressed" event, debounce it:

```typescript
private wasPinching: boolean = false

private onUpdate(): void {
  const pinching = this.rightHand.isPinching() || this.leftHand.isPinching()

  if (pinching && !this.wasPinching) {
    this.wasPinching = true
    this.onPinchStart()          // fires once per pinch
  }
  if (!pinching && this.wasPinching) {
    this.wasPinching = false
    this.onPinchEnd()
  }
}
```

## Fingertip Positions

`TrackedHand` exposes named landmark points. Access their world-space positions:

```typescript
// Index fingertip — most common for pointing interactions
const tipPos: vec3 = rightHand.indexTip.position

// Other useful joints
const thumbTip:   vec3 = rightHand.thumbTip.position
const middleTip:  vec3 = rightHand.middleTip.position
const wrist:      vec3 = rightHand.wrist.position
const knuckle:    vec3 = rightHand.indexKnuckle.position
```

Use `indexTip.position` as a raycast origin for pointing interactions, or to check distance to spawned objects.

## Proximity Detection

`TrackedHand.getProximitySensor(landmarkName)` returns a `ProximitySensor` for a specific joint. The sensor detects nearby colliders:

```typescript
const proxSensor = rightHand.getProximitySensor('indexTip')
proxSensor.radius = 5.0    // detection radius in world units

this.createEvent('UpdateEvent').bind(() => {
  const overlapping = proxSensor.getOverlappingColliders()
  if (overlapping.length > 0) {
    print('Index tip near: ' + overlapping[0].sceneObject.name)
  }
})
```

> **Note:** `ProximitySensor` requires the target objects to have `PhysicsBodyComponent` / collider components. For simple distance checks without colliders, compute `vec3.distance(tipPos, objectPos)` manually.

## HandInteractor — Hover and Targeting

For hover-over-object interactions, use `HandInteractor` (attached to the camera or a hand anchor in your scene):

```typescript
import { HandInteractor } from 'SpectaclesInteractionKit.lspkg/Components/Interaction/HandInteractor/HandInteractor'

// Access via scene object reference
const interactor = this.handInteractorObject.getComponent(
  HandInteractor.getTypeName()
) as HandInteractor

// Is the hand currently pointing at / near any interactable?
if (interactor.isTargeting()) {
  print('Hovering over: ' + interactor.currentInteractable?.sceneObject.name)
}

// React to hover changes
interactor.onCurrentInteractableChanged.add((interactable) => {
  if (interactable) {
    print('Hover started: ' + interactable.sceneObject.name)
  } else {
    print('Hover ended')
  }
})
```

## Near-Field vs Far-Field Interaction

Spectacles supports two interaction distances:

| Mode | Distance | How activated |
|---|---|---|
| Near-field | < ~30 cm | Hand physically close to the object |
| Far-field (ray) | Any distance | Index-finger targeting pose + pinch |

`HandInteractor.nearFieldProximity` returns a float (0–1) indicating how close the hand is to the current interactable. Use this to drive hover visual feedback.

## Making a SceneObject Interactable

To receive hover and pinch events from `HandInteractor`, the target object needs an `Interactable` component:

1. Add Component → SpectaclesInteractionKit → Interactable
2. The object now fires `onTriggerStart`, `onTriggerEnd`, `onHoverStart`, `onHoverEnd` events

```typescript
import { Interactable } from 'SpectaclesInteractionKit.lspkg/Components/Interaction/Interactable/Interactable'

const interactable = this.targetObject.getComponent(
  Interactable.getTypeName()
) as Interactable

interactable.onTriggerStart.add((event) => {
  print('Pinched: ' + this.targetObject.name)
  print('Hit position: ' + JSON.stringify(event.intersectionHitPosition))
})

interactable.onHoverStart.add((event) => {
  // Show highlight effect
})

interactable.onHoverEnd.add((event) => {
  // Remove highlight effect
})
```

## Checking Hand Visibility

Before reading hand data, check that the hand is being tracked:

```typescript
if (rightHand.isTracked()) {
  const tipPos = rightHand.indexTip.position
  // safe to use
}
```

Use `isTracked()` to gracefully handle cases where the hand is out of the camera's field of view.
