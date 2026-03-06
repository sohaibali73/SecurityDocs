# 04 — AI Engine

> ByPotomac SDK · Claude AI Integration & Tool Calling

---

## Overview

The AI engine is the core of ByPotomac. It wraps the **Anthropic Claude Messages API** with streaming support, tool calling, context management, and the Vercel AI SDK Data Stream Protocol for client consumption.

```
┌───────────────────────────────────────────────────────────────────┐
│                        AI Engine Stack                             │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                   Chat Route Handler                         │ │
│  │                   (api/routes/chat.py)                      │ │
│  └──────────────────────────┬──────────────────────────────────┘ │
│                              │                                    │
│  ┌──────────────────────────▼──────────────────────────────────┐ │
│  │              ClaudeEngineWrapper                             │ │
│  │              (core/claude_engine_wrapper.py)                 │ │
│  │                                                             │ │
│  │  • Builds system prompt (base + skills + training)          │ │
│  │  • Builds context (history + knowledge base)                │ │
│  │  • Registers tools                                          │ │
│  │  • Runs tool calling loop                                   │ │
│  │  • Encodes output as Data Stream Protocol                   │ │
│  └──────────────────────────┬──────────────────────────────────┘ │
│                              │                                    │
│  ┌──────────────────────────▼──────────────────────────────────┐ │
│  │              ClaudeEngine                                    │ │
│  │              (core/claude_engine.py)                         │ │
│  │                                                             │ │
│  │  • Direct Anthropic SDK client                              │ │
│  │  • Streaming messages.create()                              │ │
│  │  • Handles content_block_start/delta/stop                   │ │
│  └──────────────────────────┬──────────────────────────────────┘ │
│                              │                                    │
│  ┌──────────────────────────▼──────────────────────────────────┐ │
│  │              Anthropic Messages API                          │ │
│  │              (claude-sonnet-4-20250514)                      │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

---

## ClaudeEngine (`core/claude_engine.py`)

The low-level Anthropic API client.

### Key Methods

```python
class ClaudeEngine:
    def __init__(self, api_key: str, model: str = "claude-sonnet-4-20250514"):
        self.client = anthropic.Anthropic(api_key=api_key)
        self.model = model
    
    async def stream_message(
        self,
        messages: list[dict],
        system: str = "",
        tools: list[dict] = None,
        max_tokens: int = 8192,
        temperature: float = 0.7
    ) -> AsyncGenerator:
        """
        Calls Anthropic messages.create(stream=True) and yields events:
        - message_start
        - content_block_start (text or tool_use)
        - content_block_delta (text_delta or input_json_delta)
        - content_block_stop
        - message_delta (stop_reason)
        - message_stop
        """
    
    async def generate(
        self,
        messages: list[dict],
        system: str = "",
        max_tokens: int = 4096
    ) -> str:
        """Non-streaming single response."""
```

### Model Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `model` | `claude-sonnet-4-20250514` | Claude model ID |
| `max_tokens` | `8192` | Maximum response tokens |
| `temperature` | `0.7` | Response randomness |
| `top_p` | `0.9` | Nucleus sampling |

---

## ClaudeEngineWrapper (`core/claude_engine_wrapper.py`)

The high-level orchestrator that manages the full AI response lifecycle.

### Core Method: `stream_response()`

```python
class ClaudeEngineWrapper:
    async def stream_response(
        self,
        messages: list[dict],
        user_id: str,
        conversation_id: str = None,
        model: str = None,
        system_prompt: str = None,
        skills: list[str] = None,
        brain_enabled: bool = True,
        tools_enabled: bool = True
    ) -> AsyncGenerator[str, None]:
        """
        Full streaming response pipeline:
        
        1. Build system prompt
        2. Load active skills → append to system prompt
        3. Load training examples → append as few-shot examples
        4. If brain_enabled: query knowledge base → inject context
        5. Build message history from conversation
        6. Register available tools
        7. Call Claude with streaming
        8. If Claude requests tool_use → execute tool → feed result back
        9. Encode everything as Vercel AI SDK Data Stream Protocol
        10. Yield SSE chunks to client
        """
