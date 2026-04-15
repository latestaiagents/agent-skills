---
name: agent-sdk-streaming
description: |
  Stream agent output correctly — text deltas, tool-use events, thinking deltas, progress indicators — without dropping events or blocking on long tool calls. Covers backpressure, error handling, and UX patterns.
  Use this skill when building agent UIs (chat, CLI), ensuring agents feel responsive, or debugging dropped/truncated streams.
  Activate when: agent streaming, stream events, text deltas, streaming UI, SSE agent, stream tool use.
---

# Agent SDK Streaming

**Streaming is the difference between "this agent works" and "this agent feels alive". Stream text as it generates, show tool activity, and never block the UI on a long tool call.**

## When to Use

- Building any interactive agent UI
- Long-running agents where users need to see progress
- Debugging "why does my agent hang?" (often: you're not reading the stream)

## Event Types

The SDK emits a typed stream of events. Core types:

| Event | When | UI treatment |
|---|---|---|
| `assistant` with text block | Model generates text | Append to current message bubble |
| `assistant` with thinking | Extended thinking | Show "thinking..." spinner |
| `assistant` with tool_use | Model decides to call tool | Show tool badge ("calling web_fetch...") |
| `user` with tool_result | Tool returned | Show result or just acknowledge |
| `system` with progress | Tool progress notification | Update progress bar |
| `result` | Turn complete | Remove spinner, finalize |

## TypeScript Streaming

```ts
import { query } from "@anthropic-ai/claude-agent-sdk";

const ui = new ChatUI();

for await (const msg of query({ prompt, options: { model: "claude-sonnet-4-6" } })) {
  switch (msg.type) {
    case "assistant": {
      for (const block of msg.message.content) {
        if (block.type === "text") ui.appendText(block.text);
        if (block.type === "thinking") ui.showThinking(block.thinking);
        if (block.type === "tool_use") ui.showToolCall(block.name, block.input);
      }
      break;
    }
    case "user": {
      for (const block of msg.message.content) {
        if (block.type === "tool_result") ui.showToolResult(block.tool_use_id, block.content);
      }
      break;
    }
    case "result": {
      ui.finalize(msg.result);
      break;
    }
  }
}
```

## Text Deltas vs Messages

Some SDK versions emit full messages (with cumulative text) rather than deltas. Handle both:

```ts
let lastText = "";
// on assistant text:
const newText = block.text.slice(lastText.length);
ui.appendText(newText);
lastText = block.text;
```

Check your SDK version's docs; deltas reduce UI work but complete messages simplify state.

## Streaming to a Web Client

For web UIs, bridge Node stream → SSE:

```ts
app.get("/chat", async (req, res) => {
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.flushHeaders();

  for await (const msg of query({ prompt: req.query.q, options: {...} })) {
    res.write(`data: ${JSON.stringify(msg)}\n\n`);
    if (typeof (res as any).flush === "function") (res as any).flush();
  }
  res.end();
});
```

Gotchas:

- Disable proxy buffering (`X-Accel-Buffering: no`) if behind nginx
- Flush after every write — otherwise the browser sees nothing for 30s
- Use heartbeat comments (`: ping\n\n`) every 15s to prevent proxy idle timeouts

## React Hook Pattern

```tsx
function useAgent() {
  const [messages, setMessages] = useState([]);
  const [streaming, setStreaming] = useState(false);

  const send = async (prompt: string) => {
    setStreaming(true);
    const res = await fetch("/chat?q=" + encodeURIComponent(prompt));
    const reader = res.body!.getReader();
    const decoder = new TextDecoder();
    let buf = "";
    while (true) {
      const { value, done } = await reader.read();
      if (done) break;
      buf += decoder.decode(value, { stream: true });
      const events = buf.split("\n\n");
      buf = events.pop()!;
      for (const evt of events) {
        if (evt.startsWith("data: ")) {
          const msg = JSON.parse(evt.slice(6));
          setMessages((m) => reduceAgentMsg(m, msg));
        }
      }
    }
    setStreaming(false);
  };

  return { messages, streaming, send };
}
```

## Backpressure

If the UI renders slower than events arrive:

- Coalesce adjacent text deltas into one render frame (throttle to 30 FPS)
- Batch tool-use and tool-result events into grouped UI updates
- For huge responses, consider virtualization (react-virtuoso) so DOM stays light

## Cancellation

Users change their mind. Support interrupting a running agent:

```ts
const controller = new AbortController();

for await (const msg of query({
  prompt,
  options: { abortController: controller },
})) { /* ... */ }

// elsewhere:
cancelButton.onclick = () => controller.abort();
```

Always cancel gracefully — show "Stopped by user" instead of a broken state.

## Error Handling

```ts
try {
  for await (const msg of query({ prompt, options })) { /* ... */ }
} catch (err) {
  if (err.name === "AbortError") ui.showStopped();
  else if (err.status === 429) ui.showRateLimit();
  else ui.showError(err.message);
}
```

Don't silently fail — users need to know whether to retry.

## Showing Tool Activity

Raw tool calls are noisy. Map them to user-friendly UI:

```ts
const TOOL_LABELS: Record<string, string> = {
  web_fetch: "Looking up the web",
  code_execution: "Running code",
  Task: "Delegating research",
};

ui.showToolCall = (name, input) => {
  ui.addStatusLine(TOOL_LABELS[name] ?? `Running ${name}...`);
};
```

Collapse details into an expandable disclosure — power users want to see the raw call; others don't.

## Anti-Patterns

1. **Awaiting the full response** then rendering — defeats streaming entirely
2. **No flush** on server — browser gets nothing until buffer fills
3. **No cancellation path** — users stuck watching a runaway agent
4. **Rendering every text delta synchronously** — UI jank on long responses
5. **Logging nothing on stream end** — can't diagnose partial failures

## Best Practices

1. Stream everything — text, thinking, tool calls, results
2. Flush after every SSE write; set `X-Accel-Buffering: no` at proxies
3. Coalesce text deltas to 30 FPS rendering
4. Support abort with a cancel button
5. Map tool calls to user-friendly labels; hide internals behind disclosure
6. Emit heartbeats on long tool waits so connections don't die
7. Show distinct visual states for thinking vs text vs tool use
