# 09 — Tools & Skills

> ByPotomac SDK · AI Tool Definitions & Skill System

---

## Overview

The ByPotomac AI engine gives Claude access to **tools** (functions it can call during conversations) and **skills** (specialized system prompt modules). Together, they extend Claude's capabilities beyond text generation.

```
┌─────────────────────────────────────────────────────────────┐
│                    AI Capabilities                           │
│                                                             │
│  ┌───────────────────────┐  ┌───────────────────────────┐  │
│  │      TOOLS            │  │        SKILLS             │  │
│  │                       │  │                           │  │
│  │  Claude can CALL      │  │  Injected into system     │  │
│  │  these functions      │  │  prompt to shape          │  │
│  │  during a response    │  │  Claude's behavior        │  │
│  │                       │  │                           │  │
│  │  • Real-time data     │  │  • Domain expertise       │  │
│  │  • External APIs      │  │  • Output formatting      │  │
│  │  • Document search    │  │  • Specialized knowledge  │  │
│  │  • Calculations       │  │  • Compliance rules       │  │
│  └───────────────────────┘  └───────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Part 1: Tools

### How Tools Work

1. Tools are registered with Claude as part of the API call
2. Claude decides when to use a tool based on the user's request
3. Claude generates a `tool_use` block with the tool name and arguments
4. The backend executes the tool function
5. The result is sent back to Claude as a `tool_result`
6. Claude incorporates the result into its response

### Tool Definition Format (Anthropic)

Each tool is defined as:

```python
{
    "name": "tool_name",
    "description": "What the tool does and when to use it",
    "input_schema": {
        "type": "object",
        "properties": {
            "param1": {"type": "string", "description": "..."},
            "param2": {"type": "integer", "description": "..."}
        },
        "required": ["param1"]
    }
}
```

### Available Tools

#### 1. `get_stock_quote`

Get real-time stock price and market data.

```python
{
    "name": "get_stock_quote",
    "description": "Get current stock quote including price, change, volume, market cap, P/E ratio",
    "input_schema": {
        "properties": {
            "ticker": {"type": "string", "description": "Stock ticker symbol (e.g., AAPL)"}
        },
        "required": ["ticker"]
    }
}
```

**Returns:** `{ price, change, change_percent, volume, market_cap, pe_ratio, high_52w, low_52w }`

#### 2. `get_stock_history`

Get historical price data for charting and analysis.

```python
{
    "name": "get_stock_history",
    "description": "Get historical stock price data",
    "input_schema": {
        "properties": {
            "ticker": {"type": "string", "description": "Stock ticker"},
            "period": {"type": "string", "description": "Time period: 1d, 5d, 1mo, 3mo, 6mo, 1y, 2y, 5y, max"},
            "interval": {"type": "string", "description": "Data interval: 1m, 5m, 15m, 1h, 1d, 1wk, 1mo"}
        },
        "required": ["ticker"]
    }
}
```

**Returns:** Array of `{ date, open, high, low, close, volume }`

#### 3. `get_company_info`

Get detailed company information.

```python
{
    "name": "get_company_info",
    "description": "Get company profile including sector, industry, description, officers",
    "input_schema": {
        "properties": {
            "ticker": {"type": "string", "description": "Stock ticker"}
        },
        "required": ["ticker"]
    }
}
```

**Returns:** `{ name, sector, industry, description, market_cap, employees, website, officers[] }`

#### 4. `get_financial_statements`

Get company financial statements (income, balance sheet, cash flow).

```python
{
    "name": "get_financial_statements",
    "description": "Get financial statements for a company",
    "input_schema": {
        "properties": {
            "ticker": {"type": "string", "description": "Stock ticker"},
            "statement_type": {"type": "string", "enum": ["income", "balance", "cashflow"]},
            "period": {"type": "string", "enum": ["annual", "quarterly"]}
        },
        "required": ["ticker", "statement_type"]
    }
}
```

#### 5. `search_sec_filings`

Search SEC EDGAR for company filings.

```python
{
    "name": "search_sec_filings",
    "description": "Search SEC EDGAR for filings (10-K, 10-Q, 8-K, etc.)",
    "input_schema": {
        "properties": {
            "query": {"type": "string", "description": "Company name or ticker"},
            "filing_type": {"type": "string", "description": "Filing type: 10-K, 10-Q, 8-K, DEF 14A, etc."},
            "limit": {"type": "integer", "description": "Max results (default 10)"}
        },
        "required": ["query"]
    }
}
```

**Returns:** Array of `{ company, cik, filing_type, date, accession_number, url }`

#### 6. `get_sec_filing`

Get the full text of a specific SEC filing.

```python
{
    "name": "get_sec_filing",
    "description": "Retrieve and parse a specific SEC filing by accession number",
    "input_schema": {
        "properties": {
            "accession_number": {"type": "string", "description": "SEC accession number"}
        },
        "required": ["accession_number"]
    }
}
```

#### 7. `web_search`

Search the web using Tavily search API.

```python
{
    "name": "web_search",
    "description": "Search the web for current information, news, and data",
    "input_schema": {
        "properties": {
            "query": {"type": "string", "description": "Search query"},
            "max_results": {"type": "integer", "description": "Max results (default 5)"}
        },
        "required": ["query"]
    }
}
```

**Returns:** Array of `{ title, url, content, score }`

#### 8. `search_news`

Search financial news.

```python
{
    "name": "search_news",
    "description": "Search recent financial news articles",
    "input_schema": {
        "properties": {
            "query": {"type": "string", "description": "News search query"},
            "days_back": {"type": "integer", "description": "Days to look back (default 7)"}
        },
        "required": ["query"]
    }
}
```

#### 9. `query_knowledge_base`

Search the user's uploaded knowledge base documents.

```python
{
    "name": "query_knowledge_base",
    "description": "Search the user's personal knowledge base for relevant information",
    "input_schema": {
        "properties": {
            "query": {"type": "string", "description": "What to search for"},
            "top_k": {"type": "integer", "description": "Number of results (default 5)"}
        },
        "required": ["query"]
    }
}
```

**Returns:** Array of `{ chunk_text, similarity, document_title, document_id }`

#### 10. `calculate`

Perform mathematical calculations.

```python
{
    "name": "calculate",
    "description": "Evaluate mathematical expressions and financial calculations",
    "input_schema": {
        "properties": {
            "expression": {"type": "string", "description": "Math expression to evaluate"}
        },
        "required": ["expression"]
    }
}
```

#### 11. `get_economic_data`

Get macroeconomic data from FRED (Federal Reserve).

```python
{
    "name": "get_economic_data",
    "description": "Get economic indicators from FRED (GDP, CPI, unemployment, fed funds rate)",
    "input_schema": {
        "properties": {
            "series_id": {"type": "string", "description": "FRED series ID (e.g., GDP, CPIAUCSL, UNRATE, FEDFUNDS)"},
            "limit": {"type": "integer", "description": "Number of data points (default 12)"}
        },
        "required": ["series_id"]
    }
}
```

### Tool Execution Flow

```python
# core/claude_engine_wrapper.py (simplified)

