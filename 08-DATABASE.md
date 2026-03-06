# 08 — Database Schema

> ByPotomac SDK · Supabase PostgreSQL Database

---

## Overview

ByPotomac uses **Supabase** as its database platform, providing:

- **PostgreSQL 15** with pgvector extension for embeddings
- **Supabase Auth (GoTrue)** for user authentication
- **Row Level Security (RLS)** for data isolation
- **Storage Buckets** for file uploads
- **14 migrations** that build the complete schema

---

## Database Client (`db/supabase_client.py`)

```python
from supabase import create_client
from functools import lru_cache

@lru_cache(maxsize=1)
def get_supabase():
    """
    Singleton Supabase client.
    Prefers service_role key (bypasses RLS) for backend operations.
    Falls back to anon key with warning.
    """
    return create_client(
        settings.supabase_url,
        settings.supabase_service_key or settings.supabase_key
    )

def get_supabase_with_token(token: str):
    """
    Create a client authenticated with a user's JWT.
    Used when service_role key is not available.
    """
    client = create_client(settings.supabase_url, settings.supabase_key)
    client.auth.set_session(token)
    return client
```

---

## Complete Table Schema

### User & Auth Tables

#### `user_profiles`

The main user table, auto-created via trigger when a user signs up.

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK → `auth.users` | Supabase Auth user ID |
| `email` | `TEXT` NOT NULL UNIQUE | User email |
| `name` | `TEXT` | Display name |
| `nickname` | `TEXT` | Short name |
| `avatar_url` | `TEXT` | Profile picture URL |
| `is_admin` | `BOOLEAN` DEFAULT `false` | Admin flag |
| `is_active` | `BOOLEAN` DEFAULT `true` | Account active |
| `claude_api_key` | `TEXT` | Encrypted (`enc:` prefix) Anthropic key |
| `tavily_api_key` | `TEXT` | Encrypted (`enc:` prefix) Tavily key |
| `preferences` | `JSONB` | User preferences |
| `created_at` | `TIMESTAMPTZ` | Registration time |
| `updated_at` | `TIMESTAMPTZ` | Last update |
| `last_active_at` | `TIMESTAMPTZ` | Last activity |

**Trigger:** `on_auth_user_created` → auto-creates `user_profiles` row.

---

### Conversation & Messages

#### `conversations`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | Conversation ID |
| `user_id` | `UUID` → `auth.users` | Owner |
| `title` | `TEXT` | Conversation title |
| `summary` | `TEXT` | AI-generated summary |
| `system_prompt` | `TEXT` | Custom system prompt |
| `model` | `TEXT` | AI model used |
| `settings` | `JSONB` | Conversation settings |
| `is_archived` | `BOOLEAN` DEFAULT `false` | Archived flag |
| `message_count` | `INTEGER` DEFAULT `0` | Message counter |
| `created_at` | `TIMESTAMPTZ` | Creation time |
| `updated_at` | `TIMESTAMPTZ` | Last message time |

#### `messages`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | Message ID |
| `conversation_id` | `UUID` → `conversations` | Parent conversation |
| `role` | `TEXT` NOT NULL | `user`, `assistant`, `system`, `tool` |
| `content` | `TEXT` | Message text |
| `metadata` | `JSONB` | Tool calls, artifacts, etc. |
| `token_count` | `INTEGER` | Tokens used |
| `created_at` | `TIMESTAMPTZ` | Send time |

#### `conversation_files`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | Link ID |
| `conversation_id` | `UUID` → `conversations` | Conversation |
| `file_name` | `TEXT` | Original filename |
| `file_path` | `TEXT` | Storage path |
| `file_type` | `TEXT` | MIME type |
| `file_size` | `BIGINT` | Size in bytes |
| `created_at` | `TIMESTAMPTZ` | Upload time |

---

### Brain / Knowledge Base

#### `brain_documents`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | Document ID |
| `user_id` | `UUID` → `auth.users` | Owner |
| `title` | `TEXT` | Document title |
| `description` | `TEXT` | Description |
| `file_name` | `TEXT` | Original filename |
| `file_path` | `TEXT` | Storage bucket path |
| `file_type` | `TEXT` | MIME type |
| `file_size` | `BIGINT` | Size in bytes |
| `content_text` | `TEXT` | Extracted full text |
| `chunk_count` | `INTEGER` DEFAULT `0` | Number of chunks |
| `status` | `TEXT` DEFAULT `'processing'` | `processing`, `ready`, `error` |
| `metadata` | `JSONB` | Extra metadata |
| `created_at` | `TIMESTAMPTZ` | Upload time |
| `updated_at` | `TIMESTAMPTZ` | Last update |

