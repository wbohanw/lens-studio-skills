<img width="1098" height="465" alt="image" src="https://github.com/user-attachments/assets/6694e827-4538-4b97-9246-c5549db2dab1" />


# lens-studio-spectacles-ar

> An AI agent skill for building production AR experiences on Snap Spectacles in Lens Studio.

```
npx skills add wbohanw/lens-studio-skills
```



This skill encodes hard-won patterns from building a real Spectacles app end-to-end: AI-driven 3D animal spawning, chatbot integration over HTTP, TTS narration with typewriter subtitles, voice command input, and live scene editing via MCP.

It is structured for [skills.sh](https://skills.sh) — a standard format any MCP-compatible AI agent can load and use.

---

## What's Inside

```
SKILL.md                  ← Skill manifest + quick reference table
references/               ← 16 deep-dive reference files
  world-space-anchoring.md
  scene-graph-hierarchy.md
  coordinate-system.md
  fbx-spawner-pattern.md
  idle-animation.md
  http-polling.md
  botpress-chat-integration.md
  asr-voice-input.md
  tts-typewriter-hud.md
  debugging-checklist.md
  sik-buttons.md
  hand-finger-interaction.md
  cards-frames-panels.md
  asset-library.md
  local-asset-spawning.md
  lens-studio-mcp.md        ← Full MCP tool reference
evals/
  evals.json               ← 11 correctness evaluations
eval/
  queries.json             ← 40 trigger/no-trigger queries
```

---

## Topics Covered

| Problem / Topic | Reference |
|---|---|
| Object follows head everywhere | `world-space-anchoring.md` |
| Spawned object invisible at runtime | `scene-graph-hierarchy.md` |
| Spectacles axes and safe spawn positions | `coordinate-system.md` |
| FBX import + visibility-pool spawner | `fbx-spawner-pattern.md` |
| Idle bob/sway animation for static meshes | `idle-animation.md` |
| HTTP polling (no WebSocket) for chatbot APIs | `http-polling.md` |
| Botpress `channel: "*"` vs `"chat.channel"` | `botpress-chat-integration.md` |
| ASR error codes + simulator limitations | `asr-voice-input.md` |
| TTS + typewriter subtitle HUD | `tts-typewriter-hud.md` |
| Invisible/stuck/silent bug triage | `debugging-checklist.md` |
| SUIK `RoundButton` vs deprecated `PinchButton` | `sik-buttons.md` |
| Hand pinch detection + debounce | `hand-finger-interaction.md` |
| SUIK `Frame` panels + `renderOrder` | `cards-frames-panels.md` |
| Asset Library + `.lspkg` import paths | `asset-library.md` |
| `ObjectPrefab.instantiateAsync()` | `local-asset-spawning.md` |
| Full MCP tool reference for Lens Studio 5.15+ | `lens-studio-mcp.md` |

---

## MCP Setup (Lens Studio 5.15+)

To use the MCP reference in practice, install the proxy:

```bash
pip install lens-studio-mcp
```

Add to your Claude Code / Claude Desktop / Cursor config:

```json
{
  "mcpServers": {
    "lens-studio": {
      "command": "lens-studio-mcp",
      "args": ["--transport", "stdio"]
    }
  }
}
```

1. Open Lens Studio 5.15+ with a project loaded
2. Your AI agent calls `connect()` — Lens Studio shows a permission popup
3. Click **Allow** — the token is cached and reused automatically
4. The agent now has full live access to the scene

See `references/lens-studio-mcp.md` for the complete tool reference.

---

## Companion Skill

This skill assumes basic Lens Studio TypeScript knowledge. For the component system, decorators, lifecycle events, and `@input` wiring, see:

**[lens-studio-scripting](https://skills.sh/rolandsmeenk/lensstudioagents/lens-studio-scripting)** — BaseScriptComponent, `onAwake`, `UpdateEvent`, `@component`, `@input`, `@hint`, `getComponent`, cross-script imports.

---

## Evals

The `evals/evals.json` file contains 11 structured evaluations covering:
- World-space anchoring fix
- Visibility pool / invisible object diagnosis
- Botpress channel name bug
- ASR error code identification
- FBX import workflow
- HTTP polling pattern
- SIK buttons (SUIK vs deprecated)
- Hand pinch debounce
- SUIK Frame panels
- Asset Library installation
- ObjectPrefab instantiation

The `eval/queries.json` file contains 40 queries (33 positive triggers, 7 negative) for testing skill routing.

---

## License

MIT
