# 01 — Architecture Overview

> ByPotomac SDK · System Architecture

---

## High-Level Architecture

ByPotomac follows a **three-tier architecture** with two client applications connecting to a single backend API, which itself connects to Supabase for persistence and Anthropic Claude for AI capabilities.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT TIER                                    │
│                                                                             │
│   ┌─────────────────────────┐       ┌─────────────────────────────────┐    │
│   │   Web App (Next.js 16)  │       │   Windows App (WinUI 3 / C#)   │    │
│   │                         │       │                                 │    │
│   │ • React 19              │       │ • .NET 8                        │    │
│   │ • Vercel AI SDK v6      │       │ • Windows App SDK 1.8           │    │
│   │ • Tailwind CSS          │       │ • MSIX Packaged                 │    │
│   │ • Radix UI / shadcn     │       │ • NavigationView Shell          │    │
│   │ • App Router (SSR)      │       │ • Mica Backdrop                 │    │
│   │                         │       │                                 │    │
│   │ Hosted: Vercel          │       │ Installed: Windows 10+          │    │
│   │ analystbypotomac.       │       │                                 │    │
│   │ vercel.app              │       │                                 │    │
│   └───────────┬─────────────┘       └───────────────┬─────────────────┘    │
│               │                                     │                      │
└───────────────┼─────────────────────────────────────┼──────────────────────┘
                │          HTTPS / SSE                 │
                ▼                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API TIER                                       │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     FastAPI Application                              │  │
│   │                     Python 3.11+ / Uvicorn                          │  │
│   │                                                                     │  │
│   │  ┌─────────────────────────────────────────────────────────┐       │  │
│   │  │                   Middleware Layer                        │       │  │
│   │  │  • CORS (whitelisted origins)                           │       │  │
│   │  │  • Rate Limiting (120 req/min per IP, in-memory)        │       │  │
│   │  │  • Global Exception Handler                             │       │  │
│   │  └─────────────────────────────────────────────────────────┘       │  │
│   │                                                                     │  │
│   │  ┌─────────────────────────────────────────────────────────┐       │  │
│   │  │                    20 Route Modules                      │       │  │
│   │  │                                                         │       │  │
│   │  │  auth · chat · ai · brain · skills · researcher         │       │  │
│   │  │  edgar · backtest · presentations · files · upload      │       │  │
│   │  │  content · train · admin · afl · health · yfinance      │       │  │
│   │  │  reverse_engineer · pptx_engine · pptx_generate         │       │  │
│   │  └─────────────────────────────────────────────────────────┘       │  │
│   │                                                                     │  │
│   │  ┌─────────────────────────────────────────────────────────┐       │  │
│   │  │                    Core Engine Layer                      │       │  │
│   │  │                                                         │       │  │
│   │  │  ClaudeEngine         → Anthropic Messages API          │       │  │
│   │  │  ClaudeEngineWrapper  → Tool orchestration loop         │       │  │
│   │  │  VercelAIStreamProtocol → Data Stream encoding          │       │  │
│   │  │  ContextManager       → Conversation memory             │       │  │
│   │  │  KnowledgeBase        → RAG with pgvector               │       │  │
│   │  │  SkillGateway         → Skill routing                   │       │  │
│   │  │  ResearcherEngine     → Multi-source research           │       │  │
│   │  └─────────────────────────────────────────────────────────┘       │  │
│   │                                                                     │  │
│   │  Hosted: Railway                                                   │  │
│   │  URL: potomac-analyst-workbench-production.up.railway.app          │  │
│   │  Port: 8070                                                        │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            DATA TIER                                        │
│                                                                             │
│   ┌──────────────────────────────────────┐  ┌────────────────────────────┐ │
│   │         Supabase Platform            │  │    External APIs           │ │
│   │                                      │  │                            │ │
│   │  • PostgreSQL 15 + pgvector          │  │  • Anthropic Claude API    │ │
│   │  • Supabase Auth (GoTrue)            │  │  • Tavily Search API       │ │
│   │  • Storage Buckets                   │  │  • SEC EDGAR               │ │
│   │  • Row Level Security (RLS)          │  │  • Yahoo Finance           │ │
│   │  • Edge Functions                    │  │  • Finnhub                 │ │
│   │                                      │  │  • FRED                    │ │
│   │                                      │  │  • NewsAPI                 │ │
│   └──────────────────────────────────────┘  └────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Component Inventory

### Backend (`/api` + `/core` + `/db`)

| Component | Location | Purpose |
|-----------|----------|---------|
| `main.py` | Root | FastAPI app bootstrap, middleware, router registration |
| `config.py` | Root | Pydantic `BaseSettings`, all env vars |
| `api/routes/` | 18 files | HTTP endpoint handlers |
| `api/dependencies.py` | API | `get_current_user` JWT dependency |
| `core/claude_engine.py` | Core | Direct Anthropic API streaming client |
| `core/claude_engine_wrapper.py` | Core | Tool loop orchestrator |
| `core/vercel_ai.py` | Core | Vercel AI SDK Data Stream Protocol encoder |
| `core/streaming.py` | Core | `StreamingResponse` helpers |
| `core/ui_message_stream.py` | Core | UI Message Stream Protocol encoder |
| `core/tools.py` | Core | 11 AI tool definitions |
| `core/skills.py` | Core | Skill discovery + execution |
| `core/skill_gateway.py` | Core | Skill routing hub |
| `core/context_manager.py` | Core | Conversation context builder |
| `core/knowledge_base.py` | Core | RAG: embedding + semantic search |
| `core/researcher_engine.py` | Core | Multi-source research pipeline |
| `core/researcher.py` | Core | Research session management |
| `core/edgar_client.py` | Core | SEC EDGAR filing fetcher |
| `core/encryption.py` | Core | Fernet AES-256 key encryption |
| `core/storage.py` | Core | Supabase Storage bucket operations |
| `core/training.py` | Core | Training data CRUD |
| `core/document_parser.py` | Core | PDF/DOCX/CSV text extraction |
| `core/docx_generator.py` | Core | DOCX report generation |
| `core/pptx_generator.py` | Core | PPTX presentation generation |
| `db/supabase_client.py` | DB | Supabase client singleton |
| `pptx_engine/` | Engine | Full presentation generation pipeline |

### Web Frontend (`/src`)

| Component | Location | Purpose |
|-----------|----------|---------|
| `src/app/` | App Router | 15+ page routes |
| `src/app/api/` | API Routes | Proxy layer for chat streaming |
| `src/lib/api.ts` | Library | `APIClient` class, all backend calls |
| `src/contexts/AuthContext.tsx` | Context | Auth state, login/logout/register |
| `src/components/` | Components | Shared UI (shadcn/ui + custom) |
| `src/hooks/` | Hooks | Custom React hooks |

### Native Windows App

| Component | Location | Purpose |
|-----------|----------|---------|
| `App.xaml.cs` | Root | DI container, singleton services |
| `Services/ApiService.cs` | Services | HTTP client, streaming parser |
| `Services/AuthService.cs` | Services | Login/register, token management |
| `Services/SessionService.cs` | Services | Token + user state |
| `Pages/` | Pages | 16 page views |
| `Models/` | Models | DTOs matching backend schemas |

---

## Data Flow Patterns

### 1. Standard REST Request

```
Client → HTTP Request (Bearer JWT) → FastAPI Route → Supabase → Response
```

### 2. AI Chat Streaming

```
Client → POST /chat/stream (Bearer JWT)
       → ClaudeEngineWrapper.stream_response()
         → Build context (conversation history + knowledge base + tools)
         → Anthropic Messages API (streaming)
         → For each chunk:
           → Encode as Vercel AI SDK Data Stream Protocol
           → Yield via StreamingResponse (SSE)
       → Client parses SSE stream
         → Web: Vercel AI SDK useChat() auto-parses
         → Windows: Custom StreamAsync<T> parser
```

### 3. Tool Calling Loop

```
Claude Response contains tool_use block
  → Backend executes tool (e.g., search_sec_filings)
  → Tool result sent back to Claude
  → Claude generates final response
  → All streamed as Data Stream Protocol events
```

### 4. RAG Knowledge Query

```
User message → Embed with Anthropic
             → pgvector similarity search on brain_documents
             → Top-K results injected into Claude system prompt
             → Claude generates contextual response
```

---

## Security Architecture

| Layer | Mechanism |
|-------|-----------|
| **Transport** | HTTPS everywhere (Railway + Vercel SSL) |
| **Authentication** | Supabase Auth (GoTrue), JWT tokens |
| **Authorization** | `get_current_user` FastAPI dependency on all protected routes |
| **API Keys** | User Claude/Tavily keys encrypted with Fernet AES-256 (`enc:` prefix) |
| **Database** | Row Level Security (RLS) policies per table |
| **CORS** | Whitelist of allowed origins |
| **Rate Limiting** | 120 req/min per IP (in-memory) |
| **Admin** | `is_admin` flag checked for admin-only routes |

---

## Key Design Decisions

1. **Vercel AI SDK Data Stream Protocol** — The backend speaks the same streaming protocol as Vercel AI SDK, allowing the Next.js frontend to use `useChat()` directly and the Windows app to parse the same format.

2. **Service-role Supabase client** — The backend uses the `service_role` key to bypass RLS for server-side operations, while RLS still protects direct Supabase access.

3. **User-owned API keys** — Each user can provide their own Claude and Tavily API keys (encrypted at rest). The backend falls back to shared keys if none are provided.

4. **Modular AI tools** — Tools are defined declaratively in `core/tools.py` and executed by `ClaudeEngineWrapper`, making it easy to add new capabilities.

5. **Skills system** — Skills are stored in Supabase and can be dynamically loaded to extend Claude's behavior per-conversation.

---

*Next: [02 — Backend API Reference](02-BACKEND-API.md)*
