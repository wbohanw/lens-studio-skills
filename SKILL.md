---
name: lens-studio-spectacles-ar
description: Advanced patterns for building AR experiences on Snap Spectacles in Lens Studio — world-space anchoring vs camera-relative positioning, scene-graph enabled-state cascading, 3D object spawner from FBX pool, HTTP polling for external APIs (no WebSocket support), Botpress Chat channel name gotcha, TextToSpeech + typewriter subtitle HUD, ASR voice input and error codes, idle animation for static FBX meshes, Spectacles coordinate system, and full Lens Studio MCP tool reference (SetLensStudioProperty, GetLensStudioSceneGraph, scripting workflow, face filters, macOS UI automation). Use when building Spectacles apps with dynamic 3D content, AI backends, voice interaction, or when controlling Lens Studio via MCP from an AI agent. Companion to lens-studio-scripting.
---

# Lens Studio Spectacles AR — Advanced Patterns

This skill covers platform-specific patterns and bugs that arise when building production AR experiences on Snap Spectacles. It is a direct companion to `lens-studio-scripting` (component lifecycle, decorators, basic API) — read that first if you are new to Lens Studio TypeScript.

All patterns here are derived from building a real Spectacles app end-to-end: AI-driven 3D animal spawning, chatbot integration over HTTP, TTS narration with typewriter subtitles, and voice command input.

---

## Quick Reference

| Problem | Reference file | One-line fix |
|---|---|---|
| Object follows user's head everywhere | `world-space-anchoring.md` | Reparent to scene root, not Camera |
| Object visible in editor, invisible at runtime | `scene-graph-hierarchy.md` | Only register direct children in spawner |
| Animal re-enabled but still invisible | `scene-graph-hierarchy.md` | Parent-enabled cascade — never disable inner mesh nodes individually |
| Objects appear behind the user | `coordinate-system.md` | Use negative Z for forward |
| FBX model appears tiny or giant | `fbx-spawner-pattern.md` | Apply scale correction — FBX units vs Lens units mismatch |
| Animals don't move | `idle-animation.md` | `Math.sin(getTime())` bob with per-animal phase offset |
| Chatbot never replies | `http-polling.md` | ISO timestamp filter rejecting bot reply — add debug logging |
| Bot receives messages but never responds | `botpress-chat-integration.md` | ADK `channel: "*"` does not match Chat API — use `"chat.channel"` |
| ASR always Error 3 in development | `asr-voice-input.md` | ASR not supported in desktop simulator — test on device only |
| ASR Error 3 on physical device | `asr-voice-input.md` | Mic permission not granted in Spectacles device settings |
| TTS plays but subtitles blank | `tts-typewriter-hud.md` | `UpdateEvent` not wired or `isTyping` flag not set |
| Button callback never fires | `sik-buttons.md` | Bind in `OnStartEvent` for scene buttons; call `initialize()` first for code-created buttons |
| `onButtonPinched` vs `onTriggerUp` confusion | `sik-buttons.md` | `onButtonPinched` = legacy `PinchButton`; use `onTriggerUp` on SUIK `RoundButton` |
| Hand pinch fires every frame | `hand-finger-interaction.md` | Add `wasPinching` debounce flag |
| Object doesn't respond to hand hover/pinch | `hand-finger-interaction.md` | Add `Interactable` component to the target object |
| Panel always faces wrong direction | `cards-frames-panels.md` | Enable Billboard on SUIK `Frame` |
| Panels overlap incorrectly | `cards-frames-panels.md` | Set `renderOrder` on `Frame` — higher value = in front |
| Can't find package import path | `asset-library.md` | Expand package in Asset Browser — folder structure maps to import path |
| Need multiple instances of same animal | `local-asset-spawning.md` | Use `ObjectPrefab.instantiateAsync()` instead of visibility pool |
| Don't know a scene object's UUID | `lens-studio-mcp.md` | Call `GetLensStudioSceneGraph` — never guess UUIDs |
| Property set via MCP has no effect | `lens-studio-mcp.md` | Wrong `valueType` — check the type table |
| Script works in editor, won't link via MCP | `lens-studio-mcp.md` | Must compile first, then set `scriptAsset` reference on ScriptComponent |
| Face filter elements all misaligned | `lens-studio-mcp.md` | Head Anchor `localTransform` must be `{0,0,0}` |

---

## Available References

Reference files are in `./references/` relative to this skill. Read the relevant file when the problem matches:

