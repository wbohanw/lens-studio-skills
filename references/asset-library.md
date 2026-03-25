# Asset Library and Package Management

## Opening the Asset Library

Two ways to open it in Lens Studio:

1. Click the **Asset Library** button in the top-left toolbar
2. Menu: **Window → Asset Library**

From there, browse by category or search by name. Click **Install** on any package to add it to your project.

## Essential Packages for Spectacles Development

Install these for any serious Spectacles AR project:

| Package | Asset Library path | What it gives you |
|---|---|---|
| **SpectaclesInteractionKit (SIK)** | Spectacles → Spectacles Interaction Kit | Hand tracking, `TrackedHand`, `HandInteractor`, `Interactable`, raw gesture detection |
| **SpectaclesUIKit (SUIK)** | Spectacles → Spectacles UI Kit | `RoundButton`, `Frame`, `ScrollWindow`, `SnapOS2Styles` — modern Spectacles UI |
| **Spectacles Sync Kit** | Spectacles → Spectacles Sync Kit | Connected Lens / multiplayer sync — only if you need shared sessions |

> **TextToSpeechModule note:** Snap's current docs mark `TextToSpeechModule` as deprecated for Spectacles. The recommended path for new projects is `ASRModule` for voice input. If you need TTS narration, the module still works — just know it may not be supported long-term.

## Recommended Install Order for a New Project

```
1. SpectaclesInteractionKit   ← hand tracking; almost always needed
2. SpectaclesUIKit            ← if you have any buttons or panels
3. Add TTS / ASR modules via Resources panel (not Asset Library)
4. Add InternetModule via Resources panel for HTTP requests
```

## How Installed Packages Appear

After installing, the package appears under **Packages** in the Asset Browser. The package name becomes the `.lspkg` identifier used in import paths.

## Importing from a Package in TypeScript

The `.lspkg` suffix is how Lens Studio resolves package imports at build time:

```typescript
// Import a TypeScript class from an installed package
import { SIK } from 'SpectaclesInteractionKit.lspkg/SIK'
import { RoundButton } from 'SpectaclesUIKit.lspkg/Scripts/Components/Button/RoundButton'
import { Frame } from 'SpectaclesUIKit.lspkg/Scripts/Components/Frame/Frame'

// Require a JS module from a package
const asrModule = require('SpectaclesInteractionKit.lspkg/AsrModule')

// Require a built-in Lens Studio module (not from Asset Library)
const ttsModule = require('LensStudio:TtsModule')
const internetModule = require('LensStudio:RemoteServiceModule')
```

### The three require/import patterns

| Pattern | Use for |
|---|---|
| `import { X } from 'Package.lspkg/path'` | TypeScript classes from installed packages |
| `require('Package.lspkg/module.js')` | JavaScript modules from installed packages |
| `require('LensStudio:ModuleName')` | Built-in Lens Studio modules (TTS, ASR, HTTP, etc.) |
| `requireAsset('Package.lspkg/asset.png')` | Asset files inside packages |

## Finding What's Inside a Package

After installing, expand the package in the Asset Browser to explore its folder structure. For TypeScript, look inside the `Scripts/` subfolder — the folder structure maps directly to the import path.

Example for SUIK:
```
SpectaclesUIKit.lspkg/
└── Scripts/
    ├── Components/
    │   ├── Button/
    │   │   ├── BaseButton.ts      → import path: 'SpectaclesUIKit.lspkg/Scripts/Components/Button/BaseButton'
    │   │   └── RoundButton.ts
    │   └── Frame/
    │       └── Frame.ts           → 'SpectaclesUIKit.lspkg/Scripts/Components/Frame/Frame'
    └── Themes/
        └── SnapOS-2.0/
            └── SnapOS2.ts         → 'SpectaclesUIKit.lspkg/Scripts/Themes/SnapOS-2.0/SnapOS2'
```

## Adding Built-in Modules (Not From Asset Library)

Some modules are built into Lens Studio and added via the **Resources panel** (not the Asset Library):

1. In the Resources/Assets panel click **+**
2. Select the module type:
   - **InternetModule** — for HTTP requests (`RemoteServiceModule`)
   - **TextToSpeechModule** — for TTS (legacy on Spectacles)
   - **VoiceML Module** — for ASR voice recognition
3. The module asset appears in your project — drag it to a script's `@input` field in the Inspector

```typescript
@input internetModule: InternetModule       // drag RemoteServiceModule asset here
@input ttsModule: TextToSpeechModule        // drag TTS module asset here
```

## Browsing Useful Community / Official Assets

Beyond SIK and SUIK, the Asset Library includes:

| Category | What to look for |
|---|---|
| **3D Objects** | Pre-made environment props, characters (use as visibility-pool prefabs — see `fbx-spawner-pattern.md`) |
| **Materials** | PBR materials, unlit materials for UI surfaces |
| **Post Effects** | Bloom, color grade — useful for AR aesthetics |
| **Audio** | Ambient loops, SFX |
| **Fonts** | Additional typography for Text components |

Search tip: type a keyword (e.g., "animal", "particle", "glow") in the Asset Library search bar — results include both official Snap assets and community contributions.

## Keeping Packages Updated

When Snap releases a new version of SIK or SUIK, the Asset Library shows an **Update** badge on the installed package. Update carefully — SUIK breaking changes (like the `PinchButton` → `RoundButton` migration) can require refactoring existing button wiring.
