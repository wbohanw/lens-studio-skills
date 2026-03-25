# HTTP Polling for External APIs

## Why Polling (No WebSocket)

Lens Studio and Spectacles have no WebSocket API. Persistent connections to external services require a **polling loop**: send a message via HTTP POST, then periodically GET the conversation history until a response appears.

## The Pattern

```
User speaks
    ↓
POST /messages  →  record afterTime (ISO timestamp of sent message)
    ↓
Wait 0.8s
    ↓
GET /messages   →  filter: isBot=true AND createdAt > afterTime
    ↓ (no reply yet)
Wait 1.2s, 1.8s, 2.7s, 4s...
    ↓ (reply found)
Parse response → drive scene
```

## Full Implementation

```typescript
@component
export class ChatClient extends BaseScriptComponent {
  @input internetModule: InternetModule    // Assign RemoteServiceModule in Inspector

  private readonly BASE_URL: string = 'https://chat.botpress.cloud/YOUR_CLIENT_ID'
  private readonly BOT_ID:   string = 'your-bot-id'
  private myUserId:   string = ''
  private userKey:    string = ''
  private convId:     string = ''
  private afterTime:  string = ''
  private pollEvent:  DelayedCallbackEvent | null = null
  private pollDelay:  number = 0.8   // seconds; grows exponentially
  private readonly MAX_DELAY = 16
  private readonly POLL_DELAYS = [0.8, 1.2, 1.8, 2.7, 4.0, 4.0, 4.0]
  private pollAttempt: number = 0

  // ── Send a message ────────────────────────────────────────────────────────

  sendMessage(text: string): void {
    this.cancelPoll()
    this.pollAttempt = 0

    const req = RemoteServiceHttpRequest.create()
    req.url = `${this.BASE_URL}/conversations/${this.convId}/messages`
    ;(req as any).method = 1   // POST
    req.setHeader('Content-Type', 'application/json')
    req.setHeader('x-bot-id',   this.BOT_ID)
    req.setHeader('x-user-key', this.userKey)
    req.body = JSON.stringify({ payload: { type: 'text', text } })

    this.internetModule.performHttpRequest(req, (res: RemoteServiceHttpResponse) => {
      if (res.statusCode >= 200 && res.statusCode < 300) {
        const data = JSON.parse(res.body)
        // afterTime = createdAt of the message we just sent
        this.afterTime = data.message.createdAt
        print('[Chat] Sent. afterTime=' + this.afterTime)
        this.schedulePoll()
      } else {
        print('[Chat] Send failed: HTTP ' + res.statusCode)
      }
    })
  }

  // ── Poll loop ─────────────────────────────────────────────────────────────

  private schedulePoll(): void {
    const delay = this.POLL_DELAYS[
      Math.min(this.pollAttempt, this.POLL_DELAYS.length - 1)
    ]
    this.pollEvent = this.createEvent('DelayedCallbackEvent') as DelayedCallbackEvent
    this.pollEvent.bind(() => this.poll())
    this.pollEvent.reset(delay)
  }

  private poll(): void {
    const req = RemoteServiceHttpRequest.create()
    req.url = `${this.BASE_URL}/conversations/${this.convId}/messages`
    ;(req as any).method = 0   // GET
    req.setHeader('x-bot-id',   this.BOT_ID)
    req.setHeader('x-user-key', this.userKey)

    this.internetModule.performHttpRequest(req, (res: RemoteServiceHttpResponse) => {
      if (res.statusCode !== 200) {
        this.pollAttempt++
        this.schedulePoll()
        return
      }

      const data     = JSON.parse(res.body)
      const messages = (data.messages ?? []) as any[]

      const reply = messages.find(m =>
        m.userId   !== this.myUserId &&   // not from me
        m.payload?.type === 'text'    &&   // text payload
        m.createdAt > this.afterTime       // arrived after I sent
      )

      if (reply) {
        print('[Chat] Bot replied: ' + reply.payload.text.substring(0, 80))
        this.onBotReply(reply.payload.text)
      } else {
        this.pollAttempt++
        if (this.pollAttempt < this.POLL_DELAYS.length) {
          this.schedulePoll()
        } else {
          print('[Chat] Poll timed out after ' + this.POLL_DELAYS.length + ' attempts')
        }
      }
    })
  }

  private cancelPoll(): void {
    if (this.pollEvent) {
      this.pollEvent.enabled = false
      this.pollEvent = null
    }
    this.pollAttempt = 0
  }

  // Override in subclass or set as callback
  onBotReply(text: string): void {
    print('[Chat] Received: ' + text)
  }
}
```

