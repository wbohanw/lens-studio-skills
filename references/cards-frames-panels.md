# Cards, Frames, and UI Panels

## Current vs Legacy

| Component | Package | Status | Notes |
|---|---|---|---|
| `Frame` | SpectaclesUIKit (SUIK) | ✅ Current | Use for all new panels |
| `ScrollWindow` | SpectaclesUIKit (SUIK) | ✅ Current | Scrollable content areas |
| `ContainerFrame` | SpectaclesInteractionKit (SIK) | ⚠️ Deprecated | Use `Frame` instead |
| `ScrollView` | SpectaclesInteractionKit (SIK) | ⚠️ Deprecated | Use `ScrollWindow` instead |

**Use SUIK `Frame` for all new floating panel / card work.**

## SUIK Frame

`Frame` is a floating panel component — a window-like container for UI content. It handles billboarding (always faces the user), close/follow controls, inner sizing, padding, and interaction.

### Inspector setup

1. Add Component → SpectaclesUIKit → Frame
2. Set **Inner Size** — the content area dimensions
3. Set **Padding** — spacing between edge and content
4. Enable **Billboard** to keep the panel always facing the user
5. The Frame automatically adds a `closeButton` and `followButton` (both `RoundButton` instances)

### TypeScript access

```typescript
import { Frame } from 'SpectaclesUIKit.lspkg/Scripts/Components/Frame/Frame'

@component
export class PanelController extends BaseScriptComponent {
  @input frameObject: SceneObject

  private frame: Frame

  onAwake(): void {
    this.createEvent('OnStartEvent').bind(() => {
      this.frame = this.frameObject.getComponent(
        Frame.getTypeName()
      ) as Frame

      // Wire the built-in close button
      this.frame.closeButton.onTriggerUp.add(() => {
        this.frameObject.enabled = false
      })

      // Wire the built-in follow button
      this.frame.followButton.onTriggerUp.add(() => {
        print('Follow toggled')
      })
    })
  }

  showPanel(): void {
    this.frameObject.enabled = true
  }

  hidePanel(): void {
    this.frameObject.enabled = false
  }
}
```

### Frame properties

| Property | Type | Description |
|---|---|---|
| `innerSize` | `vec2` | Content area size |
| `padding` | `number` | Padding around content |
| `closeButton` | `RoundButton` | Built-in close button |
| `followButton` | `RoundButton` | Built-in follow/pin button |
| `renderOrder` | `number` | Z-order for layering |
| `relativeZ` | `number` | Depth offset relative to siblings |

## ScreenTransform for 2D Layout

`ScreenTransform` positions UI elements in 2D screen space. It only works when:
- The SceneObject is under another `ScreenTransform` parent, **or**
- The SceneObject is directly under an orthographic camera

```typescript
// Get the ScreenTransform component
const st = this.sceneObject.getComponent('ScreenTransform') as ScreenTransform

// Position: anchors are -1 to 1 (left/bottom to right/top)
st.anchors.left   = -0.5
st.anchors.right  =  0.5
st.anchors.top    =  0.5
st.anchors.bottom = -0.5

// Offsets in screen units
st.offsets.left   = 0
st.offsets.right  = 0
st.offsets.top    = 0
st.offsets.bottom = 0

// Local position (centre of the ScreenTransform region)
st.position = new vec2(0, 0.2)   // slightly above centre
```

> **Important:** `ScreenTransform` operates in normalised 2D screen space (−1 to +1). For world-space floating panels on Spectacles, use a regular `Transform` with `Frame` instead — `ScreenTransform` is for flat 2D overlays only.

## Stacking Panels (Z-Order / Render Order)

Two mechanisms control which panel appears on top:

### 1. `renderOrder` (preferred)

Set `renderOrder` on the `Frame` or any component that exposes it. Higher values render on top:

```typescript
this.mainFrame.renderOrder    = 10
this.overlayFrame.renderOrder = 20   // renders on top of mainFrame
```

### 2. Scene hierarchy order

In the Lens Studio Objects panel, objects lower in the list render later (i.e., appear in front). For panels at the same `renderOrder`, the hierarchy order breaks ties.

### In practice

- Use `renderOrder` for intentional layering between panels.
- Use `relativeZ` on `Frame` for small depth offsets within a panel (e.g., button slightly in front of background).
- Avoid mixing both approaches — pick one per panel group.

## ScrollWindow (SUIK)

`ScrollWindow` replaces the deprecated `ScrollView`. It displays scrollable content under a `Canvas`.

```typescript
import { ScrollWindow } from 'SpectaclesUIKit.lspkg/Scripts/Components/ScrollWindow/ScrollWindow'

const scroll = this.scrollObject.getComponent(
  ScrollWindow.getTypeName()
) as ScrollWindow

// Scroll programmatically
scroll.scrollTo(0.5)    // 0 = top, 1 = bottom
```

Typical hierarchy for a scrollable panel:

```
Frame
└── Canvas
    └── ScrollWindow
        └── ContentContainer
            ├── Item 1
            ├── Item 2
            └── Item 3
```

## Building a Simple Floating Card

Minimal setup for a billboard card with a label and a close button:

```
SceneObject: InfoCard
├── Component: Frame         ← SUIK Frame; billboard=true
├── Component: Transform     ← position in world space
└── Children:
    ├── TitleText            ← Text component
    ├── BodyText             ← Text component
    └── ActionButton         ← RoundButton (style: Primary)
```

```typescript
@component
export class InfoCard extends BaseScriptComponent {
  @input frameObject:    SceneObject
  @input titleText:      Text
  @input bodyText:       Text
  @input actionButton:   SceneObject

  onAwake(): void {
    this.createEvent('OnStartEvent').bind(() => {
      const btn = this.actionButton.getComponent(
        RoundButton.getTypeName()
      ) as RoundButton

      btn.onTriggerUp.add(() => this.onAction())

      const frame = this.frameObject.getComponent(Frame.getTypeName()) as Frame
      frame.closeButton.onTriggerUp.add(() => {
        this.frameObject.enabled = false
      })
    })
  }

  showCard(title: string, body: string): void {
    this.titleText.text    = title
    this.bodyText.text     = body
    this.frameObject.enabled = true
  }

  private onAction(): void {
    print('[InfoCard] Action button pressed')
  }
}
```

## Multiple Panels — Show/Hide Pattern

Keep all panel root objects disabled at start. Show the active one, hide the rest:

```typescript
@component
export class UIManager extends BaseScriptComponent {
  @input modePanel:   SceneObject
  @input storyPanel:  SceneObject
  @input agentPanel:  SceneObject

  private allPanels: SceneObject[]

  onAwake(): void {
    this.allPanels = [this.modePanel, this.storyPanel, this.agentPanel]
    this.showPanel(this.modePanel)
  }

  showPanel(target: SceneObject): void {
    this.allPanels.forEach(p => { p.enabled = p === target })
  }

  showModePanel():  void { this.showPanel(this.modePanel)  }
  showStoryPanel(): void { this.showPanel(this.storyPanel) }
  showAgentPanel(): void { this.showPanel(this.agentPanel) }
}
```
