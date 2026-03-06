# 11 — Quick Start

> ByPotomac SDK · Getting Started for Developers

---

## Prerequisites

| Tool | Required For | Install |
|------|-------------|---------|
| **Python 3.11+** | Backend | [python.org](https://python.org) |
| **Node.js 20+** | Web Frontend | [nodejs.org](https://nodejs.org) |
| **npm** | Web Frontend | Comes with Node.js |
| **.NET 8 SDK** | Windows App | [dotnet.microsoft.com](https://dotnet.microsoft.com) |
| **Git** | Everything | [git-scm.com](https://git-scm.com) |

---

## 1. Clone the Repositories

```bash
# Backend
git clone https://github.com/sohaibali73/Potomac-Analyst-Workbench.git
cd Potomac-Analyst-Workbench

# Web Frontend (separate repo)
git clone <frontend-repo-url> Abpfrontend

# Windows App (separate repo)
git clone <windows-repo-url> PotomacAnalyst
```

---

## 2. Set Up the Backend

### Install Dependencies

```bash
cd Potomac-Analyst-Workbench
pip install -r requirements.txt
```

### Create `.env` File

```env
# Required
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=eyJ...your-anon-key
SUPABASE_SERVICE_KEY=eyJ...your-service-role-key
ANTHROPIC_API_KEY=sk-ant-...your-anthropic-key
ENCRYPTION_KEY=your-fernet-key-here
TAVILY_API_KEY=tvly-...your-tavily-key

# Optional
ADMIN_EMAILS=admin@yourcompany.com
FRONTEND_URL=http://localhost:3000
DEFAULT_AI_MODEL=claude-sonnet-4-20250514
ENVIRONMENT=development

# Optional external APIs
FINNHUB_API_KEY=
FRED_API_KEY=
NEWSAPI_KEY=
```

### Generate an Encryption Key

```python
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

### Set Up the Database

1. Go to [supabase.com](https://supabase.com) → Create a new project
2. Enable pgvector:
   ```sql
   CREATE EXTENSION IF NOT EXISTS vector;
   ```
3. Run migrations in order:
   ```bash
   # Using the Supabase SQL editor, paste and run each file:
   db/migrations/001_initial_schema.sql
   db/migrations/001_training_data.sql
   db/migrations/002_feedback_analytics.sql
   db/migrations/003_researcher_tables.sql
   db/migrations/004_history_tables.sql
   db/migrations/005_afl_uploaded_files.sql
   db/migrations/006_afl_settings_presets.sql
   db/migrations/007_conversation_files.sql
   db/migrations/008_missing_tables.sql
   db/migrations/009_brain_tables_and_embeddings.sql
   db/migrations/010_supabase_auth_migration.sql
   db/migrations/011_fix_foreign_keys.sql
   db/migrations/012_clean_slate_auth_fix.sql
   db/migrations/013_security_hardening.sql
   db/migrations/014_secure_rebuild.sql
   ```

### Run the Backend

```bash
python main.py
# or
uvicorn main:app --host 0.0.0.0 --port 8070 --reload
```

**Backend is now running at:** `http://localhost:8070`  
**Swagger docs at:** `http://localhost:8070/docs`

### Verify It Works

```bash
curl http://localhost:8070/health
# {"status": "healthy", "version": "1.3.7", ...}
```

---

## 3. Set Up the Web Frontend

### Install Dependencies

```bash
cd Abpfrontend
npm install
```

### Create `.env.local`

```env
NEXT_PUBLIC_API_URL=http://localhost:8070
```

### Run the Frontend

```bash
npm run dev
```

**Frontend is now running at:** `http://localhost:3000`

### Test the Connection

1. Open `http://localhost:3000` in your browser
2. Click "Register" → Create an account
3. You should be redirected to the dashboard
4. Open the Chat page → Send a message → You should see a streamed AI response

---

## 4. Set Up the Windows App (Optional)

### Open in Visual Studio

```bash
cd PotomacAnalyst
start PotomacAnalyst.sln    # Opens in Visual Studio 2022
```

### Point to Local Backend

Edit `Services/ApiService.cs`:

```csharp
// Change from production to local:
private const string BASE_URL = "http://localhost:8070";
```

### Build and Run

```
F5 in Visual Studio
# or
dotnet run
```

---

## 5. Test Key Features

### Register & Login

```bash
# Register
curl -X POST http://localhost:8070/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"Test123!","name":"Test User"}'

# Login
curl -X POST http://localhost:8070/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"Test123!"}'
# → Returns { "access_token": "eyJ..." }
```

### Chat with AI

```bash
TOKEN="eyJ...your-token"

# Streaming chat (SSE)
curl -N -X POST http://localhost:8070/chat/stream \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "messages": [{"role":"user","content":"What is the P/E ratio of AAPL?"}],
    "tools_enabled": true,
    "brain_enabled": false
  }'
# → Streams Vercel AI SDK Data Stream Protocol lines
```

### Upload to Knowledge Base

```bash
curl -X POST http://localhost:8070/brain/documents \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@quarterly_report.pdf" \
  -F "title=Q4 2025 Report"
# → Document is parsed, chunked, and embedded
```

### Search Knowledge Base

```bash
curl -X POST http://localhost:8070/brain/query \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"query":"What was the risk-adjusted return?","top_k":5}'
```

### Create a Skill

```bash
curl -X POST http://localhost:8070/skills \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "SEC Expert",
    "description": "Specializes in SEC filing analysis",
    "system_prompt": "You are an expert in SEC filings...",
    "category": "finance"
  }'
```

### Run Research

```bash
curl -N -X POST http://localhost:8070/researcher/research \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "query": "Comprehensive analysis of NVDA",
    "sources": ["sec_filings","news","market_data","web_search"],
    "depth": "comprehensive"
  }'
# → Streams research progress and final report
```

---

## 6. Project Structure Quick Reference

```
Potomac-Analyst-Workbench/        # Backend (Python)
├── main.py                       # FastAPI entry point
├── config.py                     # Environment config
├── api/routes/                   # 18 route modules
├── core/                         # AI engine, tools, skills
├── db/                           # Supabase client + migrations
└── pptx_engine/                  # Presentation generator

Abpfrontend/                      # Web Frontend (Next.js)
├── src/app/                      # App Router pages
├── src/app/api/                  # API proxy routes
├── src/lib/api.ts                # Backend API client
├── src/contexts/AuthContext.tsx   # Auth state
└── src/components/               # React components

PotomacAnalyst/                   # Windows App (C# / WinUI 3)
├── App.xaml.cs                   # DI container entry
├── Services/                     # API, Auth, Session services
├── Pages/                        # 16 XAML pages
└── Models/                       # DTOs
```

---

## 7. Common Issues

### CORS Errors in Browser

The backend only allows specific origins. For local dev, these are whitelisted:
- `http://localhost:3000`
- `http://localhost:3001`
- `http://127.0.0.1:3000`

If you're using a different port, add it to the `FRONTEND_URL` env var.

### "Unauthorized" on API Calls

- Ensure you're passing `Authorization: Bearer <token>`
- Tokens expire after 1 hour — use `POST /auth/refresh-token`
- Check that Supabase Auth is properly configured

### Missing Database Tables

Run all 14 migrations in order. Some have dependencies on earlier ones.

### AI Responses Not Working

- Verify `ANTHROPIC_API_KEY` is set and valid
- Check the `/health/detailed` endpoint for service status
- Look at Railway logs for error details

### Streaming Not Working

- Ensure the client supports SSE (Server-Sent Events)
- For the web frontend, verify the proxy route at `/api/chat/route.ts`
- For the Windows app, ensure `HttpCompletionOption.ResponseHeadersRead` is used

---

## 8. Documentation Map

| Need to understand... | Read... |
|----------------------|---------|
| How everything fits together | [01-ARCHITECTURE.md](01-ARCHITECTURE.md) |
| All API endpoints | [02-BACKEND-API.md](02-BACKEND-API.md) |
| Login/auth flow | [03-AUTHENTICATION.md](03-AUTHENTICATION.md) |
| How Claude AI works | [04-AI-ENGINE.md](04-AI-ENGINE.md) |
| Streaming format | [05-STREAMING-PROTOCOL.md](05-STREAMING-PROTOCOL.md) |
| Web app code | [06-WEB-FRONTEND.md](06-WEB-FRONTEND.md) |
| Windows app code | [07-NATIVE-WINDOWS.md](07-NATIVE-WINDOWS.md) |
| Database tables | [08-DATABASE.md](08-DATABASE.md) |
| AI tools & skills | [09-TOOLS-AND-SKILLS.md](09-TOOLS-AND-SKILLS.md) |
| Deploy to production | [10-DEPLOYMENT.md](10-DEPLOYMENT.md) |

---

*ByPotomac SDK — Built by Potomac Fund Management*