## `HttpRequestMethod` Numeric Values

Lens Studio's `RemoteServiceHttpRequest.method` uses integers, not strings:

| Value | Method |
|---|---|
| `0` | GET |
| `1` | POST |
| `2` | PUT |
| `3` | DELETE |

Some versions expose `RemoteServiceHttpRequest.HttpRequestMethod.Post` — use whichever your Lens Studio version supports. The numeric cast `(req as any).method = 1` works universally.

## ISO Timestamp Filtering

`m.createdAt > this.afterTime` is a **string comparison**. It works correctly for ISO 8601 UTC timestamps (`2024-01-01T12:00:00.000Z`) because they sort lexicographically in chronological order — as long as both timestamps come from the same server with consistent formatting.

This works: `"2024-01-01T12:00:01.000Z" > "2024-01-01T12:00:00.000Z"` → `true`

> **Pitfall:** Using `new Date().toISOString()` on the Lens Studio client for `afterTime` can fail if the device clock differs from the server clock by more than a few milliseconds. Always use the `createdAt` returned from the POST response, not a locally generated timestamp.

## Debugging the Poll

If the poll always times out, add this logging block to see exactly what messages are being filtered and why:

```typescript
print(`[Chat] Poll attempt ${this.pollAttempt}: got ${messages.length} msgs`)
print(`[Chat] afterTime=${this.afterTime}  myUserId=${this.myUserId}`)
messages.forEach(m => {
  print([
    `  id=${m.id.substring(0,8)}`,
    `userId=${m.userId?.substring(0,12)}`,
    `type=${m.payload?.type}`,
    `createdAt=${m.createdAt}`,
    `isBot=${m.userId !== this.myUserId}`,
    `newer=${m.createdAt > this.afterTime}`
  ].join(' '))
})
```

Common failure patterns from the log:

| Pattern | Cause | Fix |
|---|---|---|
| `got 1 msgs, isBot=false` | Only user's own message; bot never replied | Check Botpress ADK channel name (`botpress-chat-integration.md`) |
| `isBot=true, newer=false` | Bot replied but timestamp is same as sent | Use `>=` comparison, or use server-returned `afterTime` |
| `got 0 msgs` | Wrong conversation ID or network error | Verify `convId` and HTTP headers |
| `type=undefined` | Bot sent a non-text payload | Check ADK handler is sending `{ type: 'text', text: '...' }` |

## Session Initialisation

Before sending messages, create a user and conversation via the Botpress Chat API:

```typescript
async initSession(): Promise<void> {
  // 1. Create user
  const userRes = await this.post('/users', {})
  this.myUserId = userRes.user.id
  this.userKey  = userRes.key

  // 2. Create conversation
  const convRes = await this.post('/conversations', {})
  this.convId = convRes.conversation.id

  print('[Chat] Session ready. userId=' + this.myUserId)
}

private post(path: string, body: object): Promise<any> {
  return new Promise((resolve, reject) => {
    const req = RemoteServiceHttpRequest.create()
    req.url = this.BASE_URL + path
    ;(req as any).method = 1
    req.setHeader('Content-Type', 'application/json')
    req.setHeader('x-bot-id',    this.BOT_ID)
    if (this.userKey) req.setHeader('x-user-key', this.userKey)
    req.body = JSON.stringify(body)
    this.internetModule.performHttpRequest(req, res => {
      if (res.statusCode >= 200 && res.statusCode < 300) {
        resolve(JSON.parse(res.body))
      } else {
        reject(new Error('HTTP ' + res.statusCode + ': ' + res.body))
      }
    })
  })
}
```
