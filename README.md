# ByPotomac SDK

> **The complete platform documentation for Analyst by Potomac — an AI-powered financial analyst workbench.**

---

## What is ByPotomac?

ByPotomac is a full-stack AI Integrated platform built for financial analysts. It connects a **Python backend** (FastAPI), a **Next.js web frontend**, and a **native Windows app** (WinUI 3 / C#) — all powered by Anthropic Claude and backed by Supabase.

The system enables:

- **AI Chat** — Conversational financial analysis with tool calling (SEC filings, market data, news)
- **Research Engine** — Multi-source research with auto-generated reports
- **Knowledge Brain** — Upload documents, build a RAG-powered knowledge base
- **AFL Generator** — Generate American Funds Letters with compliance controls
- **Backtesting** — Strategy backtesting with AI-generated analysis
- **Presentations** — Auto-generate branded PowerPoint decks
- **Skills System** — Extensible AI skill modules for specialized tasks
- **Training** — Fine-tune the AI with custom examples

---

## System Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CLIENTS                                    │
│                                                                     │
│   ┌──────────────┐    ┌───────────────────┐    ┌────────────────┐  │
│   │  Web App      │    │  Windows Native    │    │  Any HTTP      │  │
│   │  Next.js 16   │    │  WinUI 3 / C#     │    │  Client        │  │
│   │  Vercel AI v6 │    │  .NET 8            │    │  (curl, etc.)  │  │
│   └──────┬───────┘    └────────┬───────────┘    └───────┬────────┘  │
│          │                     │                        │           │
└──────────┼─────────────────────┼────────────────────────┼───────────┘
           │                     │                        │
           ▼                     ▼                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     BACKEND  (Python / FastAPI)                      │
│                                                                     │
│   URL: https://potomac-analyst-workbench-production.up.railway.app  │
│   Host: Railway          Port: 8070                                 │
│                                                                     │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│   │ Auth     │  │ Chat/AI  │  │ Research │  │ Brain/Knowledge  │  │
│   │ Routes   │  │ Stream   │  │ Engine   │  │ Base (RAG)       │  │
│   ├──────────┤  ├──────────┤  ├──────────┤  ├──────────────────┤  │
│   │ Admin    │  │ Edgar    │  │ Backtest │  │ Skills           │  │
│   │ Routes   │  │ Client   │  │ Engine   │  │ System           │  │
│   ├──────────┤  ├──────────┤  ├──────────┤  ├──────────────────┤  │
│   │ Upload   │  │ yFinance │  │ AFL Gen  │  │ Presentations    │  │
│   │ Files    │  │ Market   │  │          │  │ PPTX Engine      │  │
│   └──────────┘  └──────────┘  └──────────┘  └──────────────────┘  │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │              Claude AI Engine (Anthropic API)                │  │
│   │  • Tool Calling  • Streaming  • Context Management          │  │
│   │  • Vercel AI SDK Data Stream Protocol                       │  │
│   └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        SUPABASE (Database)                          │
│                                                                     │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│   │ Auth     │  │ Postgres │  │ Storage  │  │ pgvector         │  │
│   │ (JWT)    │  │ Tables   │  │ Buckets  │  │ Embeddings       │  │
│   └──────────┘  └──────────┘  └──────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Documentation Index

| # | Document | Description |
|---|----------|-------------|
| 01 | [Architecture Overview](docs/01-ARCHITECTURE.md) | System architecture, component map, data flow |
| 02 | [Backend API Reference](docs/02-BACKEND-API.md) | All 20 route modules, every endpoint |
| 03 | [Authentication](docs/03-AUTHENTICATION.md) | Supabase Auth, JWT flow, API keys, encryption |
| 04 | [AI Engine](docs/04-AI-ENGINE.md) | Claude integration, tool calling, context management |
| 05 | [Streaming Protocol](docs/05-STREAMING-PROTOCOL.md) | Vercel AI SDK Data Stream Protocol specification |
| 06 | [Web Frontend](docs/06-WEB-FRONTEND.md) | Next.js app architecture, pages, API client |
| 07 | [Native Windows App](docs/07-NATIVE-WINDOWS.md) | WinUI 3 / C# app architecture, services |
| 08 | [Database Schema](docs/08-DATABASE.md) | All tables, migrations, RLS policies, storage |
| 09 | [Tools & Skills](docs/09-TOOLS-AND-SKILLS.md) | AI tool definitions, skill system |
| 10 | [Deployment](docs/10-DEPLOYMENT.md) | Railway, Vercel, Docker, environment variables |
| 11 | [Quick Start](docs/11-QUICK-START.md) | Getting started for developers |

---

## Live URLs

| Component | URL |
|-----------|-----|
| **Backend API** | https://potomac-analyst-workbench-production.up.railway.app |
| **API Docs (Swagger)** | https://potomac-analyst-workbench-production.up.railway.app/docs |
| **Web App** | https://analystbypotomac.vercel.app |
| **Health Check** | https://potomac-analyst-workbench-production.up.railway.app/health |

---

## Tech Stack Summary

| Layer | Technology | Version |
|-------|-----------|---------|
| **Backend** | Python + FastAPI | 3.11+ / FastAPI 0.115 |
| **AI Model** | Anthropic Claude | claude-sonnet-4-20250514 |
| **Web Frontend** | Next.js + React + Vercel AI SDK | 16 / 19 / v6 |
| **Native App** | WinUI 3 + .NET | Windows App SDK 1.8 / .NET 8 |
| **Database** | Supabase (PostgreSQL) | pgvector enabled |
| **Hosting** | Railway (backend) + Vercel (web) | — |
| **Auth** | Supabase Auth (JWT) | — |

---

*ByPotomac SDK — Built by Potomac Fund Management (DEVELOPED BY SOHAIB ALI)*