#### `brain_document_chunks`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | Chunk ID |
| `document_id` | `UUID` → `brain_documents` | Parent document |
| `user_id` | `UUID` → `auth.users` | Owner |
| `chunk_index` | `INTEGER` | Position in document |
| `chunk_text` | `TEXT` | Chunk content |
| `token_count` | `INTEGER` | Tokens in chunk |
| `embedding` | `vector(1536)` | pgvector embedding |
| `metadata` | `JSONB` | Position info |
| `created_at` | `TIMESTAMPTZ` | Creation time |

**Index:** `brain_document_chunks_embedding_idx` using `ivfflat` for cosine similarity search.

---

### Skills

#### `skills`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | Skill ID |
| `user_id` | `UUID` → `auth.users` | Creator |
| `name` | `TEXT` NOT NULL | Skill name |
| `description` | `TEXT` | What the skill does |
| `system_prompt` | `TEXT` NOT NULL | Prompt injected into Claude |
| `category` | `TEXT` | Grouping category |
| `is_active` | `BOOLEAN` DEFAULT `true` | Enabled |
| `is_global` | `BOOLEAN` DEFAULT `false` | Available to all users |
| `usage_count` | `INTEGER` DEFAULT `0` | Times used |
| `metadata` | `JSONB` | Extra config |
| `created_at` | `TIMESTAMPTZ` | Creation time |
| `updated_at` | `TIMESTAMPTZ` | Last update |

---

### Training

#### `training_examples`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | Example ID |
| `user_id` | `UUID` → `auth.users` | Creator |
| `input` | `TEXT` NOT NULL | Example question/input |
| `output` | `TEXT` NOT NULL | Expected answer/output |
| `category` | `TEXT` | Category |
| `tags` | `TEXT[]` | Array of tags |
| `quality_score` | `FLOAT` | Quality rating (0-1) |
| `is_active` | `BOOLEAN` DEFAULT `true` | Active |
| `metadata` | `JSONB` | Extra info |
| `created_at` | `TIMESTAMPTZ` | Creation time |

#### `feedback`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | Feedback ID |
| `user_id` | `UUID` → `auth.users` | User |
| `message_id` | `UUID` → `messages` | Message rated |
| `rating` | `INTEGER` | 1-5 star rating |
| `comment` | `TEXT` | Optional comment |
| `category` | `TEXT` | Feedback type |
| `created_at` | `TIMESTAMPTZ` | Time |

---

### Research

#### `research_sessions`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | Session ID |
| `user_id` | `UUID` → `auth.users` | User |
| `query` | `TEXT` NOT NULL | Research query |
| `status` | `TEXT` DEFAULT `'pending'` | `pending`, `running`, `complete`, `error` |
| `sources` | `TEXT[]` | Source types used |
| `depth` | `TEXT` | `quick`, `standard`, `comprehensive` |
| `report` | `TEXT` | Final research report (Markdown) |
| `source_data` | `JSONB` | Raw source data collected |
| `metadata` | `JSONB` | Stats, timing |
| `created_at` | `TIMESTAMPTZ` | Start time |
| `completed_at` | `TIMESTAMPTZ` | End time |

---

### AFL (American Funds Letters)

#### `afl_history`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | AFL ID |
| `user_id` | `UUID` → `auth.users` | Creator |
| `title` | `TEXT` | Letter title |
| `content` | `TEXT` | Generated letter content |
| `fund_name` | `TEXT` | Fund name |
| `settings` | `JSONB` | Generation settings |
| `status` | `TEXT` | Status |
| `created_at` | `TIMESTAMPTZ` | Creation time |

#### `afl_settings`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | Settings ID |
| `user_id` | `UUID` → `auth.users` | User |
| `preset_name` | `TEXT` | Preset name |
| `settings` | `JSONB` | AFL configuration |
| `is_default` | `BOOLEAN` | Default preset |
| `created_at` | `TIMESTAMPTZ` | Creation time |

#### `afl_uploaded_files`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | File ID |
| `user_id` | `UUID` → `auth.users` | Uploader |
| `file_name` | `TEXT` | Filename |
| `file_path` | `TEXT` | Storage path |
| `file_type` | `TEXT` | MIME type |
| `content_text` | `TEXT` | Extracted text |
| `created_at` | `TIMESTAMPTZ` | Upload time |

---

### Backtest

#### `backtest_sessions`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | Session ID |
| `user_id` | `UUID` → `auth.users` | User |
| `strategy` | `TEXT` | Strategy description |
| `tickers` | `TEXT[]` | Tickers tested |
| `start_date` | `DATE` | Backtest start |
| `end_date` | `DATE` | Backtest end |
| `results` | `JSONB` | Performance metrics |
| `analysis` | `TEXT` | AI analysis |
| `status` | `TEXT` | Status |
| `created_at` | `TIMESTAMPTZ` | Run time |

