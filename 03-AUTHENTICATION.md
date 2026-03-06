# 03 — Authentication

> ByPotomac SDK · Authentication & Authorization

---

## Overview

ByPotomac uses **Supabase Auth (GoTrue)** for authentication. All three clients (web, native Windows, direct API) authenticate through the backend's `/auth` endpoints, which proxy to Supabase Auth and return JWT tokens.

```
┌──────────┐     POST /auth/login      ┌──────────┐     signInWithPassword     ┌──────────┐
│  Client  │ ──────────────────────────▶│ Backend  │ ──────────────────────────▶│ Supabase │
│          │◀────────────────────────── │ (FastAPI)│◀────────────────────────── │   Auth   │
└──────────┘     { access_token }       └──────────┘     { access_token }       └──────────┘
```

---

## Authentication Flow

### 1. Registration

```
POST /auth/register
{
  "email": "analyst@firm.com",
  "password": "StrongPass123!",
  "name": "Jane Analyst"
}
```

**What happens server-side:**

1. Backend calls `supabase.auth.sign_up({ email, password })`
2. Supabase creates user in `auth.users`
3. A database trigger auto-creates a `user_profiles` row
4. If the email is in the `ADMIN_EMAILS` config list → `is_admin = true`
5. Backend returns a `Token` with the Supabase JWT

### 2. Login

```
POST /auth/login
{
  "email": "analyst@firm.com",
  "password": "StrongPass123!"
}
```

**What happens server-side:**

1. Backend calls `supabase.auth.sign_in_with_password({ email, password })`
2. Supabase validates credentials and returns a session
3. Backend extracts the `access_token` and returns it as a `Token`

### 3. Using the Token

Every subsequent API call includes the JWT:

```
GET /chat/conversations
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### 4. Token Refresh

```
POST /auth/refresh-token
Authorization: Bearer <current-token>
```

Returns a new token with extended expiry.

### 5. Password Reset

```
POST /auth/forgot-password
{ "email": "analyst@firm.com" }
// → Sends email via Supabase + SMTP

POST /auth/reset-password
{ "token": "<reset-token>", "new_password": "NewPass456!" }
```

---

## JWT Validation (`get_current_user`)

The `api/dependencies.py` module provides the `get_current_user` FastAPI dependency:

```python
async def get_current_user(authorization: str = Header(...)) -> dict:
    """
    1. Extract Bearer token from Authorization header
    2. Call supabase.auth.get_user(token) to validate
    3. Look up user_profiles for is_admin, is_active
    4. Return { id, email, is_admin }
    5. Raise 401 if token invalid or user inactive
    """
```

**Usage in routes:**

```python
@router.get("/brain/documents")
async def list_documents(user: dict = Depends(get_current_user)):
    user_id = user["id"]  # UUID from Supabase Auth
    # ... query documents for this user
```

---

## Admin Authorization

Admin-only routes check `user["is_admin"]`:

```python
@router.get("/admin/users")
async def list_users(user: dict = Depends(get_current_user)):
    if not user.get("is_admin"):
        raise HTTPException(403, "Admin access required")
    # ...
```

Admin emails are configured in `config.py`:

```python
admin_emails: list = ["admin@potomacfund.com"]
```

Users with matching emails automatically get `is_admin = true` on registration.

---

## API Key Management

Users can store their own Anthropic Claude and Tavily API keys. These are **encrypted at rest** using Fernet AES-256 symmetric encryption.

### Encryption Flow

```
User provides API key
  → Backend encrypts with Fernet(ENCRYPTION_KEY)
  → Stored in user_profiles as "enc:<base64-ciphertext>"
  → When needed, decrypt with same ENCRYPTION_KEY
```

**Implementation** (`core/encryption.py`):

```python
from cryptography.fernet import Fernet