- **world-space-anchoring.md** — Why `setWorldPosition` follows the camera when objects are Camera children; how to detect and fix; MCP reparenting
- **scene-graph-hierarchy.md** — Recursive `scanChildren` registration trap; parent-enabled cascade; the visibility-toggle spawner pattern
- **coordinate-system.md** — Spectacles axis orientation; safe spawn positions; units (approx. centimetres); world origin at session start
- **fbx-spawner-pattern.md** — Importing FBX files via disk copy; building the visibility pool; scale normalisation; scene hierarchy layout
- **idle-animation.md** — Procedural bob and sway for static meshes; random phase offsets; keeping animation in sync with world position updates
- **http-polling.md** — Full polling pattern for external chatbot APIs; POST + exponential-backoff GET; ISO timestamp filtering; debug logging recipe
- **botpress-chat-integration.md** — `chat.channel` vs `"*"` in ADK conversation handlers; diagnosing silent message drops; deploy workflow
- **asr-voice-input.md** — ASR module setup; full error code table; simulator limitations; mic permission steps; silence timeout tuning
- **tts-typewriter-hud.md** — `TextToSpeechModule` setup; `AudioComponent` wiring; `UpdateEvent` typewriter loop; calibrating characters-per-second
- **debugging-checklist.md** — Invisible object triage tree; common error matrix with root causes and section pointers
- **sik-buttons.md** — `RoundButton` / `BaseButton` (SUIK, current) vs `PinchButton` (SIK, deprecated); `onTriggerUp` vs `onButtonPinched`; styles enum; scene-authored vs code-created wiring; toggle buttons
- **hand-finger-interaction.md** — `TrackedHand` vs `HandInteractor`; `isPinching()`, `isInTargetingPose()`; fingertip positions; `ProximitySensor`; `Interactable` component for hover+pinch events; near-field vs far-field
- **cards-frames-panels.md** — SUIK `Frame` (current) vs `ContainerFrame` (deprecated); `ScrollWindow` vs `ScrollView`; `ScreenTransform` for 2D layout; `renderOrder` and `relativeZ` for z-ordering; multi-panel show/hide manager
- **asset-library.md** — Opening the Asset Library; essential packages (SIK, SUIK, Sync Kit); `.lspkg` import path conventions; built-in modules (InternetModule, TTS, ASR) via Resources panel; browsing and updating packages
- **local-asset-spawning.md** — `ObjectPrefab.instantiate()` vs `instantiateAsync()`; `@input` as the only reliable asset reference at runtime (no `findAssetByName`); multi-prefab registry; destroying instances; visibility pool vs prefab decision guide
- **lens-studio-mcp.md** — Full MCP tool reference for Lens Studio 5.15+; `SetLensStudioProperty` valueType table; property paths for transforms/materials/scripts; TypeScript-first scripting workflow (write → compile → attach → run); face filter landmark positions; macOS UI automation fallback tools; troubleshooting connection and auth

---

## Core Concepts

### World space vs camera space

The single most disorienting Spectacles bug. Any `SceneObject` parented to `Camera` has its transforms resolved in camera-local space — so `setWorldPosition(0, -30, -50)` means "50 units in front of the camera" not "50 units in front of where the user started." The object follows the headset.

Fix: reparent the object to the scene root. See `world-space-anchoring.md`.

### The visibility-toggle spawner

Lens Studio has no runtime `Instantiate` for FBX assets. The standard pattern is a pool of pre-placed, disabled scene objects — one per animal/prop type — toggled via `enabled`. The critical rule: only register and toggle **direct children** of the pool root. Recursing into deeper descendants and toggling their `enabled` flags causes invisible objects that cannot be restored. See `scene-graph-hierarchy.md` and `fbx-spawner-pattern.md`.

### HTTP polling (no WebSocket)

Spectacles has no WebSocket API. Connecting to a real-time AI backend requires polling: POST a message, then repeatedly GET the conversation history until a bot reply appears. Filter by `userId !== myUserId` and `createdAt > afterTime` (ISO string comparison works). See `http-polling.md`.

### Procedural animation for static FBX

FBX files without rigged animations look dead. A simple `Math.sin(getTime())` bob on Y and sway on X in `UpdateEvent`, with a random phase offset per object, adds enough life that objects read as "present" rather than placed. See `idle-animation.md`.

---

## Spectacles Coordinate Quick Reference

| Axis | Direction | Note |
|---|---|---|
| +X | Right | |
| −X | Left | |
| +Y | Up | |
| −Y | Down | |
| **−Z** | **Forward** | Content the user should see |
| +Z | Behind user | Avoid for visible content |

Safe default spawn: `new vec3(0, -30, -50)` — ground level, ~50 cm in front, confirmed visible.

World origin = head position when the lens launched. Units ≈ centimetres.
