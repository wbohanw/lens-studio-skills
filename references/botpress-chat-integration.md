# Botpress Chat API Integration

## The Silent Drop Bug

When a Spectacles app sends messages to a Botpress bot via the Chat REST API, the bot receives the messages — you can confirm this because `POST /messages` returns a `201` with a valid message ID. But the bot **never replies**.

The cause: the ADK conversation handler is registered on `channel: "*"` instead of `channel: "chat.channel"`. The wildcard does not match messages from the Chat integration. Messages arrive, find no matching handler, and are silently dropped.

## The Fix

```typescript
// src/conversations/index.ts

// ❌ WRONG — wildcard does not match the Chat integration
export default new Conversation({
  channel: '*',
  handler: async ({ execute }) => {
    await execute({ instructions: SYSTEM_PROMPT })
  }
})

// ✅ CORRECT
export default new Conversation({
  channel: 'chat.channel',
  handler: async ({ execute }) => {
    await execute({ instructions: SYSTEM_PROMPT })
  }
})
```

After fixing, redeploy:

```bash
adk deploy
```

## Diagnosing the Bug

Add debug logging to the HTTP polling loop (see `http-polling.md`). The pattern that reveals this bug:

```
[Chat] Poll attempt 0: got 1 msgs
[Chat] afterTime=2024-01-01T12:00:00.000Z  myUserId=user_abc123
  id=3d031ee3 userId=user_abc123 type=text createdAt=2024-01-01T12:00:00.000Z isBot=false newer=false
[Chat] Poll attempt 1: got 1 msgs
...same...
[Chat] Poll timed out after 7 attempts
```

**Exactly 1 message, always the user's own.** The bot received the message but no conversation handler fired — zero bot replies in the conversation history.

## Botpress Chat vs Webchat

Botpress has two channel integrations that both appear in `agent.config.ts`:

```typescript
dependencies: {
  integrations: {
    chat:    { version: 'chat@0.7.6',    enabled: true },   // Chat REST API
    webchat: { version: 'webchat@0.3.0', enabled: true },   // Browser webchat widget
  }
}
```

| Integration | Channel ID | Used by |
|---|---|---|
| Chat REST API | `chat.channel` | Spectacles app, mobile apps, custom clients |
| Webchat widget | `webchat.hitl` or `*` | Browser embedded chat |

The Spectacles app uses the **Chat REST API**, so the conversation handler must target `"chat.channel"`.

## Handling Multiple Modes in One Handler

The Spectacles app sends prefixed messages to route between story mode and agent mode:

```typescript
export default new Conversation({
  channel: 'chat.channel',
  handler: async ({ execute, message }) => {
    const text    = (message as any)?.payload?.text ?? ''
    const isAgent = text.toLowerCase().startsWith('agent:')

    await execute({
      instructions: isAgent ? AGENT_SYSTEM_PROMPT : STORY_SYSTEM_PROMPT,
    })
  }
})
```

The `execute()` call uses the conversation history and the provided system prompt to generate a reply, then sends it back to the conversation automatically.

## Structured JSON Responses

For story beats, the bot returns structured JSON that the Lens side parses into scene commands. Configure this in the system prompt:

```typescript
const STORY_SYSTEM_PROMPT = `
You are a children's AR story narrator. Reply ONLY with valid JSON in this exact format:
{
  "narration": "The text to display and speak aloud",
  "commands": [
    { "type": "spawn",  "object": "fox",  "position": "front" },
    { "type": "effect", "name": "fireflies" },
    { "type": "remove", "object": "owl" },
    { "type": "clear" }
  ],
  "beat": 1,
  "isEnd": false
}

AVAILABLE 3D OBJECTS: bear, fox, wolf, deer, rabbit, squirrel, owl, hedgehog, boar
AVAILABLE EFFECTS: fireflies, snow, rain, rainbow, fireworks, hearts, stars
POSITIONS: front, left, right, above, center
`
```

## Parsing the Response on the Lens Side

The LLM sometimes wraps JSON in a markdown code fence. Handle both:

```typescript
parseStoryBeat(rawText: string): StoryBeat {
  // Try to extract from ```json ... ``` fence
  const fenceMatch = rawText.match(/```(?:json)?\s*([\s\S]*?)```/i)
  const candidate  = fenceMatch ? fenceMatch[1].trim() : null

  for (const text of candidate ? [candidate, rawText.trim()] : [rawText.trim()]) {
    const jsonMatch = text.match(/\{[\s\S]*\}/)
    const toTry     = jsonMatch ? jsonMatch[0] : text
    try {
      const parsed = JSON.parse(toTry)
      return {
        narration: String(parsed.narration ?? toTry),
        commands:  Array.isArray(parsed.commands) ? parsed.commands : [],
        beat:      Number(parsed.beat ?? 1),
        isEnd:     Boolean(parsed.isEnd ?? false),
      }
    } catch {
      // try next candidate
    }
  }

  // Fallback: treat as plain narration
  return { narration: rawText.trim(), commands: [], beat: 1, isEnd: false }
}
```

## Deploy and Test Workflow

```bash
# In the XRchatbot/ directory
adk deploy          # push latest handler to Botpress Cloud

# Quick test without Spectacles
adk chat            # interactive CLI chat — confirms the handler fires
```

Test with `adk chat` first. If the bot responds there, the `chat.channel` handler is working and the issue is on the Lens side. If the bot does not respond in `adk chat`, the handler code has an error — check ADK deploy output for compilation errors.
