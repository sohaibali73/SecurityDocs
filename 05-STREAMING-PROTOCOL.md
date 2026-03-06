# 05 — Streaming Protocol

> ByPotomac SDK · Vercel AI SDK Data Stream Protocol

---

## Overview

The ByPotomac backend streams AI responses using the **Vercel AI SDK Data Stream Protocol**. This is a line-based SSE (Server-Sent Events) format where each line is prefixed with a type code that tells the client what kind of data follows.

Both the web frontend (using Vercel AI SDK `useChat()`) and the native Windows app (custom parser) consume this same protocol.

---

## Protocol Format

Each line in the stream follows this format:

```
{type_code}:{json_value}\n
```

Where `type_code` is a single character (or hex letter) and `json_value` is a JSON-encoded value.

---

## Type Codes

| Code | Name | Payload | Description |
|------|------|---------|-------------|
| `0` | **Text** | `"string"` | A text delta (chunk of the AI response) |
| `2` | **Data** | `[{...}]` | Structured data (metadata, artifacts, charts) |
| `3` | **Error** | `"string"` | Error message |
| `7` | **ToolCallStreamStart** | `{"toolCallId":"...","toolName":"..."}` | A tool call is starting |
| `8` | **ToolCallDelta** | `{"toolCallId":"...","argsTextDelta":"..."}` | Streaming tool call arguments |
| `9` | **ToolCall** | `{"toolCallId":"...","toolName":"...","args":{...}}` | Completed tool call |
| `a` | **ToolResult** | `{"toolCallId":"...","result":{...}}` | Tool execution result |
| `d` | **FinishMessage** | `{"finishReason":"stop","usage":{...}}` | Message complete |
| `e` | **FinishStep** | `{"finishReason":"stop","isContinued":false}` | Step complete |
| `f` | **StartStep** | `{"messageId":"..."}` | New step starting |

---

## Complete Stream Example

Here's a real example of a chat response where Claude uses a tool:

```
f:{"messageId":"msg_01XYZ"}
0:"Let me look up "
0:"the latest AAPL "
0:"financial data.\n\n"
7:{"toolCallId":"toolu_01ABC","toolName":"get_stock_quote"}
8:{"toolCallId":"toolu_01ABC","argsTextDelta":"{\"ticker\""}
8:{"toolCallId":"toolu_01ABC","argsTextDelta":":\"AAPL\"}"}
9:{"toolCallId":"toolu_01ABC","toolName":"get_stock_quote","args":{"ticker":"AAPL"}}
a:{"toolCallId":"toolu_01ABC","result":{"price":187.50,"change":2.3,"volume":45000000}}
e:{"finishReason":"tool-calls","isContinued":true}
f:{"messageId":"msg_01XYZ"}
0:"Based on the data, "
0:"AAPL is currently trading at "
0:"$187.50, up 2.3% today "
0:"with 45M shares traded."
e:{"finishReason":"stop","isContinued":false}
d:{"finishReason":"stop","usage":{"promptTokens":1500,"completionTokens":200}}
```

---

## Stream Without Tool Calls

A simple text response:

```
f:{"messageId":"msg_02DEF"}
0:"The price-to-earnings "
0:"ratio (P/E) is a "
0:"valuation metric that "
0:"compares a company's "
0:"stock price to its "
0:"earnings per share."
e:{"finishReason":"stop","isContinued":false}
d:{"finishReason":"stop","usage":{"promptTokens":800,"completionTokens":50}}
```

---

## Data Events (Type `2`)

Data events carry structured payloads for rich UI rendering:

### Metadata

```
2:[{"conversationId":"uuid-123","messageId":"msg_01"}]
```

### Code Artifacts

```
2:[{"type":"code","language":"python","code":"import pandas as pd\n..."}]
```

### Charts

```
2:[{"type":"chart","chartType":"line","data":[{"date":"2025-01","value":100},...]}]
```

### Mermaid Diagrams

```
2:[{"type":"mermaid","diagram":"graph TD\n  A-->B\n  B-->C"}]
```

---

## Backend Encoder (`core/vercel_ai.py`)

The `VercelAIStreamProtocol` class encodes each type:

```python
class StreamPartType(Enum):
    TEXT = "0"
    DATA = "2"
    ERROR = "3"
    TOOL_CALL_STREAMING_START = "7"
    TOOL_CALL_DELTA = "8"
    TOOL_CALL = "9"
    TOOL_RESULT = "a"
    FINISH_MESSAGE = "d"
    FINISH_STEP = "e"
    START_STEP = "f"

class VercelAIStreamProtocol:
    @staticmethod
    def text(content: str) -> str:
        return f'0:{json.dumps(content)}\n'
    
    @staticmethod
    def data(payload: list) -> str:
        return f'2:{json.dumps(payload)}\n'
    
    @staticmethod
    def error(message: str) -> str:
        return f'3:{json.dumps(message)}\n'
    
    @staticmethod
    def tool_call_start(tool_call_id: str, tool_name: str) -> str:
        return f'7:{json.dumps({"toolCallId": tool_call_id, "toolName": tool_name})}\n'
    
    @staticmethod
    def tool_call_delta(tool_call_id: str, args_delta: str) -> str:
        return f'8:{json.dumps({"toolCallId": tool_call_id, "argsTextDelta": args_delta})}\n'
    
    @staticmethod
    def tool_call(tool_call_id: str, tool_name: str, args: dict) -> str:
        return f'9:{json.dumps({"toolCallId": tool_call_id, "toolName": tool_name, "args": args})}\n'
    
    @staticmethod
    def tool_result(tool_call_id: str, result: Any) -> str:
        return f'a:{json.dumps({"toolCallId": tool_call_id, "result": result})}\n'
    
    @staticmethod
    def finish_message(reason: str = "stop", usage: dict = None) -> str:
        return f'd:{json.dumps({"finishReason": reason, "usage": usage or {}})}\n'
    
    @staticmethod
    def finish_step(reason: str = "stop", continued: bool = False) -> str:
        return f'e:{json.dumps({"finishReason": reason, "isContinued": continued})}\n'
    
    @staticmethod
    def start_step(message_id: str = "") -> str:
        return f'f:{json.dumps({"messageId": message_id})}\n'
```

