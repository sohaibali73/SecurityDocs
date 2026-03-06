# 10 — Deployment

> ByPotomac SDK · Infrastructure & Deployment Guide

---

## Architecture Map

```
┌───────────────────────────────────────────────────────────┐
│                    DEPLOYMENT TOPOLOGY                     │
│                                                           │
│  ┌─────────────────┐     ┌────────────────────────────┐  │
│  │    Vercel        │     │       Railway              │  │
│  │                  │     │                            │  │
│  │  Web Frontend    │────▶│  Python Backend            │  │
│  │  (Next.js 16)   │     │  (FastAPI + Uvicorn)       │  │
│  │                  │     │                            │  │
│  │  analystby       │     │  potomac-analyst-workbench │  │
│  │  potomac.        │     │  -production.up.           │  │
│  │  vercel.app      │     │  railway.app               │  │
│  └─────────────────┘     └─────────────┬──────────────┘  │
│                                         │                 │
│  ┌─────────────────┐     ┌─────────────▼──────────────┐  │
│  │  Windows App     │     │       Supabase             │  │
│  │  (MSIX)         │────▶│                            │  │
│  │                  │     │  PostgreSQL + Auth          │  │
│  │  Local Install   │     │  Storage + pgvector         │  │
│  └─────────────────┘     └────────────────────────────┘  │
│                                                           │
│  ┌────────────────────────────────────────────────────┐  │
│  │              External APIs                          │  │
│  │  Anthropic · Tavily · SEC EDGAR · Yahoo Finance    │  │
│  │  Finnhub · FRED · NewsAPI                          │  │
│  └────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
```

---

## Backend Deployment (Railway)

### Platform