class EncryptionManager:
    def __init__(self, key: str):
        self.fernet = Fernet(key.encode())
    
    def encrypt(self, value: str) -> str:
        encrypted = self.fernet.encrypt(value.encode())
        return f"enc:{encrypted.decode()}"
    
    def decrypt(self, value: str) -> str:
        if value.startswith("enc:"):
            value = value[4:]
        return self.fernet.decrypt(value.encode()).decode()
    
    def is_encrypted(self, value: str) -> bool:
        return value.startswith("enc:")
```

### API Key Priority

When making Claude API calls, the backend checks:

1. **User's own key** (decrypted from `user_profiles.claude_api_key`) — preferred
2. **System key** (from `ANTHROPIC_API_KEY` env var) — fallback

This allows individual users to use their own API keys while the system provides a shared fallback.

---

## Client-Side Token Storage

### Web Frontend (Next.js)

```typescript
// src/lib/api.ts
class APIClient {
  private getToken(): string | null {
    return storage.getItem('auth_token');
  }
  
  async login(email: string, password: string) {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    storage.setItem('auth_token', data.access_token);
    return data;
  }
  
  async logout() {
    storage.removeItem('auth_token');
  }
}
```

**Note:** Auth endpoints in the web app are proxied through Next.js API routes (`/api/auth/login`) to avoid CORS issues. All other endpoints call the backend directly.

### Auth Context (`src/contexts/AuthContext.tsx`)

```typescript
const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    // On mount: check for existing token and load user
    const token = storage.getItem('auth_token');
    if (token) {
      apiClient.getMe().then(setUser).catch(logout);
    }
  }, []);
  
  // Provides: user, loading, login, register, logout, updateUser
}

export const useAuth = () => useContext(AuthContext);
```

### Native Windows App (C#)

```csharp
// Services/SessionService.cs
public class SessionService
{
    public string? Token { get; private set; }
    public UserProfile? CurrentUser { get; private set; }
    public bool IsLoggedIn => !string.IsNullOrEmpty(Token);
    
    public void SetSession(string token, UserProfile user)
    {
        Token = token;
        CurrentUser = user;
    }
    
    public void ClearSession()
    {
        Token = null;
        CurrentUser = null;
    }
}

// Services/AuthService.cs
public class AuthService
{
    public async Task<LoginResponse> LoginAsync(string email, string password)
    {
        var response = await _apiService.PostAsync<LoginRequest, LoginResponse>(
            "/auth/login", new LoginRequest { Email = email, Password = password });
        _sessionService.SetSession(response.AccessToken, response.User);
        return response;
    }
}
```

The `ApiService` automatically attaches the token from `SessionService`:

```csharp
// Services/ApiService.cs
private void AddAuthHeader()
{
    var token = _sessionService.Token;
    if (!string.IsNullOrEmpty(token))
    {
        _httpClient.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);
    }
}
```

---

## CORS Configuration

The backend whitelists specific origins:

```python
ALLOWED_ORIGINS = [
    "https://analystbypotomac.vercel.app",
    "https://www.analystbypotomac.vercel.app",
    "https://abpfrontend.vercel.app",
    "https://abpfrontend-sohaib1.vercel.app",
    "http://localhost:3000",
    "http://localhost:3001",
    "http://127.0.0.1:3000",
]
# + FRONTEND_URL env var if set
```

CORS middleware allows:
- All methods
- All headers
- Credentials (cookies/auth headers)

---

## Security Summary

| Aspect | Implementation |
|--------|---------------|
| **Password hashing** | Handled by Supabase Auth (bcrypt) |
| **JWT signing** | Supabase Auth (HS256 with project JWT secret) |
| **Token expiry** | 1 hour (configurable in Supabase) |
| **API key storage** | Fernet AES-256 encryption (`enc:` prefix) |
| **Admin check** | `is_admin` flag in `user_profiles` |
| **Rate limiting** | 120 req/min per IP |
| **CORS** | Origin whitelist |
| **RLS** | Database-level row security per user |

---

*Next: [04 — AI Engine](04-AI-ENGINE.md)*
