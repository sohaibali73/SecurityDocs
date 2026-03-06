# 02 — Backend API Reference

> ByPotomac SDK · Complete Endpoint Documentation

**Base URL:** `https://potomac-analyst-workbench-production.up.railway.app`  
**Swagger UI:** `{base}/docs`  
**OpenAPI JSON:** `{base}/openapi.json`

---

## Table of Contents

1. [Authentication](#1-authentication-auth)
2. [Chat](#2-chat-chat)
3. [AI](#3-ai-ai)
4. [Brain / Knowledge Base](#4-brain--knowledge-base-brain)
5. [Skills](#5-skills-skills)
6. [Researcher](#6-researcher-researcher)
7. [Edgar (SEC Filings)](#7-edgar-sec-filings-edgar)
8. [Backtest](#8-backtest-backtest)
9. [Presentations](#9-presentations-presentations)
10. [Files](#10-files-files)
11. [Upload](#11-upload-upload)
12. [Content](#12-content-content)
13. [Train](#13-train-train)
14. [Admin](#14-admin-admin)
15. [AFL Generator](#15-afl-generator-afl)
16. [Health](#16-health-health)
17. [Yahoo Finance](#17-yahoo-finance-yfinance)
18. [Reverse Engineer](#18-reverse-engineer-reverse-engineer)
19. [PPTX Engine](#19-pptx-engine)

---

## Authentication Header

All protected endpoints require:

```
Authorization: Bearer <supabase-jwt-token>
```

The `get_current_user` dependency validates the JWT against Supabase Auth and returns the user dict `{ id, email, is_admin }`.

---

## 1. Authentication (`/auth`)

**Tag:** `Authentication`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/auth/register` | ❌ | Register a new user |
| `POST` | `/auth/login` | ❌ | Login, receive JWT |
| `POST` | `/auth/logout` | ✅ | Logout (server-side acknowledgment) |
| `GET` | `/auth/me` | ✅ | Get current user profile |
| `PUT` | `/auth/me` | ✅ | Update profile (name, nickname, API keys) |
| `PUT` | `/auth/api-keys` | ✅ | Update Claude/Tavily API keys |
| `GET` | `/auth/api-keys` | ✅ | Check which API keys are configured |
| `POST` | `/auth/refresh-token` | ✅ | Refresh JWT token |
| `POST` | `/auth/forgot-password` | ❌ | Send password reset email |
| `POST` | `/auth/reset-password` | ❌ | Reset password with token |
| `PUT` | `/auth/change-password` | ✅ | Change password (authenticated) |
| `GET` | `/auth/admin/users` | ✅🔒 | List all users (admin only) |
| `POST` | `/auth/admin/users/{user_id}/make-admin` | ✅🔒 | Grant admin (admin only) |
| `POST` | `/auth/admin/users/{user_id}/revoke-admin` | ✅🔒 | Revoke admin (admin only) |
| `POST` | `/auth/admin/users/{user_id}/toggle-active` | ✅🔒 | Activate/deactivate user (admin only) |

### Request/Response Models

**Register:**
```json
// POST /auth/register
// Request:
{ "email": "user@example.com", "password": "securepass", "name": "John Doe" }

// Response (Token):
{
  "access_token": "eyJ...",
  "token_type": "bearer",
  "user_id": "uuid",
  "email": "user@example.com",
  "expires_in": 3600
}
```

**Login:**
```json
// POST /auth/login
// Request:
{ "email": "user@example.com", "password": "securepass" }

// Response: same Token model as register
```

**Get Profile:**
```json
// GET /auth/me
// Response (UserResponse):
{
  "id": "uuid",
  "email": "user@example.com",
  "name": "John Doe",
  "nickname": "JD",
  "is_admin": false,
  "is_active": true,
  "has_api_keys": { "claude": true, "tavily": false },
  "created_at": "2026-01-01T00:00:00Z"
}
```

**Update API Keys:**
```json
// PUT /auth/api-keys
// Request:
{ "claude_api_key": "sk-ant-...", "tavily_api_key": "tvly-..." }
// Both fields optional. Keys are Fernet AES-256 encrypted before storage.
```

---

## 2. Chat (`/chat`)

**Tag:** `Chat`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/chat/stream` | ✅ | Stream AI response (Data Stream Protocol) |
| `POST` | `/chat/v6` | ✅ | Stream AI response (UI Message Stream) |
| `GET` | `/chat/conversations` | ✅ | List user's conversations |
| `POST` | `/chat/conversations` | ✅ | Create a new conversation |
| `GET` | `/chat/conversations/{id}` | ✅ | Get conversation with messages |
| `PUT` | `/chat/conversations/{id}` | ✅ | Update conversation title/settings |
| `DELETE` | `/chat/conversations/{id}` | ✅ | Delete a conversation |
| `GET` | `/chat/conversations/{id}/messages` | ✅ | Get messages for a conversation |

### Streaming Chat Request

```json
// POST /chat/stream
// Content-Type: application/json
{
  "messages": [
    { "role": "user", "content": "Analyze AAPL earnings" }
  ],
  "conversationId": "uuid-or-null",
  "model": "claude-sonnet-4-20250514",
  "systemPrompt": "You are a financial analyst...",
  "skills": ["skill-uuid-1"],
  "brain_enabled": true,
  "tools_enabled": true
}
```

**Response:** Server-Sent Events (SSE) stream using Vercel AI SDK Data Stream Protocol. See [05-STREAMING-PROTOCOL.md](05-STREAMING-PROTOCOL.md).

### V6 Streaming (UI Message Stream)

```json
// POST /chat/v6
// Same request body as /chat/stream
// Response: UI Message Stream format (used by Vercel AI SDK v6 useChat)
```

---

## 3. AI (`/ai`)

**Tag:** `AI`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/ai/generate` | ✅ | One-shot AI generation (non-streaming) |
| `POST` | `/ai/analyze` | ✅ | Analyze text/data with AI |
| `GET` | `/ai/models` | ✅ | List available AI models |

```json
// POST /ai/generate
{
  "prompt": "Summarize the latest AAPL 10-K filing",
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 4096,
  "system_prompt": "optional system prompt"
}
```

---

## 4. Brain / Knowledge Base (`/brain`)

**Tag:** `Brain`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/brain/documents` | ✅ | List all knowledge base documents |
| `POST` | `/brain/documents` | ✅ | Upload a document to the brain |
| `GET` | `/brain/documents/{id}` | ✅ | Get a specific document |
| `DELETE` | `/brain/documents/{id}` | ✅ | Delete a document |
| `POST` | `/brain/query` | ✅ | Semantic search the knowledge base |
| `GET` | `/brain/stats` | ✅ | Get brain statistics (doc count, size) |
| `POST` | `/brain/documents/{id}/reprocess` | ✅ | Re-chunk and re-embed a document |

### Upload Document

```
POST /brain/documents
Content-Type: multipart/form-data

file: <PDF, DOCX, TXT, CSV, MD>
title: "Document Title" (optional)
description: "Description" (optional)
```

The document is:
1. Stored in Supabase Storage (`brain-documents` bucket)
2. Text extracted (PDF/DOCX/CSV parsing)
3. Split into chunks (1000 tokens, 200 overlap)
4. Each chunk embedded via Anthropic embeddings
5. Embeddings stored in `brain_document_chunks` with pgvector

### Semantic Search

```json
// POST /brain/query
{ "query": "What is the fund's risk-adjusted return?", "top_k": 5 }

// Response:
{
  "results": [
    {
      "chunk_text": "The risk-adjusted return...",
      "similarity": 0.89,
      "document_id": "uuid",
      "document_title": "Q4 Report"
    }
  ]
}
```

---

## 5. Skills (`/skills`)

**Tag:** `Skills`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/skills` | ✅ | List all available skills |
| `POST` | `/skills` | ✅ | Create a new skill |
| `GET` | `/skills/{id}` | ✅ | Get skill details |
| `PUT` | `/skills/{id}` | ✅ | Update a skill |
| `DELETE` | `/skills/{id}` | ✅ | Delete a skill |
| `POST` | `/skills/{id}/toggle` | ✅ | Enable/disable a skill |

```json
// POST /skills
{
  "name": "SEC Filing Analyzer",
  "description": "Specialized in analyzing SEC filings",
  "system_prompt": "You are an expert SEC filing analyst...",
  "is_active": true,
  "category": "finance"
}
```

---

## 6. Researcher (`/researcher`)

**Tag:** `Researcher`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/researcher/research` | ✅ | Start a research session (streaming) |
| `GET` | `/researcher/sessions` | ✅ | List research sessions |
| `GET` | `/researcher/sessions/{id}` | ✅ | Get research session details |
| `DELETE` | `/researcher/sessions/{id}` | ✅ | Delete a research session |
| `POST` | `/researcher/sessions/{id}/export` | ✅ | Export research as DOCX |

### Start Research

```json
// POST /researcher/research
{
  "query": "Comprehensive analysis of NVDA for 2026",
  "sources": ["sec_filings", "news", "market_data", "web_search"],
  "depth": "comprehensive"
}
// Response: SSE stream with research progress + final report
```

The research engine runs a multi-step pipeline:
1. **Query decomposition** — Break into sub-queries
2. **Source gathering** — Parallel fetch from SEC, news, market data, Tavily
3. **Synthesis** — Claude synthesizes findings into a report
4. **Citation** — All sources cited

---

## 7. Edgar (SEC Filings) (`/edgar`)

**Tag:** `EDGAR`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/edgar/search` | ✅ | Search SEC filings |
| `GET` | `/edgar/filings/{cik}` | ✅ | Get filings for a company |
| `GET` | `/edgar/filing/{accession}` | ✅ | Get a specific filing |
| `GET` | `/edgar/company/{ticker}` | ✅ | Company info by ticker |

```
GET /edgar/search?query=AAPL&filing_type=10-K&limit=10
```

---

## 8. Backtest (`/backtest`)

**Tag:** `Backtest`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/backtest/run` | ✅ | Run a backtest strategy |
| `GET` | `/backtest/sessions` | ✅ | List backtest sessions |
| `GET` | `/backtest/sessions/{id}` | ✅ | Get backtest results |
| `DELETE` | `/backtest/sessions/{id}` | ✅ | Delete a backtest session |

```json
// POST /backtest/run
{
  "strategy_description": "60/40 portfolio rebalanced quarterly",
  "tickers": ["SPY", "AGG"],
  "start_date": "2020-01-01",
  "end_date": "2025-12-31",
  "initial_capital": 100000
}
```

---

## 9. Presentations (`/presentations`)

**Tag:** `Presentations`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/presentations/generate` | ✅ | Generate a PPTX presentation |
| `GET` | `/presentations` | ✅ | List generated presentations |
| `GET` | `/presentations/{id}` | ✅ | Get presentation details |
| `GET` | `/presentations/{id}/download` | ✅ | Download PPTX file |
| `DELETE` | `/presentations/{id}` | ✅ | Delete a presentation |

```json
// POST /presentations/generate
{
  "topic": "Q4 2025 Market Outlook",
  "slides_count": 12,
  "style": "bull-bear",
  "include_charts": true,
  "content_context": "Focus on technology sector..."
}
```

---

## 10. Files (`/files`)

**Tag:** `Files`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/files` | ✅ | List user's uploaded files |
| `POST` | `/files/upload` | ✅ | Upload a file to storage |
| `GET` | `/files/{id}` | ✅ | Get file metadata |
| `GET` | `/files/{id}/download` | ✅ | Download a file |
| `DELETE` | `/files/{id}` | ✅ | Delete a file |

Max file size: **50 MB**

---

## 11. Upload (`/upload`)

**Tag:** `Upload`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/upload` | ✅ | Upload file with AI analysis |
| `POST` | `/upload/parse` | ✅ | Parse document (extract text) |

Accepts: PDF, DOCX, CSV, TXT, MD, XLSX

---

## 12. Content (`/content`)

**Tag:** `Content`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/content/generate` | ✅ | Generate content (articles, reports) |
| `POST` | `/content/summarize` | ✅ | Summarize a document |
| `POST` | `/content/export/docx` | ✅ | Export content as DOCX |

---

## 13. Train (`/train`)

**Tag:** `Training`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/train/examples` | ✅ | List training examples |
| `POST` | `/train/examples` | ✅ | Add a training example |
| `PUT` | `/train/examples/{id}` | ✅ | Update a training example |
| `DELETE` | `/train/examples/{id}` | ✅ | Delete a training example |
| `POST` | `/train/import` | ✅ | Bulk import training examples |
| `GET` | `/train/stats` | ✅ | Training statistics |

```json
// POST /train/examples
{
  "input": "What is a P/E ratio?",
  "output": "The Price-to-Earnings ratio measures...",
  "category": "fundamentals",
  "tags": ["finance", "valuation"]
}
```

---

## 14. Admin (`/admin`)

**Tag:** `Admin`

All endpoints require `is_admin: true`.

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/admin/stats` | ✅🔒 | System statistics |
| `GET` | `/admin/users` | ✅🔒 | List all users |
| `GET` | `/admin/conversations` | ✅🔒 | All conversations |
| `GET` | `/admin/usage` | ✅🔒 | API usage metrics |
| `POST` | `/admin/broadcast` | ✅🔒 | Broadcast notification |

---

## 15. AFL Generator (`/afl`)

**Tag:** `AFL`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/afl/generate` | ✅ | Generate an AFL letter |
| `POST` | `/afl/generate/stream` | ✅ | Generate AFL (streaming) |
| `GET` | `/afl/history` | ✅ | List generated AFLs |
| `GET` | `/afl/history/{id}` | ✅ | Get a specific AFL |
| `DELETE` | `/afl/history/{id}` | ✅ | Delete an AFL |
| `GET` | `/afl/settings` | ✅ | Get AFL settings/presets |
| `PUT` | `/afl/settings` | ✅ | Update AFL settings |
| `POST` | `/afl/validate` | ✅ | Validate AFL content |
| `POST` | `/afl/upload-reference` | ✅ | Upload reference document |

---

## 16. Health (`/health`)

**Tag:** `Health`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/health` | ❌ | Health check |
| `GET` | `/health/detailed` | ❌ | Detailed health (DB, AI, storage) |

```json
// GET /health
{
  "status": "healthy",
  "version": "1.3.7",
  "timestamp": "2026-03-06T16:00:00Z"
}
```

---

## 17. Yahoo Finance (`/yfinance`)

**Tag:** `Yahoo Finance`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/yfinance/quote/{ticker}` | ✅ | Get stock quote |
| `GET` | `/yfinance/history/{ticker}` | ✅ | Get price history |
| `GET` | `/yfinance/info/{ticker}` | ✅ | Get company info |
| `GET` | `/yfinance/financials/{ticker}` | ✅ | Get financial statements |

```
GET /yfinance/quote/AAPL
GET /yfinance/history/AAPL?period=1y&interval=1d
```

---

## 18. Reverse Engineer (`/reverse-engineer`)

**Tag:** `Reverse Engineer`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/reverse-engineer/analyze` | ✅ | Reverse engineer a strategy from a document |
| `POST` | `/reverse-engineer/analyze/stream` | ✅ | Streaming version |

```json
// POST /reverse-engineer/analyze
{
  "document_text": "The fund maintains a 60/40...",
  "analysis_type": "strategy"
}
```

---

## 19. PPTX Engine

**Prefix:** `/pptx` (routes) and `/pptx/generate` (generate)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/pptx/generate` | ✅ | Generate presentation from template |
| `GET` | `/pptx/templates` | ✅ | List available templates |
| `POST` | `/pptx/upload-template` | ✅ | Upload a custom template |

---

## Error Responses

All endpoints return errors in this format:

```json
{
  "detail": "Human-readable error message"
}
```

| Status Code | Meaning |
|-------------|---------|
| `400` | Bad request / validation error |
| `401` | Missing or invalid JWT |
| `403` | Insufficient permissions (not admin) |
| `404` | Resource not found |
| `429` | Rate limit exceeded |
| `500` | Internal server error |

---

## Rate Limiting

- **120 requests per minute** per IP address
- In-memory counter (resets on server restart)
- Skipped for `/health` and `/docs` endpoints
- Returns `429 Too Many Requests` when exceeded

---

*Next: [03 — Authentication](03-AUTHENTICATION.md)*