- **Host:** [Railway](https://railway.app)
- **URL:** `https://potomac-analyst-workbench-production.up.railway.app`
- **Port:** `8070`
- **Runtime:** Python 3.11+ via Nixpacks

### Configuration Files

#### `railway.json`

```json
{
  "$schema": "https://railway.app/railway.schema.json",
  "build": {
    "builder": "NIXPACKS"
  },
  "deploy": {
    "startCommand": "uvicorn main:app --host 0.0.0.0 --port ${PORT:-8070}",
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 10
  }
}
```

#### `nixpacks.toml`

```toml
[phases.setup]
nixPkgs = ["python311"]

[phases.install]
cmds = ["pip install -r requirements.txt"]

[start]
cmd = "uvicorn main:app --host 0.0.0.0 --port ${PORT:-8070} --keep-alive-timeout 120"
```

#### `Procfile`

```
web: uvicorn main:app --host 0.0.0.0 --port ${PORT:-8070} --keep-alive-timeout 120
```

#### `Dockerfile` (alternative)

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8070
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8070", "--keep-alive-timeout", "120"]
```

### Railway Environment Variables

All environment variables are set in the Railway dashboard:

| Variable | Required | Description |
|----------|----------|-------------|
| `PORT` | Auto | Set by Railway (default 8070) |
| `SUPABASE_URL` | ✅ | Supabase project URL |
| `SUPABASE_KEY` | ✅ | Supabase anon key |
| `SUPABASE_SERVICE_KEY` | ✅ | Supabase service_role key |
| `ANTHROPIC_API_KEY` | ✅ | Anthropic Claude API key |
| `ENCRYPTION_KEY` | ✅ | Fernet encryption key for API key storage |
| `TAVILY_API_KEY` | ✅ | Tavily web search API key |
| `ADMIN_EMAILS` | ❌ | Comma-separated admin emails |
| `FRONTEND_URL` | ❌ | Additional CORS origin |
| `FINNHUB_API_KEY` | ❌ | Finnhub market data |
| `FRED_API_KEY` | ❌ | FRED economic data |
| `NEWSAPI_KEY` | ❌ | News API |
| `SMTP_HOST` | ❌ | Email server for password resets |
| `SMTP_PORT` | ❌ | SMTP port |
| `SMTP_USER` | ❌ | SMTP username |
| `SMTP_PASSWORD` | ❌ | SMTP password |
| `DEFAULT_AI_MODEL` | ❌ | Default Claude model (claude-sonnet-4-20250514) |
| `ENVIRONMENT` | ❌ | `production` or `development` |

### Deploying Backend Updates

```bash
# From the backend directory
cd C:\Users\SohaibAli\PycharmProjects\Potomac-Analyst-Workbench

# Push to Railway (auto-deploys on push)
git add .
git commit -m "Update backend"
git push origin main
```

Railway auto-deploys when the linked Git branch is updated.

### Generating a Fernet Encryption Key

```python
from cryptography.fernet import Fernet
key = Fernet.generate_key()
print(key.decode())
# Set this as ENCRYPTION_KEY in Railway
```

---

## Web Frontend Deployment (Vercel)

### Platform

- **Host:** [Vercel](https://vercel.com)
- **URL:** `https://analystbypotomac.vercel.app`
- **Framework:** Next.js 16 (auto-detected)

### Vercel Environment Variables

| Variable | Value |
|----------|-------|
| `NEXT_PUBLIC_API_URL` | `https://potomac-analyst-workbench-production.up.railway.app` |

### Deploying Frontend Updates

```bash
# From the frontend directory
cd C:\Users\SohaibAli\Documents\Abpfrontend

# Push to Vercel (auto-deploys)
git add .
git commit -m "Update frontend"
git push origin main
```

Vercel auto-deploys from the linked Git repo.

### Alternative: Deploy on Railway

The frontend can also run on Railway with these configs:

```json
// railway.json
{
  "build": { "builder": "NIXPACKS" },
  "deploy": {
    "startCommand": "npm run build && npm start"
  }
}
```

```toml
# nixpacks.toml
[phases.setup]
nixPkgs = ["nodejs_20"]

[phases.install]
cmds = ["npm ci"]

[phases.build]
cmds = ["npm run build"]

[start]
cmd = "npm start"
```

---

## Native Windows App Distribution

### Build

```powershell
cd C:\Users\SohaibAli\Documents\PotomacAnalyst

# Build release
dotnet build -c Release

# Create MSIX package
dotnet publish -c Release -r win-x64 --self-contained
```

### Distribution Options

1. **Side-loading** — Share the MSIX package directly
2. **Microsoft Store** — Submit through Partner Center
3. **Enterprise** — Deploy via Intune/SCCM

### Backend URL

The Windows app has the backend URL hardcoded:

```csharp
// Services/ApiService.cs
private const string BASE_URL = 
    "https://potomac-analyst-workbench-production.up.railway.app";
```

To change, update this constant and rebuild.

---

## Database (Supabase)

### Setup

1. Create a Supabase project at [supabase.com](https://supabase.com)
2. Enable the `pgvector` extension:
   ```sql
   CREATE EXTENSION IF NOT EXISTS vector;
   ```
3. Run migrations in order (001 → 014):
   ```bash
   # Each migration file in db/migrations/
   psql $DATABASE_URL < db/migrations/001_initial_schema.sql
   psql $DATABASE_URL < db/migrations/002_feedback_analytics.sql
   # ... etc
   ```

### Required Keys

From the Supabase dashboard → Settings → API:

| Key | Where It Goes |
|-----|---------------|
| **URL** | `SUPABASE_URL` |
| **anon (public)** | `SUPABASE_KEY` |
| **service_role** | `SUPABASE_SERVICE_KEY` |

### Storage Buckets

Create these buckets in Supabase Storage:

```sql
-- In Supabase SQL editor
INSERT INTO storage.buckets (id, name, public) VALUES
  ('brain-documents', 'brain-documents', false),
  ('uploaded-files', 'uploaded-files', false),
  ('afl-files', 'afl-files', false),
  ('presentations', 'presentations', false),
  ('avatars', 'avatars', true);
```

---

## Health Monitoring

### Health Check Endpoint

```
GET https://potomac-analyst-workbench-production.up.railway.app/health
```

Response:
```json
{
  "status": "healthy",
  "version": "1.3.7",
  "timestamp": "2026-03-06T12:00:00Z"
}
```

### Detailed Health Check

```
GET https://potomac-analyst-workbench-production.up.railway.app/health/detailed
```

Checks: Database connection, Anthropic API, Storage, etc.

### Railway Monitoring

Railway provides:
- **Deploy logs** — Build and runtime logs
- **Metrics** — CPU, memory, network
- **Alerts** — Configurable notifications
- **Restart policies** — Auto-restart on failure (max 10 retries)

---

## CI/CD Pipeline

### Current Flow

```
Developer pushes to GitHub
      │
      ├──── Railway detects push → auto-build → auto-deploy (backend)
      │
      └──── Vercel detects push → auto-build → auto-deploy (frontend)
```

### Recommended Pre-Push Checks

```bash
# Backend
cd Potomac-Analyst-Workbench
python -m pytest tests/ -v          # Run tests
python -m mypy core/ api/           # Type checking
python -m ruff check .              # Linting

# Frontend
cd Abpfrontend
npm run lint                        # ESLint
npm run build                       # Build check
```

---

## SSL/TLS

- **Railway:** Automatic SSL via Let's Encrypt
- **Vercel:** Automatic SSL via Let's Encrypt
- **Supabase:** SSL enabled by default
- **Custom domains:** Both Railway and Vercel support custom domains

---

## Scaling

### Backend (Railway)

- Vertical scaling: Increase RAM/CPU in Railway settings
- Railway supports replicas for horizontal scaling (paid plan)
- Current: Single instance, sufficient for moderate load

### Frontend (Vercel)

- Serverless by default, scales automatically
- Edge functions for API routes
- Global CDN for static assets

### Database (Supabase)

- Scale via Supabase dashboard (Pro plan → larger compute)
- Connection pooling via PgBouncer (built-in)
- Read replicas available on Enterprise plans

---

*Next: [11 — Quick Start](11-QUICK-START.md)*