---

## UI Message Stream Protocol (`/chat/v6`)

The V6 endpoint uses a different format designed for Vercel AI SDK v6's `useChat()`:

```python
class UIMessageStreamProtocol:
    @staticmethod
    def text_delta(content: str) -> str:
        return json.dumps({"type": "text-delta", "textDelta": content}) + "\n"
    
    @staticmethod
    def tool_call_begin(id: str, name: str) -> str:
        return json.dumps({"type": "tool-call-begin", "toolCallId": id, "toolName": name}) + "\n"
    
    @staticmethod
    def tool_call_delta(id: str, delta: str) -> str:
        return json.dumps({"type": "tool-call-delta", "toolCallId": id, "argsTextDelta": delta}) + "\n"
    
    @staticmethod
    def tool_call_end(id: str) -> str:
        return json.dumps({"type": "tool-call-end", "toolCallId": id}) + "\n"
    
    @staticmethod
    def tool_result(id: str, result: Any) -> str:
        return json.dumps({"type": "tool-result", "toolCallId": id, "result": result}) + "\n"
    
    @staticmethod
    def finish(reason: str = "stop", usage: dict = None) -> str:
        return json.dumps({"type": "finish", "finishReason": reason, "usage": usage or {}}) + "\n"
```

---

## Client-Side Parsing

### Web Frontend (Next.js)

The web app has two approaches:

**1. Direct `useChat()` with V6 endpoint:**
```typescript
const { messages, append } = useChat({
  api: '/api/chat',       // Next.js proxy route
  streamProtocol: 'data', // or 'text'
});
```

**2. Next.js proxy route translates Data Stream → UI Message Stream:**
```typescript
// app/api/chat/route.ts
// Reads Data Stream Protocol from backend
// Translates to UI Message Stream for useChat()
export async function POST(req: Request) {
  const response = await fetch(`${API_URL}/chat/stream`, { ... });
  // Parse 0:, 2:, 9:, a:, d: lines
  // Re-encode as text-delta, tool-call-begin, etc.
  return new Response(transformedStream);
}
```

### Native Windows App (C#)

```csharp
// Services/ApiService.cs
public async IAsyncEnumerable<StreamEvent> StreamAsync(string url, object body)
{
    var response = await _httpClient.PostAsync(url, content);
    var stream = await response.Content.ReadAsStreamAsync();
    var reader = new StreamReader(stream);
    
    while (!reader.EndOfStream)
    {
        var line = await reader.ReadLineAsync();
        if (string.IsNullOrEmpty(line)) continue;
        
        var colonIndex = line.IndexOf(':');
        var typeCode = line[0..colonIndex];
        var jsonPayload = line[(colonIndex + 1)..];
        
        switch (typeCode)
        {
            case "0": // Text delta
                yield return new StreamEvent { Type = "text", Text = JsonSerializer.Deserialize<string>(jsonPayload) };
                break;
            case "3": // Error
                yield return new StreamEvent { Type = "error", Text = JsonSerializer.Deserialize<string>(jsonPayload) };
                break;
            case "8": // Metadata (conversationId)
                var meta = JsonSerializer.Deserialize<JsonElement>(jsonPayload);
                yield return new StreamEvent { Type = "metadata", Data = meta };
                break;
            case "9": // Tool call
                yield return new StreamEvent { Type = "tool_call", Data = JsonSerializer.Deserialize<JsonElement>(jsonPayload) };
                break;
            case "a": // Tool result
                yield return new StreamEvent { Type = "tool_result", Data = JsonSerializer.Deserialize<JsonElement>(jsonPayload) };
                break;
            case "d": // Finish
                yield return new StreamEvent { Type = "finish", Data = JsonSerializer.Deserialize<JsonElement>(jsonPayload) };
                break;
        }
    }
}
```

---

## HTTP Response Headers

The streaming endpoints return:

```
Content-Type: text/event-stream; charset=utf-8
Cache-Control: no-cache
Connection: keep-alive
X-Vercel-AI-Data-Stream: v1
```

---

## Comparison of Protocol Versions

| Feature | Data Stream (`/chat/stream`) | UI Message Stream (`/chat/v6`) |
|---------|------------------------------|-------------------------------|
| Format | `{code}:{json}\n` | `{json}\n` (NDJSON) |
| Text | `0:"text"` | `{"type":"text-delta","textDelta":"text"}` |
| Tool Start | `7:{...}` | `{"type":"tool-call-begin",...}` |
| Tool Args | `8:{...}` | `{"type":"tool-call-delta",...}` |
| Tool Result | `a:{...}` | `{"type":"tool-result",...}` |
| Finish | `d:{...}` | `{"type":"finish",...}` |
| Used by | Windows app, direct clients | Vercel AI SDK v6 `useChat()` |

---

*Next: [06 — Web Frontend](06-WEB-FRONTEND.md)*
