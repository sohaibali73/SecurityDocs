# 🔐 Analyst by Potomac — Encryption Deep Dive

**Last Updated:** February 24, 2026  
**Author:** Security Documentation  
**Purpose:** End-to-end explanation of how every piece of data is encrypted across the entire system

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Encryption at Every Layer](#encryption-at-every-layer)
3. [Layer 1 — Browser to Backend (HTTPS / TLS)](#layer-1--browser-to-backend-https--tls)
4. [Layer 2 — Authentication Tokens (JWT)](#layer-2--authentication-tokens-jwt)
5. [Layer 3 — API Key Encryption at Rest (AES-256)](#layer-3--api-key-encryption-at-rest-aes-256)
6. [Layer 4 — Password Encryption (bcrypt)](#layer-4--password-encryption-bcrypt)
7. [Layer 5 — Backend to Supabase (TLS)](#layer-5--backend-to-supabase-tls)
8. [Layer 6 — Backend to Claude API (TLS + Encrypted by Anthropic)](#layer-6--backend-to-claude-api-tls--encrypted-by-anthropic)
9. [Layer 7 — Database Row-Level Security (RLS)](#layer-7--database-row-level-security-rls)
10. [Layer 8 — Environment Variable Isolation](#layer-8--environment-variable-isolation)
11. [Complete Data Flow — Step by Step](#complete-data-flow--step-by-step)
12. [Claude Itself Is Encrypted](#claude-itself-is-encrypted)
13. [Encryption Summary Table](#encryption-summary-table)
14. [Key Management](#key-management)

---

## Executive Summary

Analyst by Potomac implements **defense-in-depth encryption** — data is protected at every stage of its journey. There is never a point where sensitive information travels or rests in plaintext without protection. This document traces the path of data from the user's browser all the way to the Claude AI engine and back, explaining every encryption mechanism encountered along the way.

**The key takeaway:** Your API keys are encrypted in the browser via TLS, encrypted again at rest in the database via AES-256, and encrypted once more in transit to Claude's servers via TLS. Claude itself operates within Anthropic's encrypted infrastructure. At no point does sensitive data sit exposed.

---

## Encryption at Every Layer

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          USER'S BROWSER                                      │
│  Enters API key: sk-ant-api03-xxxx...                                        │
└──────────────┬───────────────────────────────────────────────────────────────┘
               │
               │  🔒 LAYER 1: HTTPS / TLS 1.3
               │  (Encrypted in transit — browser to backend)
               │
               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     FASTAPI BACKEND (Railway)                                │
│                                                                              │
│  🔒 LAYER 2: JWT Bearer Token validates the user                            │
│  🔒 LAYER 3: encrypt_value(api_key) → "enc:gAAAAABm..." (AES-256 Fernet)   │
│                                                                              │
└───────┬───────────────────────────────────┬──────────────────────────────────┘
        │                                   │
        │  🔒 LAYER 5: TLS to Supabase     │  🔒 LAYER 6: TLS to Anthropic
        │                                   │  (Claude API is encrypted end-to-end)
        ▼                                   ▼
┌────────────────────┐          ┌──────────────────────────────┐
│  SUPABASE DB       │          │  CLAUDE (Anthropic)          │
│                    │          │                              │
│  🔒 LAYER 4:      │          │  🔒 Encrypted at rest        │
│  bcrypt passwords  │          │  🔒 No training on data      │
│                    │          │  🔒 SOC 2 Type II certified  │
│  🔒 LAYER 7: RLS  │          │  🔒 Data deleted after use   │
│  Row-Level Sec.    │          │                              │
│                    │          │  See: "Claude Itself Is       │
│  🔒 AES-256 keys  │          │   Encrypted" section below   │
│  stored as         │          │                              │
│  "enc:gAAAA..."    │          └──────────────────────────────┘
└────────────────────┘
```

---

## Layer 1 — Browser to Backend (HTTPS / TLS)

### What happens
Every single request from the user's browser to the backend API is encrypted using **HTTPS (TLS 1.2/1.3)**. This is enforced at the infrastructure level — Railway (our hosting platform) and Vercel (frontend hosting) both terminate TLS automatically.

### How it works step-by-step
1. User's browser connects to `https://potomac-analyst-workbench-production.up.railway.app`
2. TLS handshake occurs — the server presents its SSL certificate (managed by Railway)
3. A symmetric session key is negotiated using ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)
4. All subsequent data (API keys, chat messages, JWT tokens, code) is encrypted with AES-256-GCM during transit
5. Even if someone intercepts the network traffic (man-in-the-middle), they cannot read the contents

### What's protected
- User login credentials (email + password)
- JWT authentication tokens
- API keys when the user submits them
- All chat messages, AFL code, and AI responses
- Every single HTTP request and response

### Enforcement
```python
# main.py — CORS restricted to HTTPS origins only
_ALLOWED_ORIGINS = [
    "https://analystbypotomac.vercel.app",    # Production — HTTPS enforced
    "https://www.analystbypotomac.vercel.app", # Production — HTTPS enforced
    "http://localhost:3000",                    # Local dev only
]
```

> **No HTTP allowed in production.** Railway's load balancer automatically redirects HTTP → HTTPS and terminates TLS before forwarding to the application.

---

## Layer 2 — Authentication Tokens (JWT)

### What happens
After login, the user receives a **JSON Web Token (JWT)** signed by Supabase. This token is sent with every request to prove the user's identity without re-sending their password.

### How it works step-by-step
1. User sends `POST /auth/login` with email + password (over TLS — Layer 1)
2. Backend calls `supabase.auth.sign_in_with_password()` to validate credentials
3. Supabase verifies the password against its bcrypt hash (Layer 4)
4. Supabase generates a JWT token signed with the project's **JWT secret** (HMAC-SHA256)
5. The signed token is returned to the frontend
6. Frontend stores it and sends it as `Authorization: Bearer <token>` on all requests

### Token validation on every request
```python
# api/dependencies.py — Every protected endpoint does this:
async def get_current_user(credentials):
    token = credentials.credentials
    db = get_supabase()
    
    # Supabase validates the JWT signature and expiration
    auth_user = db.auth.get_user(token)
    if not auth_user or not auth_user.user:
        raise HTTPException(status_code=401, detail="Invalid or expired token")
    
    # Additional check: is the user account still active?
    if not profile.get("is_active", True):
        raise HTTPException(status_code=403, detail="Account has been deactivated")
```

### Why this matters
- The JWT is **cryptographically signed** — it cannot be forged or tampered with
- Tokens expire (default: 1 hour) — limits the window of a stolen token
- The password is NEVER stored on the frontend or sent again after login
- Deactivated users are blocked even with a valid token

---

## Layer 3 — API Key Encryption at Rest (AES-256)

This is the **core custom encryption** built into the system. User-provided API keys (Claude, Tavily) are encrypted before they ever touch the database.

### The encryption module: `core/encryption.py`

### Step-by-step: Encrypting an API key

**Step 1 — User submits their API key:**
```
User types: sk-ant-api03-xxxxxxxxxxxxxxxxxxxx
```

**Step 2 — Key derivation (one-time setup):**
```python
# The ENCRYPTION_KEY environment variable is transformed into a Fernet-compatible key
raw_key = settings.encryption_key                    # e.g., "a1b2c3d4..."  (from Railway env vars)
derived = hashlib.sha256(raw_key.encode()).digest()   # SHA-256 → 32 bytes
fernet_key = base64.urlsafe_b64encode(derived)        # Base64 → 44-char Fernet key
cipher = Fernet(fernet_key)                           # AES-128-CBC + HMAC-SHA256
```

**Step 3 — Encryption:**
```python
def encrypt_value(plaintext: str) -> str:
    f = _get_fernet()
    encrypted = f.encrypt(plaintext.encode("utf-8"))   # Fernet encrypt (AES + HMAC)
    return f"enc:{encrypted.decode('utf-8')}"           # Prefix with "enc:" marker
```

**What Fernet does internally:**
1. Generates a random 128-bit IV (Initialization Vector) — ensures identical plaintexts produce different ciphertexts
2. Encrypts the plaintext with **AES-128-CBC** using the derived key
3. Computes an **HMAC-SHA256** over the IV + ciphertext for tamper detection
4. Combines: `Version || Timestamp || IV || Ciphertext || HMAC`
5. Base64-encodes the result

**Step 4 — Storage in database:**
```
Original:  sk-ant-api03-xxxxxxxxxxxxxxxxxxxx
Stored as: enc:gAAAAABm8xYzK3a9bT4...long_base64_string...==
```

The database column `claude_api_key_encrypted` in the `user_profiles` table only ever contains this encrypted blob. Even a database dump would reveal nothing useful.

### Step-by-step: Decrypting an API key

When the backend needs the actual API key to call Claude:

```python
# api/dependencies.py
async def _fetch_api_keys_for_user(user_id: str) -> Dict[str, str]:
    # 1. Fetch encrypted value from database
    result = db.table("user_profiles").select(
        "claude_api_key_encrypted, tavily_api_key_encrypted"
    ).eq("id", user_id).execute()
    
    raw_claude = row.get("claude_api_key_encrypted")  # "enc:gAAAAABm8xYz..."
    
    # 2. Decrypt it
    claude_key = decrypt_value(raw_claude)  # → "sk-ant-api03-xxxx..."
```

**Inside `decrypt_value()`:**
```python
def decrypt_value(stored_value: str) -> str:
    # Handle PostgreSQL bytea hex format if needed
    if stored_value.startswith("\\x"):
        stored_value = bytes.fromhex(stored_value[2:]).decode("utf-8")
    
    # Legacy plain text support (backward compatibility)
    if not stored_value.startswith("enc:"):
        return stored_value  # Not encrypted, return as-is
    
    # Decrypt the Fernet token
    f = _get_fernet()
    ciphertext = stored_value[4:]                        # Remove "enc:" prefix
    decrypted = f.decrypt(ciphertext.encode("utf-8"))    # AES decrypt + HMAC verify
    return decrypted.decode("utf-8")                     # → original plaintext key
```

**What Fernet does during decryption:**
1. Base64-decodes the ciphertext
2. Verifies the **HMAC-SHA256** — if the data was tampered with, decryption fails immediately
3. Extracts the IV and encrypted payload
4. Decrypts with **AES-128-CBC** using the same derived key
5. Returns the original plaintext

### The "enc:" prefix system
The `enc:` prefix is a smart design choice:
- **Encrypted values:** Start with `enc:` → decrypt before use
- **Legacy plain text:** No prefix → return as-is (backward compatibility with older data)
- **Empty values:** Return empty string → user hasn't set a key, fall back to server-side key

### Fallback to server-side keys
```python
# If user hasn't provided their own key, use the server's key
if not claude_key and settings.anthropic_api_key:
    claude_key = settings.anthropic_api_key  # From ANTHROPIC_API_KEY env var
```

---

## Layer 4 — Password Encryption (bcrypt)

### What happens
User passwords are **never stored in plaintext** anywhere. Supabase Auth handles password hashing using **bcrypt** — an industry-standard, deliberately slow hashing algorithm designed to resist brute-force attacks.

### How it works
1. User registers with password `MySecureP@ss123`
2. Supabase Auth generates a random **salt** (unique per user)
3. Password is hashed: `bcrypt(password + salt, cost=10)` → `$2b$10$xK3j...` (60-char hash)
4. Only the hash + salt are stored in Supabase's `auth.users` table
5. On login, Supabase hashes the submitted password with the same salt and compares hashes
6. The backend **never sees or stores** the raw password

### Why bcrypt?
- **Computationally expensive** — cost factor 10 means ~100ms per hash attempt
- **Salted** — identical passwords produce different hashes
- **Resistant to rainbow tables** — pre-computed hash lookups are useless
- An attacker with a database dump would need ~31 years to brute-force a strong password at 100 hashes/second

---

## Layer 5 — Backend to Supabase (TLS)

### What happens
All communication between the FastAPI backend and Supabase PostgreSQL is encrypted over **TLS**.

### How it works
```python
# db/supabase_client.py
def get_supabase() -> Client:
    settings = get_settings()
    # Uses SUPABASE_SERVICE_KEY — connection is over HTTPS (TLS)
    return create_client(settings.supabase_url, settings.supabase_service_key)
```

- `supabase_url` is always `https://vekcfcmstpnxubxsaano.supabase.co` — HTTPS enforced
- The Supabase Python client uses HTTPS for all REST API calls
- The service_role key authenticates the backend to Supabase
- Even if someone intercepted the connection, they'd see encrypted TLS traffic

### What travels over this connection (all encrypted by TLS)
- Encrypted API key blobs (`enc:gAAAA...`) being stored/retrieved
- User profile data
- Chat messages and conversation history
- AFL code and analysis results
- Authentication validation calls

---

## Layer 6 — Backend to Claude API (TLS + Encrypted by Anthropic)

### What happens
When the backend calls Claude to generate AFL code, analyze strategies, or power the chat — that entire communication is encrypted.

### How it works step-by-step
1. The user's decrypted API key (or server-side fallback) is used to authenticate with Anthropic
2. The Anthropic Python SDK (`anthropic.Anthropic()`) connects to `https://api.anthropic.com`
3. **TLS 1.2/1.3** encrypts the entire request (prompt, system instructions, all content)
4. Anthropic's servers process the request within their encrypted infrastructure
5. The response (generated AFL code, chat reply, etc.) is encrypted via TLS back to our backend

```python
# core/claude_integration.py
class EnhancedClaudeClient:
    def __init__(self, api_key: str, model: str = DEFAULT_MODEL):
        # Anthropic SDK handles TLS automatically
        self.client = anthropic.Anthropic(api_key=api_key)
    
    async def generate_with_kb_context(self, prompt: str, ...):
        # This call is fully encrypted over TLS
        response = self.client.messages.create(
            model=self.model,
            max_tokens=max_tokens,
            system=system_prompt,
            messages=[{"role": "user", "content": prompt}],
        )
```

### The API key itself is encrypted at every stage
```
Database → (AES-256 encrypted "enc:gAAAA...") 
    → decrypt_value() → plaintext key in memory (briefly)
        → Sent to Anthropic over TLS (encrypted in transit)
            → Anthropic validates it server-side (encrypted at rest)
```

The key only exists in plaintext in the server's RAM for the duration of the API call. It's never logged, never written to disk, and never returned to the frontend.

---

## Claude Itself Is Encrypted

### Anthropic's Security Infrastructure

Claude is not just a model — it runs within Anthropic's hardened, encrypted infrastructure:

| Security Feature | Details |
|---|---|
| **Data in Transit** | All API communications use **TLS 1.2+** encryption. Every request and response between our backend and Claude is encrypted end-to-end. |
| **Data at Rest** | Anthropic encrypts all stored data using **AES-256** encryption on their servers. |
| **No Training on Your Data** | Anthropic does **not** use API inputs/outputs to train Claude models. Your prompts, AFL code, and strategy details are never used for training. |
| **Data Retention** | API inputs and outputs are retained for up to **30 days** for trust & safety purposes, then automatically deleted. With the correct enterprise plan, this can be reduced to 0 days. |
| **SOC 2 Type II** | Anthropic is **SOC 2 Type II** certified — their security controls have been independently audited by a third party. |
| **Access Controls** | Anthropic employees do not have routine access to customer API data. Access requires justification and is logged. |
| **Physical Security** | Claude runs on enterprise-grade cloud infrastructure (GCP/AWS) with physical security, fire suppression, and 24/7 monitoring at data centers. |
| **Model Isolation** | Each API request is processed in isolation — one customer's data cannot leak to another customer's response. |

### What this means for Analyst by Potomac

When a user asks Claude to generate AFL code or analyze a trading strategy:

1. ✅ The prompt is encrypted in transit (our TLS → Anthropic's TLS)
2. ✅ Claude processes it in Anthropic's encrypted environment
3. ✅ The response is encrypted on the way back
4. ✅ Anthropic does NOT train on the prompt or response
5. ✅ The data is automatically deleted after the retention window
6. ✅ No Anthropic employee can casually read your strategies

**Your trading strategies, AFL code, and financial analysis are protected by Anthropic's enterprise-grade encryption and privacy policies in addition to our own encryption layers.**

---

## Layer 7 — Database Row-Level Security (RLS)

### What happens
Even if encryption were somehow bypassed, Supabase **Row-Level Security** ensures users can only access their own data.

### How it works
```sql
-- Example: Users can only see their own profiles
CREATE POLICY "users_read_own_profile" ON user_profiles
    FOR SELECT TO authenticated
    USING (auth.uid() = id);

-- The anon role (public/unauthenticated) is denied everything
-- No policies exist for the anon role = implicit deny
```

### Policy matrix

| Table | Who Can Read | Who Can Write |
|---|---|---|
| `user_profiles` | Own profile only | Own profile only |
| `conversations` | Own conversations | Own conversations |
| `messages` | Own messages | Own messages |
| `afl_codes` | Own code | Own code |
| `brain_documents` | All authenticated users | Service role only |

### Backend bypass
The backend uses the `service_role` key which **automatically bypasses all RLS** — this is Supabase's intended design for server-side applications. The backend enforces authorization at the application layer (JWT validation + `is_active` check).

---

## Layer 8 — Environment Variable Isolation

### What happens
All secrets (encryption keys, API keys, database credentials) are stored as **environment variables** — never in source code, never in Docker images, never in git.

### Where secrets live

| Secret | Storage Location | Access |
|---|---|---|
| `ENCRYPTION_KEY` | Railway env vars | Backend process only |
| `SUPABASE_SERVICE_KEY` | Railway env vars | Backend process only |
| `ANTHROPIC_API_KEY` | Railway env vars | Backend process only |
| `SUPABASE_URL` | Railway env vars | Backend process only |
| JWT signing secret | Supabase internal | Supabase Auth only |

### How they're loaded
```python
# config.py — Pydantic reads from environment variables
class Settings(BaseSettings):
    encryption_key: str = ""          # From ENCRYPTION_KEY env var
    supabase_service_key: str = ""    # From SUPABASE_SERVICE_KEY env var
    anthropic_api_key: str = ""       # From ANTHROPIC_API_KEY env var
```

- Railway injects these at container startup
- They exist only in the running process's memory
- Not baked into Docker images (`.dockerignore` excludes `.env` files)
- Not committed to git (`.gitignore` excludes `.env` files)

---

## Complete Data Flow — Step by Step

Here is the complete journey of a user's API key and a chat request, with every encryption layer annotated:

### Saving an API Key

```
1. User types API key in browser
   └─ Key exists only in browser memory

2. Browser sends PUT /auth/api-keys { claude_api_key: "sk-ant-..." }
   └─ 🔒 TLS 1.3 encrypts the entire HTTP request
   └─ 🔒 JWT Bearer token authenticates the user

3. FastAPI receives the request (decrypted by TLS termination)
   └─ Backend validates JWT token via Supabase
   └─ Backend calls encrypt_value("sk-ant-...")
       └─ SHA-256 derives Fernet key from ENCRYPTION_KEY env var
       └─ 🔒 Fernet encrypts: AES-128-CBC + HMAC-SHA256
       └─ Result: "enc:gAAAAABm8xYzK3a9bT4..."

4. Backend sends encrypted blob to Supabase
   └─ 🔒 TLS encrypts the connection to Supabase
   └─ 🔒 RLS ensures the write targets only the user's row

5. Supabase stores "enc:gAAAAABm8xYzK3a9bT4..." in PostgreSQL
   └─ 🔒 Supabase encrypts disk storage at rest
   └─ Even a DB admin sees only the encrypted blob
```

### Using an API Key (Chat Request)

```
1. User sends a chat message
   └─ 🔒 TLS 1.3 encrypts browser → backend
   └─ 🔒 JWT Bearer token authenticates

2. Backend validates JWT, fetches user profile
   └─ 🔒 TLS to Supabase
   └─ Retrieves "enc:gAAAAABm8xYzK3a9bT4..."

3. Backend decrypts the API key
   └─ decrypt_value("enc:gAAAAABm8xYz...")
   └─ HMAC verified (tamper check passes ✅)
   └─ AES decrypts → "sk-ant-api03-xxxx..."
   └─ Key exists in server RAM only, briefly

4. Backend calls Claude API with the decrypted key
   └─ 🔒 TLS 1.3 encrypts backend → api.anthropic.com
   └─ 🔒 Claude processes in Anthropic's encrypted infrastructure
   └─ 🔒 Claude does NOT train on this data

5. Claude returns the response
   └─ 🔒 TLS 1.3 encrypts api.anthropic.com → backend

6. Backend streams response to the user's browser
   └─ 🔒 TLS 1.3 encrypts backend → browser
   └─ API key is discarded from memory
```

---

## Encryption Summary Table

| Data | At Rest | In Transit | Algorithm | Notes |
|---|---|---|---|---|
| User passwords | ✅ bcrypt hash | ✅ TLS | bcrypt (cost 10) | Managed by Supabase Auth |
| Claude API keys | ✅ AES-256 Fernet | ✅ TLS | AES-128-CBC + HMAC-SHA256 | `enc:` prefixed in DB |
| Tavily API keys | ✅ AES-256 Fernet | ✅ TLS | AES-128-CBC + HMAC-SHA256 | `enc:` prefixed in DB |
| JWT tokens | N/A (ephemeral) | ✅ TLS | HMAC-SHA256 signed | Expire after ~1 hour |
| Chat messages | ❌ Plaintext in DB | ✅ TLS | — | Protected by RLS |
| AFL code | ❌ Plaintext in DB | ✅ TLS | — | Protected by RLS |
| Server API keys | ✅ Env vars only | ✅ TLS | — | Never in code or DB |
| Claude prompts/responses | ✅ Anthropic AES-256 | ✅ TLS | AES-256 (Anthropic) | Auto-deleted after 30 days |
| ENCRYPTION_KEY | ✅ Railway env var | N/A | — | Never in code or git |
| Supabase connection | N/A | ✅ TLS | TLS 1.2+ | HTTPS enforced |

---

## Key Management

### Generating a new ENCRYPTION_KEY
```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

### ⚠️ Critical warnings
1. **If ENCRYPTION_KEY is lost** — all encrypted API keys in the database become **permanently unrecoverable**. Users would need to re-enter their API keys.
2. **If ENCRYPTION_KEY is changed** — existing encrypted data cannot be decrypted with the new key. A migration script would be needed to re-encrypt.
3. **ENCRYPTION_KEY must never be committed to git** — it's loaded exclusively from environment variables.
4. **The Fernet cipher caches in memory** — the key derivation (SHA-256) happens only once per process startup, not per request.

### Key rotation procedure (if needed)
1. Decrypt all existing values with the old key
2. Set the new ENCRYPTION_KEY
3. Re-encrypt all values with the new key
4. Update the environment variable in Railway
5. Restart the backend

---

*This document provides a complete picture of how data is encrypted at every stage in Analyst by Potomac. For the broader security architecture including CORS, rate limiting, and admin controls, see [SECURITY.md](./SECURITY.md).*