```

### System Prompt Construction

The system prompt is built in layers:

```
┌─────────────────────────────────────┐
│  Base System Prompt                 │  ← "You are Analyst by Potomac..."
│  (core/prompts/base.py)            │
├─────────────────────────────────────┤
│  Active Skills                      │  ← Dynamically loaded from Supabase
│  "When analyzing SEC filings..."    │
├─────────────────────────────────────┤
│  Training Examples                  │  ← Few-shot examples from training data
│  "Example: Q: What is PE? A: ..."   │
├─────────────────────────────────────┤
│  Knowledge Base Context             │  ← RAG results from brain_documents
│  "Based on your documents: ..."     │
├─────────────────────────────────────┤
│  Custom System Prompt               │  ← User-provided per-conversation
│  (if any)                           │
└─────────────────────────────────────┘
```

### Tool Calling Loop

When Claude responds with a `tool_use` content block, the wrapper executes the tool and continues:

```
┌────────┐                    ┌─────────────┐                    ┌───────────┐
│ Client │                    │   Wrapper   │                    │  Claude   │
└───┬────┘                    └──────┬──────┘                    └─────┬─────┘
    │  Send message                  │                                 │
    │───────────────────────────────▶│  Build context + tools           │
    │                                │────────────────────────────────▶│
    │                                │                                 │
    │  Stream: text tokens           │◀─── text delta ────────────────│
    │◀───────────────────────────────│                                 │
    │                                │                                 │
    │  Stream: tool_call_start       │◀─── tool_use block ────────────│
    │◀───────────────────────────────│                                 │
    │                                │                                 │
    │                                │  Execute tool locally           │
    │                                │  (e.g., search_sec_filings)     │
    │                                │                                 │
    │  Stream: tool_result           │                                 │
    │◀───────────────────────────────│  Send tool_result to Claude     │
    │                                │────────────────────────────────▶│
    │                                │                                 │
    │  Stream: more text tokens      │◀─── text delta (with findings)─│
    │◀───────────────────────────────│                                 │
    │                                │                                 │
    │  Stream: finish                │◀─── message_stop ──────────────│
    │◀───────────────────────────────│                                 │
```

---

## VercelAIGatewayClient (`core/vercel_ai.py`)

An alternative streaming client that supports both direct Anthropic API and Vercel AI Gateway modes.

### Key Features

- **Message format conversion** — Converts Vercel AI SDK message format (with `toolInvocations`) to Anthropic format (`tool_use`/`tool_result` content blocks)
- **Tool format conversion** — Converts AI SDK function tools to Anthropic's `name/description/input_schema` format
- **Stream encoding** — Maps Anthropic streaming events to Data Stream Protocol type codes

### Supported Stream Modes

1. **Data Stream Protocol** (`/chat/stream`) — Type-prefixed lines (`0:`, `2:`, `9:`, etc.)
2. **UI Message Stream** (`/chat/v6`) — JSON events with `type` field for Vercel AI SDK v6

---

## Context Manager (`core/context_manager.py`)

Manages conversation context for Claude:

```python
class ContextManager:
    def build_context(
        self,
        conversation_id: str,
        user_id: str,
        max_messages: int = 50,
        max_tokens: int = 100000
    ) -> list[dict]:
        """
        1. Load conversation messages from Supabase
        2. Trim to max_messages (most recent)
        3. Ensure total tokens < max_tokens
        4. Format as Anthropic messages array
        """
```

### Token Management

- Default context window: **100,000 tokens** (Claude's context limit is 200K)
- Messages are trimmed from oldest first
- System prompt + tools + knowledge context all count toward the budget

---

## Knowledge Base Integration (RAG)

When `brain_enabled: true`, the wrapper queries the knowledge base before calling Claude:

```python
class KnowledgeBase:
    async def query(self, text: str, user_id: str, top_k: int = 5) -> list[dict]:
        """
        1. Embed the query text using Anthropic embeddings
        2. Run pgvector cosine similarity search on brain_document_chunks
        3. Return top-K matching chunks with similarity scores
        """
    
    async def add_document(self, file, user_id: str) -> dict:
        """
        1. Parse document (PDF/DOCX/CSV/TXT)
        2. Split into chunks (1000 tokens, 200 overlap)
        3. Embed each chunk
        4. Store in brain_document_chunks with vector
        """
```

The retrieved chunks are injected into the system prompt:

```
[Knowledge Base Context]
The following information from your documents may be relevant:

Document: "Q4 Report" (similarity: 0.89)
---
The risk-adjusted return for the quarter was 12.3%...
---

Document: "Strategy Guide" (similarity: 0.85)
---
The fund employs a multi-factor approach...
---
```

---

## API Key Resolution

The engine resolves which API key to use:

```python
def get_api_key(user_id: str) -> str:
    # 1. Check user's encrypted key in user_profiles
    profile = supabase.table("user_profiles").select("claude_api_key").eq("id", user_id).single()
    if profile and profile["claude_api_key"]:
        return encryption_manager.decrypt(profile["claude_api_key"])
    
    # 2. Fall back to system key
    return settings.anthropic_api_key
```

---

## Error Handling

The engine handles several failure modes:

| Error | Behavior |
|-------|----------|
| **Invalid API key** | Returns error stream event (`3:` type) |
| **Rate limited by Anthropic** | Retries with exponential backoff |
| **Tool execution failure** | Sends error as tool result, Claude adapts |
| **Context too long** | Trims older messages automatically |
| **Network timeout** | Returns error event, client can retry |

---

## Model Support

| Model ID | Description | Use Case |
|----------|-------------|----------|
| `claude-sonnet-4-20250514` | Default model | General analysis, chat |
| `claude-3-5-haiku-20241022` | Fast, cheaper | Quick queries |
| `claude-3-5-sonnet-20241022` | Previous gen | Fallback |

The model is configurable per-request via the `model` field in the chat request body.

---

*Next: [05 — Streaming Protocol](05-STREAMING-PROTOCOL.md)*
