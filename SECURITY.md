# 🔒 Analyst by Potomac — Security Architecture

**Last Updated:** February 23, 2026  
**Version:** 1.3.0 — Migration 014 Secure Rebuild

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Authentication](#authentication)
3. [API Key Encryption](#api-key-encryption)
4. [Database Security (RLS)](#database-security-rls)
5. [CORS & Network Security](#cors--network-security)
6. [Knowledge Base & Documents](#knowledge-base--documents)
7. [Deployment Security](#deployment-security)
8. [Environment Variables](#environment-variables)
9. [Services Used](#services-used)
10. [Security Checklist](#security-checklist)

---

## Architecture Overview

```
┌─────────────────────┐     HTTPS      ┌──────────────────────┐
│   Frontend          │ ◄─────────────► │   Backend API        │
│   Next.js 16        │   JWT Bearer    │   FastAPI (Python)   │
│   Vercel            │                 │   Railway            │
└─────────────────────┘                 └──────────┬───────────┘
                                                   │ service_role key
                                                   ▼
                                        ┌──────────────────────┐
                                        │   Supabase           │
                                        │   PostgreSQL + Auth   │
                                        │   RLS Enabled         │
                                        └──────────────────────┘
```

**Data flow:** Frontend → Backend API → Supabase. The frontend NEVER talks directly to the database. All requests go through the FastAPI backend which validates JWT tokens and enforces authorization.

---

## Authentication

### How Login Works
1. User enters email + password on the frontend
2. Frontend sends `POST /auth/login` to the backend
3. Backend calls `supabase.auth.sign_in_with_password()` to validate credentials against Supabase Auth (`auth.users` table)
4. Supabase returns a JWT token (signed with the project's JWT secret)
5. Backend returns the JWT token to the frontend
6. Frontend stores the token in `localStorage` and sends it as `Authorization: Bearer <token>` on all subsequent requests

### How Token Validation Works
1. Every protected endpoint uses the `get_current_user` dependency
2. This extracts the Bearer token from the `Authorization` header
3. Calls `supabase.auth.get_user(token)` to validate the JWT
4. Fetches the user's profile from `user_profiles` table
5. Checks `is_active` — deactivated users are blocked with HTTP 403
6. Returns user data (without API keys) to the endpoint

### Password Security
- Passwords are managed by **Supabase Auth** (bcrypt hashing, salted)
- Minimum 8 characters required
- Password reset via email (Supabase built-in)
- The backend NEVER stores or sees raw passwords

### Admin Access
- Admin status stored in `user_profiles.is_admin` boolean
- Admin emails configured via `ADMIN_EMAILS` environment variable
- Auto-granted on registration if email matches admin list
- Admin-only endpoints use `verify_admin` dependency

---

## API Key Encryption

### How User API Keys Are Stored
User-provided API keys (Claude, Tavily) are **encrypted at rest** using AES-256 before being stored in the database.

**Encryption Module:** `core/encryption.py`

```
User enters API key → encrypt_value(key) → "enc:gAAAAABm..." → stored in DB
                                                                      │
User makes request → read from DB → decrypt_value("enc:gAAAAABm...") → plain key → API call
```

### Technical Details
- **Algorithm:** Fernet (AES-128-CBC + HMAC-SHA256) from the `cryptography` library
- **Key Derivation:** `ENCRYPTION_KEY` env var → SHA-256 hash → 32-byte Fernet key
- **Format:** Encrypted values are prefixed with `enc:` to distinguish from legacy plain text
- **Backward Compatibility:** `decrypt_value()` handles both encrypted (`enc:...`) and legacy plain text values

### Key Management
- The `ENCRYPTION_KEY` is stored as an environment variable (never in code)
- Set in Railway env vars for production
- If the key is lost/changed, existing encrypted data cannot be recovered
- Generate a new key: `python -c "import secrets; print(secrets.token_urlsafe(32))"`

### What's Encrypted
| Data | Encrypted? | Location |
|------|-----------|----------|
| User Claude API keys | ✅ AES-256 | `user_profiles.claude_api_key` |
| User Tavily API keys | ✅ AES-256 | `user_profiles.tavily_api_key` |
| Server Anthropic key | ✅ Env var only | `ANTHROPIC_API_KEY` env var |
| Passwords | ✅ bcrypt (Supabase) | `auth.users` |
| Chat messages | ❌ Plain text | `messages` table |
| AFL code | ❌ Plain text | `afl_codes` table |
| Documents | ❌ Plain text | `brain_documents` table |

---

## Database Security (RLS)

### Row Level Security
Supabase RLS (Row Level Security) ensures that even if someone obtains the database URL + anon key, they can only access their own data.

**Policy Design (Migration 014):**
- `service_role` key → **automatically bypasses all RLS** (used by the backend — no policies needed)
- `authenticated` users → can only access their own rows (via `auth.uid() = user_id` policies with `TO authenticated` clause)
- `anon` role → **❌ DENIED all access** to user data tables (anon key is public in Supabase)

> ⚠️ **Critical:** Never create `USING (true)` catch-all policies — they open access to ALL roles including `anon`. Migration 014 fixes this from earlier migrations.

### Table Policies

| Table | RLS | Policy |
|-------|-----|--------|
| `user_profiles` | ✅ Enabled | Users read/update own profile. Service role: full access |
| `conversations` | ✅ Enabled | Users manage own conversations |
| `messages` | ✅ Enabled | Users manage messages in own conversations |
| `afl_codes` | ✅ Enabled | Users manage own AFL codes |
| `afl_history` | ✅ Enabled | Users manage own history |
| `user_feedback` | ✅ Enabled | Users manage own feedback |
| `brain_documents` | ✅ Enabled | Authenticated users can read. Service role can write |
| `training_data` | ✅ Enabled | Authenticated users can read. Service role can write |

### How the Backend Bypasses RLS
The backend uses `SUPABASE_SERVICE_KEY` (service_role) which automatically bypasses all RLS policies. This is the Supabase-recommended pattern for server-side applications.

---

## CORS & Network Security

### CORS Policy
The API only accepts requests from whitelisted origins:

```python
ALLOWED_ORIGINS = [
    "https://analystbypotomac.vercel.app",
    "http://localhost:3000",  # Local development
    "http://localhost:3001",
]
```

Any request from an unlisted origin is rejected by the browser's CORS enforcement.

### Rate Limiting
- **120 requests/minute per IP** address
- Health check and docs endpoints are exempt
- Returns HTTP 429 with `Retry-After: 60` header when exceeded

### Error Handling
- Production: Internal errors return generic "Internal server error" (no stack traces)
- Development: Full error details returned for debugging
- All errors logged server-side for monitoring

### Debug Endpoints
Removed from production:
- ~~`/routes`~~ — Removed (listed all API routes)
- ~~`/debug/routers`~~ — Removed (showed internal architecture)
- ~~`/debug/missing-modules`~~ — Removed (exposed file system)

System diagnostics available at `/admin/health/system` (requires admin auth).

---

## Knowledge Base & Documents

### Current State
The knowledge base (Brain system) is **real and functional**:
- Document upload with PDF, DOCX, TXT support
- SHA-256 content hashing for deduplication
- AI-powered document classification via Claude
- Text chunking (500-character chunks)
- Text-based search (`ilike` matching)

### What's Missing (Future Enhancement)
- **Vector embeddings** — The `brain_chunks` table has an `embedding` column but it's not populated. Semantic search requires:
  1. Enable `pgvector` extension in Supabase
  2. Add an embedding API (OpenAI `text-embedding-3-small` recommended, ~$0.02/1M tokens)
  3. Generate embeddings on document upload
  4. Use cosine similarity for semantic search

### Document Storage Security
- Documents stored as text in Supabase PostgreSQL
- Protected by RLS policies (only authenticated users can read)
- Backend validates user authentication before any document access
- File size limit: 10MB per document, 50 chunks per document

---

## Deployment Security

### Backend (Railway)
- **Platform:** Railway (auto-deploys from GitHub master branch)
- **URL:** `https://potomac-analyst-workbench-production.up.railway.app`
- **Runtime:** Python with Gunicorn + Uvicorn workers
- **Environment:** All secrets stored as Railway environment variables
- **HTTPS:** Enforced by Railway's load balancer

### Frontend (Vercel)
- **Platform:** Vercel (auto-deploys from GitHub)
- **URL:** `https://analystbypotomac.vercel.app`
- **Framework:** Next.js 16 with Turbopack
- **Environment:** `NEXT_PUBLIC_API_URL` and `NEXT_PUBLIC_SUPABASE_*` vars

### Docker Security
- Non-root user recommended (current Dockerfile runs as root — future improvement)
- No secrets baked into the Docker image
- All sensitive values injected via environment variables at runtime

---

## Environment Variables

### Required for Production

| Variable | Purpose | Where to Set |
|----------|---------|-------------|
| `SUPABASE_URL` | Supabase project URL | Railway env vars |
| `SUPABASE_KEY` | Supabase anon/public key | Railway env vars |
| `SUPABASE_SERVICE_KEY` | Supabase service_role key (bypasses RLS) | Railway env vars |
| `ENCRYPTION_KEY` | AES-256 key for API key encryption | Railway env vars |
| `ANTHROPIC_API_KEY` | Server-side Claude API key (fallback) | Railway env vars |
| `FRONTEND_URL` | Frontend URL for CORS and password reset | Railway env vars |
| `ADMIN_EMAILS` | Comma-separated admin email list | Railway env vars |

### How to Get SUPABASE_SERVICE_KEY
1. Go to https://supabase.com/dashboard/project/vekcfcmstpnxubxsaano/settings/api
2. Copy the `service_role` key (NOT the anon key)
3. Set it as `SUPABASE_SERVICE_KEY` in Railway env vars

### How to Generate ENCRYPTION_KEY
```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

---

## Services Used

| Service | Purpose | Cost | Security Notes |
|---------|---------|------|---------------|
| **Supabase** | Database + Auth | Free tier | Manages auth.users, JWT signing, RLS |
| **Railway** | Backend hosting | ~$5/mo | Env vars for secrets, auto HTTPS |
| **Vercel** | Frontend hosting | Free tier | Auto HTTPS, edge deployment |
| **Anthropic Claude** | AI engine | Pay-per-use | API key encrypted at rest |
| **Tavily** | Web search for research | Free tier available | API key encrypted at rest |

**No additional services needed** for the current architecture. Vector search can be added using Supabase's built-in `pgvector` extension (no extra cost on paid plan).

---

## Security Checklist

### ✅ Implemented
- [x] **Authentication** via Supabase Auth (JWT tokens, bcrypt passwords)
- [x] **API key encryption** at rest (AES-256 via Fernet)
- [x] **CORS lockdown** to specific allowed origins
- [x] **Rate limiting** (120 req/min per IP)
- [x] **Debug endpoints removed** from production
- [x] **Error messages sanitized** in production (no stack traces)
- [x] **RLS policies** protecting user-specific data
- [x] **is_active enforcement** — deactivated users blocked
- [x] **Admin-only endpoints** protected by `verify_admin`
- [x] **HTTPS enforced** by Railway and Vercel

### 🔶 Recommended Next Steps
- [ ] **Set SUPABASE_SERVICE_KEY** in Railway env vars
- [ ] **Run migration 014** in Supabase SQL Editor (the secure rebuild — single source of truth)
- [ ] **🚨 Rotate Claude API key** — hardcoded key was exposed in git history via `update_user_api_key.py`
- [ ] **Rotate all API keys** (Anthropic, Supabase, Tavily) — they may be in git history
- [ ] **Add vector embeddings** to knowledge base for semantic search
- [ ] **Implement token refresh** flow on the frontend
- [ ] **Add Docker non-root user** in Dockerfile
- [ ] **Set up monitoring/alerting** for failed auth attempts

---

*This document is maintained alongside the codebase. Update it when security-relevant changes are made.*
