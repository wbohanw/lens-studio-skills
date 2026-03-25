# Debugging Checklist and Error Matrix

## Invisible Object Triage

Work through this in order — each step rules out one root cause.

```
1. Is the object (or any ancestor) a child of the Camera SceneObject?
   YES → Reparent to scene root
         Reference: world-space-anchoring.md

2. Did hideAll() or any registry method recurse into descendant nodes?
   YES → Refactor to direct-children-only registration
         Reference: scene-graph-hierarchy.md § Bug A

3. Was any inner mesh node explicitly set to enabled=false at any point?
   YES → Parent re-enable does not restore it — check and reset inner node flags
         Reference: scene-graph-hierarchy.md § Bug B

4. Is setWorldPosition() called BEFORE setParent()?
   YES → Move setWorldPosition() to after setParent()
         Reference: world-space-anchoring.md § setWorldPosition After setParent

5. Is the Z coordinate positive?
   YES → Use negative Z for forward-facing content
         Reference: coordinate-system.md

6. Is the FBX model at a very small scale (e.g., 0.01)?
   YES → Apply scale correction in spawn()
         Reference: fbx-spawner-pattern.md § Scale Normalisation

7. Is the pool root itself disabled?
   Check: poolRoot.enabled in logs
   YES → Enable the pool root (it must be enabled for children to show)

8. Is the object spawning at world position (0,0,0)?
   Check: add print(obj.getTransform().getWorldPosition()) after spawn
   YES → setWorldPosition is not being called, or is being called before setParent
```

## Chatbot Never Replies

```
1. Does the POST /messages return a valid message ID?
   NO  → Network issue, wrong URL, or wrong headers — check HTTP status code
   YES → Continue

2. Does the poll return more than 1 message?
   NO (always 1 = only the user's own message)
   → Bot is not replying at all
   → Check ADK conversation handler channel name:
     channel: 'chat.channel'   NOT channel: '*'
     Reference: botpress-chat-integration.md

3. Does the poll return a bot message but the filter rejects it?
   (isBot=true, newer=false)
   → afterTime comparison failing
   → Use createdAt from POST response, not local new Date().toISOString()
   Reference: http-polling.md § ISO Timestamp Filtering

4. Is the bot message payload.type something other than 'text'?
   → ADK handler is sending a non-text message type
   → Check ADK execute() output and system prompt
```

## ASR Issues

```
1. Error 3 in desktop simulator
   → Expected — ASR is not supported in the simulator
   → Test on physical Spectacles only

2. Error 3 on physical device, immediately on start
   → Mic permission not granted
   → Settings → Lens permissions → enable Microphone for your lens

3. Error 3 after speaking
   → silenceUntilTerminationMs too short — user ran out of time
   → Increase to 3000ms or more for children / longer phrases
   Reference: asr-voice-input.md § Silence Timeout Tuning

4. Error 2
   → Device not connected to internet
   → Check Wi-Fi / cellular on Spectacles
```

## Common Error Matrix

| Symptom | Root cause | Reference |
|---|---|---|
| Object moves with user's head | Camera-parented SceneObject | world-space-anchoring.md |
| Invisible at runtime, visible in editor | Recursive hideAll() disabled inner mesh nodes | scene-graph-hierarchy.md |
| Animal re-enabled but stays invisible | Parent-child enabled cascade | scene-graph-hierarchy.md |
| Objects appear at feet or far away | Wrong scale or positive Z | coordinate-system.md, fbx-spawner-pattern.md |
| All animals bob in synchrony | No phase offset on idle animation | idle-animation.md |
| FBX appears as tiny speck | FBX unit mismatch — apply scale correction | fbx-spawner-pattern.md |
| Chatbot never replies (poll times out) | ADK `channel: '*'` — use `'chat.channel'` | botpress-chat-integration.md |
| Poll only shows 1 message (user's own) | Bot not responding — channel mismatch | botpress-chat-integration.md |
| Poll has bot message but filter rejects | `afterTime` is client-side — use server timestamp | http-polling.md |
| ASR Error 3 in desktop | Simulator does not support ASR | asr-voice-input.md |
| ASR Error 3 on device | Mic permission not granted | asr-voice-input.md |
| TTS audio plays, subtitles blank | `isTyping` not set or `UpdateEvent` not wired | tts-typewriter-hud.md |
| Subtitles finish before audio | `charsPerSecond` too high | tts-typewriter-hud.md |
| Story auto-advances too fast | `autoAdvanceDelay` too short, or `isBusy()` not checked | tts-typewriter-hud.md |

## Logging Patterns

### Spawner state dump
```typescript
debugSpawner(): void {
  this.registry.forEach((obj, key) => {
    const pos = obj.getTransform().getWorldPosition()
    print(`[Spawner] "${key}" enabled=${obj.enabled} pos=(${pos.x.toFixed(1)},${pos.y.toFixed(1)},${pos.z.toFixed(1)})`)
  })
  print('[Spawner] poolRoot.enabled=' + this.poolRoot.enabled)
}
```

### Poll debug output
```typescript
// In poll(), after parsing messages:
print(`[Poll] attempt=${attempt} msgs=${messages.length} afterTime=${afterTime}`)
messages.forEach(m =>
  print(`  userId=${m.userId?.slice(0,12)} type=${m.payload?.type} isBot=${m.userId !== myUserId} newer=${m.createdAt > afterTime}`)
)
```

### ASR state
```typescript
// In onTranscriptionErrorEvent handler:
const labels = { 1: 'GENERIC', 2: 'NETWORK', 3: 'NO_SPEECH', 4: 'NOT_AVAILABLE' }
print('[ASR] Error ' + code + ' (' + (labels[code] ?? 'unknown') + ')')
```
