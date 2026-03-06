# 06 — Web Frontend

> ByPotomac SDK · Next.js Web Application

**Live URL:** https://analystbypotomac.vercel.app  
**Source:** `C:\Users\SohaibAli\Documents\Abpfrontend`

---

## Technology Stack

| Technology | Version | Purpose |
|-----------|---------|---------|
| **Next.js** | 16 | React framework (App Router) |
| **React** | 19 | UI library |
| **TypeScript** | 5.x | Type safety |
| **Vercel AI SDK** | v6 (`ai@^6.0.78`) | AI chat streaming |
| **@ai-sdk/react** | `^3.0.75` | React hooks for AI |
| **Tailwind CSS** | 3.4 | Styling |
| **Radix UI** | Latest | Accessible UI primitives |
| **shadcn/ui** | Latest | Component library |
| **Recharts** | Latest | Charts and data visualization |
| **pptxgenjs** | Latest | Client-side PPTX generation |
| **Monaco Editor** | Latest | Code editor |
| **react-markdown** | Latest | Markdown rendering |
| **Mermaid** | Latest | Diagram rendering |
| **motion** | Latest | Animations |
| **Sonner** | Latest | Toast notifications |

---

## Project Structure

```
src/
├── app/                          # Next.js App Router
│   ├── layout.tsx                # Root layout (providers, fonts, theme)
│   ├── page.tsx                  # Landing page (/)
│   ├── api/                      # Next.js API routes (proxy layer)
│   │   ├── auth/
│   │   │   ├── login/route.ts    # Proxy: POST /auth/login
│   │   │   ├── register/route.ts # Proxy: POST /auth/register
│   │   │   └── me/route.ts       # Proxy: GET /auth/me
│   │   └── chat/
│   │       └── route.ts          # Chat stream translator (main proxy)
│   ├── (auth)/                   # Auth group (unauthenticated)
│   │   ├── login/page.tsx
│   │   ├── register/page.tsx
│   │   ├── signup/page.tsx
│   │   └── forgot-password/page.tsx
│   ├── (app)/                    # App group (authenticated)
│   │   ├── dashboard/page.tsx
│   │   ├── chat/page.tsx
│   │   ├── research/page.tsx
│   │   ├── analyst/page.tsx
│   │   ├── afl/page.tsx
│   │   ├── backtest/page.tsx
│   │   ├── brain/page.tsx
│   │   ├── skills/page.tsx
│   │   ├── presentations/page.tsx
│   │   ├── train/page.tsx
│   │   ├── upload/page.tsx
│   │   ├── settings/page.tsx
│   │   ├── profile/page.tsx
│   │   ├── admin/page.tsx
│   │   └── history/page.tsx
├── components/
│   ├── ui/                       # shadcn/ui components
│   │   ├── button.tsx
│   │   ├── dialog.tsx
│   │   ├── input.tsx
│   │   ├── card.tsx
│   │   └── ...
│   ├── chat/                     # Chat-specific components
│   │   ├── ChatInterface.tsx
│   │   ├── MessageBubble.tsx
│   │   ├── ToolCallDisplay.tsx
│   │   └── StreamingIndicator.tsx
│   ├── layout/                   # Layout components
│   │   ├── Sidebar.tsx
│   │   ├── Header.tsx
│   │   └── Navigation.tsx
│   └── ...
├── contexts/
│   └── AuthContext.tsx            # Authentication state provider
├── hooks/                        # Custom React hooks
├── lib/
│   ├── api.ts                    # APIClient class (all backend calls)
│   ├── env.ts                    # Environment variable helpers
│   ├── utils.ts                  # Utility functions
│   └── storage.ts                # localStorage wrapper
└── styles/
    └── globals.css               # Global styles + Tailwind
```

---

## Backend Communication Architecture

The frontend uses **two communication patterns**:

### Pattern 1: Direct API Calls

For REST endpoints (non-streaming), the `APIClient` class in `src/lib/api.ts` calls the backend directly:

```typescript
class APIClient {
  private API_BASE_URL: string; // From NEXT_PUBLIC_API_URL env var
  
  private async request<T>(url: string, options: RequestInit = {}): Promise<T> {
    const token = storage.getItem('auth_token');
    const response = await fetch(`${this.API_BASE_URL}${url}`, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...(token ? { 'Authorization': `Bearer ${token}` } : {}),
        ...options.headers,
      },
    });
    if (!response.ok) throw new APIError(response.status, await response.json());
    return response.json();
  }
}
```

**Endpoints called directly:**

| Feature | Endpoint |
|---------|----------|
| Brain/Knowledge | `GET/POST /brain/documents`, `POST /brain/query` |
| AFL Generator | `POST /afl/generate`, `GET /afl/history` |
| Backtest | `POST /backtest/run`, `GET /backtest/sessions` |
| Researcher | `POST /researcher/research` |
| Skills | `GET/POST/PUT/DELETE /skills` |
| Training | `GET/POST /train/examples` |
| Presentations | `POST /presentations/generate` |
| Yahoo Finance | `GET /yfinance/quote/{ticker}` |
| Files/Upload | `POST /files/upload`, `POST /upload/parse` |
| Content | `POST /content/generate` |
| Reverse Engineer | `POST /reverse-engineer/analyze` |

### Pattern 2: Proxied Through Next.js API Routes

Auth and chat streaming are proxied through Next.js API routes to handle CORS and protocol translation:

```
Browser → /api/auth/login (Next.js) → /auth/login (Backend)
Browser → /api/chat (Next.js)       → /chat/stream (Backend)
```

**Why proxy auth?** — Avoids CORS preflight issues since the browser talks to same-origin Next.js routes.