async def execute_tool(self, tool_name: str, args: dict, user_id: str) -> dict:
    """Execute a tool and return the result."""
    
    if tool_name == "get_stock_quote":
        return await yfinance_client.get_quote(args["ticker"])
    
    elif tool_name == "search_sec_filings":
        return await edgar_client.search(
            query=args["query"],
            filing_type=args.get("filing_type"),
            limit=args.get("limit", 10)
        )
    
    elif tool_name == "web_search":
        return await tavily_client.search(
            query=args["query"],
            max_results=args.get("max_results", 5)
        )
    
    elif tool_name == "query_knowledge_base":
        return await knowledge_base.query(
            text=args["query"],
            user_id=user_id,
            top_k=args.get("top_k", 5)
        )
    
    # ... etc
```

### Enabling/Disabling Tools

Tools can be toggled per-conversation via the `tools_enabled` field:

```json
// POST /chat/stream
{
  "messages": [...],
  "tools_enabled": true   // false to disable all tools
}
```

---

## Part 2: Skills

### What Are Skills?

Skills are **user-created system prompt modules** stored in Supabase. When activated for a conversation, their prompt text is appended to Claude's system prompt to shape its behavior.

### Skill Schema

```json
{
  "id": "uuid",
  "name": "SEC Filing Expert",
  "description": "Specialized analysis of SEC filings with regulatory context",
  "system_prompt": "You are an expert in SEC filings analysis. When analyzing filings:\n1. Always identify the filing type and its regulatory purpose\n2. Extract key financial metrics\n3. Note any risk factors or material changes\n4. Compare with prior period filings when relevant\n5. Use proper SEC terminology",
  "category": "finance",
  "is_active": true,
  "is_global": false,
  "usage_count": 42
}
```

### How Skills Are Applied

```python
# core/claude_engine_wrapper.py (simplified)

