# ASR Voice Input

## Module Access

The Automatic Speech Recognition module is accessed via `VoiceML.InferenceSessionModule` in Lens Studio. On Spectacles it uses the on-device microphone and cloud speech recognition.

```typescript
private asr: AsrModule = require('LensStudio:AsrModule')
```

## Error Code Reference

| Code | Enum name | Meaning | Most likely cause | Fix |
|---|---|---|---|---|
| 0 | (success) | Transcription completed normally | — | — |
| 1 | `GENERIC` | Unknown error | Various | Check device logs |
| **2** | `NETWORK` | Network unavailable | No internet / proxy | Verify device Wi-Fi or cellular |
| **3** | `NO_SPEECH` | No speech detected (silence timeout) | Mic permission not granted, OR user did not speak within `silenceUntilTerminationMs` | Grant permission; increase timeout; remind user to speak |
| 4 | `NOT_AVAILABLE` | ASR unavailable in region | Region restriction | — |

**Error 3 is the most common** in development and production. The two causes require different fixes — check permission status before assuming it is a timeout issue.

## Basic Setup

```typescript
@component
export class VoiceListener extends BaseScriptComponent {
  @input silenceMs: number = 2500   // ms of silence before session ends

  private asr: AsrModule = require('LensStudio:AsrModule')
  private options: AsrModule.AsrTranscriptionOptions | null = null
  private isListening: boolean = false

  onAwake(): void {
    this.options = AsrModule.AsrTranscriptionOptions.create()
    this.options.silenceUntilTerminationMs = this.silenceMs
    this.options.mode = AsrModule.AsrMode.HighAccuracy

    this.options.onTranscriptionUpdateEvent.add((args) => {
      if (args.text.trim() === '') return
      print('[ASR] partial: ' + args.text + ' final=' + args.isFinal)
      if (args.isFinal) {
        this.stopListening()
        this.onTranscription(args.text.trim())
      }
    })

    this.options.onTranscriptionErrorEvent.add((code) => {
      print('[ASR] error: ' + code)
      this.isListening = false
      this.onError(code)
    })
  }

  startListening(): void {
    if (this.isListening || !this.options) return
    this.isListening = true
    print('[ASR] Started')
    this.asr.startTranscribing(this.options)
  }

  stopListening(): void {
    if (!this.isListening) return
    this.isListening = false
    this.asr.stopTranscribing()
    print('[ASR] Stopped')
  }

  // Override these in subclass or assign callbacks
  onTranscription(text: string): void {
    print('[ASR] Final: ' + text)
  }

  onError(code: number): void {
    switch (code) {
      case 2:
        print('[ASR] Network error — is the device online?')
        break
      case 3:
        print('[ASR] No speech — permission denied or silence timeout')
        break
    }
  }
}
```

## Silence Timeout Tuning

`silenceUntilTerminationMs` controls how long the session waits in silence before firing Error 3. Too short and the session ends before the user starts speaking. Too long and the user has to wait after finishing.

| Scenario | Recommended `silenceMs` |
|---|---|
| Quick commands ("spawn a fox") | 1500 – 2000 ms |
| Natural speech, longer sentences | 2500 – 3000 ms |
| Children / slower speakers | 3000 – 4000 ms |

```typescript
@input silenceMs: number = 2500   // expose in Inspector for easy tuning
```

## Simulator Limitation

**ASR always fails in the Lens Studio desktop simulator.** This is a platform constraint — the simulator cannot access the microphone in the same way the device can, and cloud ASR requires the Spectacles hardware authentication.

Expected simulator behavior: Error 3 fires immediately or after the silence timeout with no transcript.

> **Always test ASR features on a physical Spectacles device.** Do not debug ASR errors in the simulator.

## Mic Permission

On Spectacles, microphone access requires a lens permission. If the permission is not granted, ASR fires Error 3 immediately.

**To grant permission:**
1. Put on the Spectacles and open Settings.
2. Navigate to Permissions → Lens permissions.
3. Find your lens (or the "Unknown lens" entry during development).
4. Enable **Microphone**.

Or via the Snapchat app: the first time an ASR-enabled lens runs, Spectacles shows a system prompt asking for mic permission. If the user denied it, they must go to Settings to re-grant.

## Mode Selection

`AsrModule.AsrMode` controls accuracy vs latency:

| Mode | Latency | Accuracy | Use for |
|---|---|---|---|
| `FastResponse` | Low | Lower | Real-time partial display |
| `HighAccuracy` | Higher | Better | Final command interpretation |

For an AR chatbot, use `HighAccuracy` — the extra latency (usually <1 second) is worth the improved recognition, especially for children's names and invented words.

## Handling Partial Transcriptions

Partial transcriptions (`isFinal = false`) arrive as the user speaks. Use them to show live feedback in the UI without sending to the AI:

```typescript
this.options.onTranscriptionUpdateEvent.add((args) => {
  if (!args.text.trim()) return

  // Show partial in status text — gives user feedback they are being heard
  if (this.statusText) this.statusText.text = args.text

  // Only send to AI when final
  if (args.isFinal) {
    this.stopListening()
    this.sendToBot(args.text.trim())
  }
})
```
