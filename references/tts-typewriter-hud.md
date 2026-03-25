# TTS + Typewriter Subtitle HUD

## Overview

Combining TextToSpeech audio with a synchronised typewriter subtitle creates an accessible, polished narration experience. The typewriter is driven by a frame-rate-independent `UpdateEvent` that advances a character index based on `getDeltaTime()`.

## Module Setup

`TextToSpeechModule` and `AudioComponent` must both be present:

```
SceneObject: StoryDirector
├── ScriptComponent: StoryDirector.ts    ← ttsModule and ttsAudio @inputs
└── Component.AudioComponent             ← assign to ttsAudio @input
```

In the scene Resources panel: **+ → TextToSpeechModule** to add the module asset.

## Implementation

```typescript
@component
export class SubtitleHUD extends BaseScriptComponent {
  // ── Inspector inputs ──────────────────────────────────────────────────────

  @input
  @hint('Text component that shows the typewriter subtitle')
  subtitleText: Text

  @input
  @hint('AudioComponent on this SceneObject for TTS playback')
  ttsAudio: AudioComponent

  @input
  @hint('TextToSpeechModule asset from Resources panel')
  @allowUndefined
  ttsModule: TextToSpeechModule

  @input
  @hint('Font size for subtitle display')
  textSize: number = 30

  @input
  @hint('Characters revealed per second — calibrate to your TTS voice')
  charsPerSecond: number = 18

  // ── State ─────────────────────────────────────────────────────────────────

  private fullText:   string  = ''
  private charIndex:  number  = 0
  private elapsed:    number  = 0
  private isTyping:   boolean = false
  private isBusyFlag: boolean = false

  // ── Lifecycle ─────────────────────────────────────────────────────────────

  onAwake(): void {
    this.createEvent('OnStartEvent').bind(() => {
      if (this.subtitleText) this.subtitleText.size = this.textSize
    })
    this.createEvent('UpdateEvent').bind(() => this.onUpdate())
  }

  // ── Public API ────────────────────────────────────────────────────────────

  showNarration(text: string): void {
    this.fullText  = text
    this.charIndex = 0
    this.elapsed   = 0
    this.isTyping  = true
    this.isBusyFlag = true
    if (this.subtitleText) this.subtitleText.text = ''

    this.speakTTS(text)
  }

  hide(): void {
    this.isTyping   = false
    this.isBusyFlag = false
    if (this.subtitleText) this.subtitleText.text = ''
    if (this.ttsAudio?.isPlaying()) this.ttsAudio.stop(false)
  }

  /** Returns true while typewriter is animating OR TTS is still playing. */
  isBusy(): boolean {
    const subtitleBusy = this.isTyping
    const ttsBusy      = this.ttsAudio ? this.ttsAudio.isPlaying() : false
    return subtitleBusy || ttsBusy
  }

  // ── Private ───────────────────────────────────────────────────────────────

  private speakTTS(text: string): void {
    if (!this.ttsModule || !this.ttsAudio) return
    if (this.ttsAudio.isPlaying()) this.ttsAudio.stop(false)

    const options = TextToSpeech.Options.create()
    options.voiceName = 'Sasha'      // child-friendly Snap built-in voice

    this.ttsModule.synthesize(
      text,
      options,
      (audioTrack) => {
        this.ttsAudio.audioTrack = audioTrack
        this.ttsAudio.play(1)
        print('[HUD] TTS playback started')
      },
      (error, description) => {
        print('[HUD] TTS error: ' + error + ' — ' + description)
      }
    )
  }

  private onUpdate(): void {
    if (!this.isTyping || !this.subtitleText) return

    this.elapsed += getDeltaTime()
    const targetIndex = Math.floor(this.elapsed * this.charsPerSecond)

    if (targetIndex > this.charIndex) {
      this.charIndex = Math.min(targetIndex, this.fullText.length)
      this.subtitleText.text = this.fullText.slice(0, this.charIndex)
    }

    if (this.charIndex >= this.fullText.length) {
      this.isTyping = false
      // isBusyFlag stays true until TTS audio also finishes — checked via isBusy()
    }
  }
}
```

## Calibrating `charsPerSecond`

The goal is for the last character to reveal at the same moment the TTS audio ends.

**Method:**
1. Pick a known test string (e.g., 120 characters).
2. Call `ttsModule.synthesize()` and measure the audio duration (use `audioTrack.duration` if exposed, or time it manually).
3. `charsPerSecond = characterCount / audioDurationSeconds`

Typical values for a natural speaking pace:
- English, adult voice: ~14–18 chars/sec
- Children's voice (slower, clearer): ~10–14 chars/sec
- `"Sasha"` Snap voice: ~16 chars/sec

> **Tip:** Expose `charsPerSecond` as an `@input` (it already is in the snippet above) so you can tune it in the Inspector without recompiling.

## Available TTS Voices

Snap provides built-in TTS voices. The set available depends on Lens Studio version. Common options:

| Voice name | Character |
|---|---|
| `"Sasha"` | Friendly, slightly child-adjacent — good for storytelling |
| `"en-US-Standard-C"` | Standard US English female |
| `"en-US-Standard-D"` | Standard US English male |

Check the TTS module documentation for the current voice list in your Lens Studio version. Pass the name string directly to `options.voiceName`.

## `isBusy()` for Auto-Advance Timing

When advancing through a story automatically, you want to wait until **both** the typewriter finishes **and** the TTS audio finishes before moving to the next beat. The `isBusy()` method covers both:

```typescript
// In StoryDirector onUpdate:
if (this.readyToAdvance && !this.hud.isBusy()) {
  this.autoAdvanceTimer += getDeltaTime()
  if (this.autoAdvanceTimer >= this.autoAdvanceDelay) {
    this.autoAdvanceTimer = 0
    this.readyToAdvance   = false
    this.triggerAdvance()
  }
}
```

This prevents the next beat from starting while the user is still hearing the current narration.

## Text Size and Layout

The `Text` component's `size` property controls font size in Lens Studio's internal units. A size of `30` works well for floating subtitles in a Spectacles scene. Larger sizes are more readable from a distance; smaller sizes fit more text.

```typescript
// Set once on start
this.subtitleText.size = this.textSize    // @input, default 30

// For a HUD subtitle that wraps text properly:
// Set stretchMode to StretchMode.Fill and horizontalOverflow/verticalOverflow to 0 in Inspector
```

## Scene Setup for the HUD Object

Position the subtitle so it appears comfortably in the lower-centre of the user's view:

```
Camera
└── SubtitleHUD SceneObject
    ├── Background Image (optional panel/blur behind text)
    └── SubtitleText (Text component)
```

Recommended local transform (relative to Camera):
- Position: `(0, -15, -50)` — lower-centre of view, at reading distance
- Scale: `(1, 1, 1)` — Text component `size` controls apparent size

Since the SubtitleHUD is a Camera child (intentionally head-locked), it always stays in the user's view.
