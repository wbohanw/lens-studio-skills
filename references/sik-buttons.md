# SIK / SUIK Buttons

## Current vs Legacy

Snap has two button families. Know which one you are using:

| Component | Package | Status | Callback |
|---|---|---|---|
| `RoundButton`, `RectangleButton`, `CapsuleButton` | SpectaclesUIKit (SUIK) | ✅ Current | `onTriggerUp` |
| `BaseButton` | SpectaclesUIKit (SUIK) | ✅ Current base class | `onTriggerUp` |
| `PinchButton` | SpectaclesInteractionKit (SIK) | ⚠️ Deprecated | `onButtonPinched` |

**Use SUIK buttons for all new work.** `PinchButton` still functions but Snap's docs explicitly mark it deprecated in favor of SUIK. If you are maintaining older code that uses `onButtonPinched`, it will still work — but migrate when possible.

## Button Styles

Styles are defined by the `SnapOS2Styles` enum. The `style` property on `BaseButton` is typed as `string`, so pass the enum value as a string:

| Style | Appearance | Use for |
|---|---|---|
| `"Primary"` | Solid accent colour | Primary CTA |
| `"PrimaryNeutral"` | Solid neutral | Standard action |
| `"Secondary"` | Outlined / lower emphasis | Secondary actions |
| `"Special"` | Highlighted / prominent | Mic button, featured action |
| `"Ghost"` | Transparent background | Subtle, icon-only buttons |
| `"Custom"` | No preset — fully custom | Design overrides |

## Inspector Setup

1. Add a `RoundButton` component to a SceneObject via the Inspector (Add Component → SUIK → RoundButton).
2. Set `_style` to one of the values above.
3. Set `_width` to control button size (default `3`).
4. In the **Add Callbacks** Inspector section, enable it and wire `onTriggerUp` to a script method if you prefer Inspector-based wiring.

For programmatic wiring (preferred for maintainability), leave callbacks unwired in the Inspector and bind in code.

## TypeScript Wiring — Scene-Authored Button

When the button exists in the scene hierarchy (most common case), bind its callback in `OnStartEvent` — the button component is guaranteed initialised by then:

```typescript
import { RoundButton } from 'SpectaclesUIKit.lspkg/Scripts/Components/Button/RoundButton'
import { BaseButton  } from 'SpectaclesUIKit.lspkg/Scripts/Components/Button/BaseButton'

@component
export class MyPanel extends BaseScriptComponent {
  @input buttonObject: SceneObject    // drag the button's SceneObject here

  onAwake(): void {
    this.createEvent('OnStartEvent').bind(() => this.wireButtons())
  }

  private wireButtons(): void {
    const btn = this.buttonObject.getComponent(
      RoundButton.getTypeName()
    ) as RoundButton

    if (!btn) {
      print('[MyPanel] RoundButton not found on ' + this.buttonObject.name)
      return
    }

    btn.onTriggerUp.add(() => {
      print('[MyPanel] Button pressed: ' + this.buttonObject.name)
      this.onButtonPressed()
    })
  }

  private onButtonPressed(): void {
    // your logic here
  }
}
```

## TypeScript Wiring — Code-Created Button

When you instantiate a button at runtime (e.g., from a prefab), call `initialize()` before attaching callbacks, then wait for `onInitialized`:

```typescript
// After instantiating the button prefab:
const btn = instance.getComponent(RoundButton.getTypeName()) as RoundButton
btn.onInitialized.add(() => {
  btn.onTriggerUp.add(() => {
    print('Runtime button pressed')
  })
})
btn.initialize()
```

> **Rule of thumb:**
> - Scene-authored button → bind in `OnStartEvent`
> - Code-created button → call `initialize()` first, then bind in `onInitialized`

## Delayed Wiring Pattern (Alternative)

Some SUIK button internals initialise asynchronously. If `onTriggerUp` callbacks do not fire even after `OnStartEvent`, add a short `DelayedCallbackEvent`:

```typescript
const delay = this.createEvent('DelayedCallbackEvent') as DelayedCallbackEvent
delay.bind(() => {
  const btn = this.buttonObject.getComponent(RoundButton.getTypeName()) as RoundButton
  btn.onTriggerUp.add(() => this.onButtonPressed())
})
delay.reset(0.3)   // 300 ms — enough for SUIK internals to settle
```

This is the most reliable pattern when buttons are nested inside prefabs or panel hierarchies.

## Legacy PinchButton (Avoid for New Code)

```typescript
import { PinchButton } from 'SpectaclesInteractionKit.lspkg/Components/UI/PinchButton/PinchButton'

// ⚠️ Deprecated API — use RoundButton + onTriggerUp for new builds
const btn = obj.getComponent(PinchButton.getTypeName()) as PinchButton
btn.onButtonPinched.add(() => {
  print('Legacy pinch button pressed')
})
```

If you are reading older Spectacles project code and see `onButtonPinched`, it is from `PinchButton`. The callback works but the component is not recommended for new development.

## Toggle Buttons

`BaseButton` supports a toggleable mode via `_toggleable = true`. In toggle mode it exposes `onValueChanged` (fires with the new boolean state) and `onFinished`:

```typescript
btn.onValueChanged.add((isOn: boolean) => {
  print('Toggle is now: ' + isOn)
  this.micActive = isOn
  this.updateMicIcon(isOn)
})
```

Set `_defaultToOn` to control the initial state.

## Disabling a Button

```typescript
// Prevent interaction without hiding
btn.enabled = false        // component disabled — ignores input
btn.sceneObject.enabled = false   // hides the whole button
```

For a greyed-out appearance, set `enabled = false` on the component; SUIK applies a visual disabled state automatically.