**Why proxy chat?** — The Next.js route translates the Data Stream Protocol into UI Message Stream format that Vercel AI SDK v6's `useChat()` expects.

---

## Chat Implementation

### Using Vercel AI SDK `useChat()`

```typescript
import { useChat } from '@ai-sdk/react';

function ChatPage() {
  const {
    messages,
    input,
    handleInputChange,
    handleSubmit,
    isLoading,
    error,
    append,
    setMessages,
  } = useChat({
    api: '/api/chat',
    headers: {
      'Authorization': `Bearer ${token}`,
    },
    body: {
      conversationId,
      model: selectedModel,
      brain_enabled: brainEnabled,
      tools_enabled: toolsEnabled,
      skills: activeSkillIds,
    },
    onFinish: (message) => {
      // Save conversation, update UI
    },
    onError: (error) => {
      toast.error(error.message);
    },
  });
  
  return (
    <div>
      {messages.map(msg => <MessageBubble key={msg.id} message={msg} />)}
      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} />
        <button type="submit" disabled={isLoading}>Send</button>
      </form>
    </div>
  );
}
```

### Chat Proxy Route (`app/api/chat/route.ts`)

The proxy translates between protocols:

```typescript
export async function POST(req: Request) {
  const body = await req.json();
  const token = req.headers.get('authorization');
  
  // Forward to backend
  const response = await fetch(`${API_URL}/chat/stream`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': token,
    },
    body: JSON.stringify(body),
  });
  
  // Transform Data Stream Protocol → UI Message Stream
  const transformStream = new TransformStream({
    transform(chunk, controller) {
      const lines = new TextDecoder().decode(chunk).split('\n');
      for (const line of lines) {
        if (!line.trim()) continue;
        const colonIdx = line.indexOf(':');
        const type = line.substring(0, colonIdx);
        const payload = line.substring(colonIdx + 1);
        
        switch (type) {
          case '0': // Text → text-delta
            controller.enqueue(`{"type":"text-delta","textDelta":${payload}}\n`);
            break;
          case '9': // Tool call
            controller.enqueue(`{"type":"tool-call",...}\n`);
            break;
          case 'a': // Tool result  
            controller.enqueue(`{"type":"tool-result",...}\n`);
            break;
          case 'd': // Finish
            controller.enqueue(`{"type":"finish",...}\n`);
            break;
        }
      }
    }
  });
  
  return new Response(response.body.pipeThrough(transformStream), {
    headers: { 'Content-Type': 'text/event-stream' },
  });
}
```

---

## Pages Overview

### Public Pages (Unauthenticated)

| Route | Page | Description |
|-------|------|-------------|
| `/` | Landing | Marketing page with feature overview |
| `/login` | Login | Email + password login form |
| `/register` | Register | Registration form |
| `/signup` | Signup | Alternate registration |
| `/forgot-password` | Password Reset | Email-based reset |

### Authenticated Pages

| Route | Page | Description |
|-------|------|-------------|
| `/dashboard` | Dashboard | Overview cards, recent activity, quick actions |
| `/chat` | Chat | AI chat interface with tool calling display |
| `/research` | Research | Multi-source research with progress tracking |
| `/analyst` | Analyst | Specialized analysis workspace |
| `/afl` | AFL Generator | American Funds Letters with compliance |
| `/backtest` | Backtest | Strategy backtesting with charts |
| `/brain` | Brain | Knowledge base document manager |
| `/skills` | Skills | Create/manage AI skill modules |
| `/presentations` | Presentations | PPTX deck generator |
| `/train` | Training | Add/manage training examples |
| `/upload` | Upload | File upload with AI analysis |
| `/settings` | Settings | App configuration |
| `/profile` | Profile | User profile, API key management |
| `/admin` | Admin | User management (admin only) |
| `/history` | History | Conversation history browser |

---

## Authentication Flow (Client-Side)

```typescript
// src/contexts/AuthContext.tsx
export function AuthProvider({ children }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = storage.getItem('auth_token');
    if (token) {
      api.auth.getMe()
        .then(user => setUser(user))
        .catch(() => {
          storage.removeItem('auth_token');
          setUser(null);
        })
        .finally(() => setLoading(false));
    } else {
      setLoading(false);
    }
  }, []);

  const login = async (email: string, password: string) => {
    const { access_token, ...userData } = await api.auth.login(email, password);
    storage.setItem('auth_token', access_token);
    setUser(userData);
    router.push('/dashboard');
  };

  const logout = () => {
    storage.removeItem('auth_token');
    setUser(null);
    router.push('/login');
  };

  return (
    <AuthContext.Provider value={{ user, loading, login, logout, ... }}>
      {children}
    </AuthContext.Provider>
  );
}
```

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `NEXT_PUBLIC_API_URL` | Yes | Backend API URL |
| `NEXT_PUBLIC_APP_NAME` | No | App display name |

**Production:** `NEXT_PUBLIC_API_URL=https://potomac-analyst-workbench-production.up.railway.app`  
**Development:** `NEXT_PUBLIC_API_URL=http://localhost:8070`

---

## Build & Dev Commands

```bash
npm run dev          # Start dev server (Turbopack)
npm run build        # Production build
npm run start        # Start production server
npm run lint         # ESLint
```

---

## Deployment

The web app deploys to **Vercel** (or Railway) via Git push:

- **Vercel:** Automatic deployment from Git repo
- **Railway:** Uses `nixpacks.toml` + `railway.json` config
- **Build:** SWC + Turbopack (no Babel)

---

*Next: [07 — Native Windows App](07-NATIVE-WINDOWS.md)*