async def build_system_prompt(self, user_id: str, skill_ids: list[str]) -> str:
    # 1. Start with base prompt
    prompt = BASE_SYSTEM_PROMPT
    
    # 2. Load active skills
    if skill_ids:
        skills = await load_skills(skill_ids)
        for skill in skills:
            prompt += f"\n\n[Skill: {skill['name']}]\n{skill['system_prompt']}"
    
    # 3. Load training examples
    examples = await load_training_examples(user_id)
    if examples:
        prompt += "\n\n[Training Examples]\n"
        for ex in examples:
            prompt += f"Q: {ex['input']}\nA: {ex['output']}\n\n"
    
    return prompt
```

### Skill API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/skills` | List all skills (user's + global) |
| `POST` | `/skills` | Create a new skill |
| `GET` | `/skills/{id}` | Get skill details |
| `PUT` | `/skills/{id}` | Update a skill |
| `DELETE` | `/skills/{id}` | Delete a skill |
| `POST` | `/skills/{id}/toggle` | Enable/disable |

### Using Skills in Chat

When starting a chat, the client sends skill IDs:

```json
// POST /chat/stream
{
  "messages": [...],
  "skills": ["uuid-skill-1", "uuid-skill-2"]
}
```

### Pre-Built Skill Examples

#### Potomac Brand Skill

```markdown
# Potomac Brand Voice
When writing content for Potomac Fund Management:
- Professional yet approachable tone
- Data-driven claims with citations
- Focus on risk-adjusted returns
- Use Potomac color scheme: Black + Yellow (#FEC00F)
- Always mention regulatory disclaimers
```

#### DOCX Report Skill

```markdown
# Report Generator
When generating financial reports:
- Use structured headings (Executive Summary, Analysis, Recommendations)
- Include data tables for financial metrics
- Add charts where appropriate
- Follow Potomac template formatting
- Include compliance disclaimers
```

#### PPTX Presentation Skill

```markdown
# Presentation Builder
When creating presentations:
- Use bull/bear slide layouts
- Potomac branding (logo, colors, fonts)
- Maximum 6 bullet points per slide
- Include speaker notes
- End with disclaimers slide
```

---

## Skill Gateway (`core/skill_gateway.py`)

The skill gateway routes requests to appropriate skill handlers:

```python
class SkillGateway:
    async def process_with_skills(
        self,
        message: str,
        skills: list[dict],
        user_id: str
    ) -> dict:
        """
        1. Check if any skill has special handling (e.g., DOCX gen)
        2. Route to specialized handler if needed
        3. Otherwise, inject skill prompts into system prompt
        """
```

Some skills trigger specialized pipelines:
- **DOCX generation** → `core/docx_generator.py`
- **PPTX generation** → `pptx_engine/`
- **AFL generation** → `core/prompts/afl.py`

---

## Training Data Integration

Training examples are loaded alongside skills:

```python
# core/training.py
class TrainingManager:
    async def get_active_examples(self, user_id: str) -> list[dict]:
        """Load active training examples for a user."""
        result = supabase.table("training_examples") \
            .select("input, output, category") \
            .eq("user_id", user_id) \
            .eq("is_active", True) \
            .execute()
        return result.data
```

These are injected as few-shot examples in the system prompt.

---

## Adding a New Tool

To add a new tool to the system:

1. **Define the tool** in `core/tools.py`:
   ```python
   NEW_TOOL = {
       "name": "my_new_tool",
       "description": "What it does",
       "input_schema": { ... }
   }
   ```

2. **Implement the executor** in `core/claude_engine_wrapper.py`:
   ```python
   elif tool_name == "my_new_tool":
       return await my_service.execute(args)
   ```

3. **Register it** in the tools list passed to Claude.

---

*Next: [10 — Deployment](10-DEPLOYMENT.md)*