---

### Presentations

#### `presentations`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | Presentation ID |
| `user_id` | `UUID` → `auth.users` | Creator |
| `title` | `TEXT` | Deck title |
| `topic` | `TEXT` | Subject matter |
| `slide_count` | `INTEGER` | Number of slides |
| `file_path` | `TEXT` | Storage path |
| `style` | `TEXT` | Template style |
| `metadata` | `JSONB` | Generation config |
| `created_at` | `TIMESTAMPTZ` | Generation time |

---

### Uploaded Files

#### `uploaded_files`

| Column | Type | Description |
|--------|------|-------------|
| `id` | `UUID` PK | File ID |
| `user_id` | `UUID` → `auth.users` | Uploader |
| `file_name` | `TEXT` | Original name |
| `file_path` | `TEXT` | Storage path |
| `file_type` | `TEXT` | MIME type |
| `file_size` | `BIGINT` | Bytes |
| `content_text` | `TEXT` | Extracted text |
| `analysis` | `TEXT` | AI analysis |
| `created_at` | `TIMESTAMPTZ` | Upload time |

---

## Row Level Security (RLS)

All tables have RLS enabled. Policies follow this pattern:

```sql
-- Users can only see their own data
CREATE POLICY "Users can view own data" ON table_name
  FOR SELECT USING (user_id = auth.uid());

CREATE POLICY "Users can insert own data" ON table_name
  FOR INSERT WITH CHECK (user_id = auth.uid());

CREATE POLICY "Users can update own data" ON table_name
  FOR UPDATE USING (user_id = auth.uid());

CREATE POLICY "Users can delete own data" ON table_name
  FOR DELETE USING (user_id = auth.uid());

-- Service role bypasses all RLS (used by backend)
CREATE POLICY "Service role full access" ON table_name
  FOR ALL USING (auth.role() = 'service_role');
```

**Important:** The backend uses the `service_role` key which bypasses RLS. Direct Supabase client access from frontends would be restricted by RLS.

---

## Storage Buckets

| Bucket | Purpose | Max Size |
|--------|---------|----------|
| `brain-documents` | Knowledge base documents | 50 MB |
| `uploaded-files` | User file uploads | 50 MB |
| `afl-files` | AFL reference documents | 50 MB |
| `presentations` | Generated PPTX files | 50 MB |
| `avatars` | User profile pictures | 5 MB |

---

## Extensions

| Extension | Purpose |
|-----------|---------|
| `pgvector` | Vector similarity search for embeddings |
| `uuid-ossp` | UUID generation |
| `pgcrypto` | (Available but not primary encryption — app uses Fernet) |

---

## Migration History

| # | Name | Description |
|---|------|-------------|
| 001 | `initial_schema` | Core tables: user_profiles, conversations, messages |
| 001 | `training_data` | Training examples table |
| 001 | `incremental_missing` | Gap-fill for missed tables |
| 002 | `feedback_analytics` | Feedback and analytics tables |
| 003 | `researcher_tables` | Research sessions |
| 004 | `history_tables` | History tracking |
| 005 | `afl_uploaded_files` | AFL file uploads |
| 006 | `afl_settings_presets` | AFL settings/presets |
| 007 | `conversation_files` | Files attached to conversations |
| 008 | `missing_tables` | Backtest, presentations, content |
| 009 | `brain_tables_and_embeddings` | Brain documents + pgvector |
| 010 | `supabase_auth_migration` | Switch to Supabase Auth |
| 011 | `fix_foreign_keys` | FK corrections |
| 012 | `clean_slate_auth_fix` | Auth cleanup, revert to app-layer encryption |
| 013 | `security_hardening` | RLS policies, security fixes |
| 014 | `secure_rebuild` | Final security rebuild |

---

## Entity Relationship Diagram

```
auth.users (Supabase Auth)
    │
    ├──── user_profiles (1:1)
    │
    ├──── conversations (1:many)
    │         ├──── messages (1:many)
    │         └──── conversation_files (1:many)
    │
    ├──── brain_documents (1:many)
    │         └──── brain_document_chunks (1:many, with pgvector)
    │
    ├──── skills (1:many)
    │
    ├──── training_examples (1:many)
    │
    ├──── research_sessions (1:many)
    │
    ├──── afl_history (1:many)
    ├──── afl_settings (1:many)
    ├──── afl_uploaded_files (1:many)
    │
    ├──── backtest_sessions (1:many)
    │
    ├──── presentations (1:many)
    │
    ├──── uploaded_files (1:many)
    │
    └──── feedback (1:many)
              └──── messages (many:1)
```

---

*Next: [09 — Tools & Skills](09-TOOLS-AND-SKILLS.md)*
