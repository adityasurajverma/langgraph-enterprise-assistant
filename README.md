# 🤖 Enterprise AI Assistant — `chatbot.py`

The core brain of the LangGraph Enterprise Assistant. This module powers a **ReAct agent** built with LangGraph and LangChain that connects to a MySQL database, a ChromaDB PDF vector store, and an LLM — delivering intelligent, streaming responses over SSE.

---

## 📁 File Overview

```
chatbot.py
├── System Prompt          → Defines agent behaviour and table knowledge
├── SQL Database Layer     → LangChain SQLDatabase singleton
├── Tool Factory           → 6 agent tools built per user session
├── stream_message()       → Primary SSE streaming entry point
└── process_message()      → Synchronous wrapper (backward compatibility)
```

---

## ⚙️ How It Works

```
User Message
     ↓
SystemMessage (prompt + user context)
     ↓
LangGraph ReAct Agent  ←──── get_llm()  (from config.py)
     ↓
  [Loop] Think → Pick Tool → Call Tool → Observe → Repeat
     ↓
Stream tokens / structured data via SSE (NDJSON)
     ↓
Frontend renders text / table / chart / attendance card
```

---

## 🛠️ Tools

The agent has access to **6 tools**, created per user session via `_make_tools()`:

| Tool | Purpose | Notes |
|---|---|---|
| `sql_query` | Run SELECT queries on MySQL | Blocks INSERT / UPDATE / DELETE / DROP |
| `sql_schema` | Inspect table columns and structure | Call before writing complex queries |
| `search_pdf_library` | Semantic search over uploaded PDFs | Uses ChromaDB vector store |
| `mark_attendance` | Check-in / check-out for current user | Writes to `attendance` table |
| `get_attendance_report` | Read attendance history | Admins can view all employees |
| `generate_chart` | Render bar / line / pie / doughnut chart | Call after `sql_query` |

---

## 🗄️ Database Tables Known to the Agent

Defined in the system prompt so the LLM understands schema without extra tool calls:

| Table | Key Columns |
|---|---|
| `products` | name, category, price |
| `sales` | product_id, quantity, amount, sale_date |
| `tstock_movement` | from_role, to_role, status, movement_type, imei, moved_date |
| `tuser_stock` | model_name, imei1, imei2, invoice_no, dbr_name, status, quantity |
| `attendance` | employee_id, date, check_in, check_out, status |
| `employees` | username, name, email, department, role |

---

## 📡 Streaming — `stream_message()`

Primary entry point used by the SSE endpoint. Yields **newline-delimited JSON (NDJSON)** events:

```python
def stream_message(message: str, user: dict | None = None):
    ...
```

### Event Types

| Event | Shape | When |
|---|---|---|
| Status indicator | `{"status": "Running Sql Query…"}` | While a tool is executing |
| Text token | `{"token": "Hello"}` | LLM streaming a reply word by word |
| Done (text) | `{"done": true}` | Plain text response complete |
| Done (structured) | `{"done": true, "data": {...}}` | Chart / attendance / error |

### Structured `data` Payloads

**Chart:**
```json
{
  "done": true,
  "data": {
    "type": "chart",
    "chart_type": "bar",
    "title": "Sales by Category",
    "labels": ["Electronics", "Clothing"],
    "dataset_label": "Revenue",
    "data": [45000.0, 23000.0]
  }
}
```

**Attendance action (check-in / check-out):**
```json
{
  "done": true,
  "data": {
    "type": "attendance",
    "status": "success",
    "message": "Check-in recorded at 09:15 AM"
  }
}
```

**Attendance table:**
```json
{
  "done": true,
  "data": {
    "type": "attendance_table",
    "data": [ { "date": "2025-07-01", "check_in": "09:00", "check_out": "18:00" } ]
  }
}
```

**Error with suggestions:**
```json
{
  "done": true,
  "data": {
    "type": "error",
    "message": "The model failed to generate a valid tool call.",
    "suggestions": ["Show top 5 selling products", "Show my attendance history"]
  }
}
```

---

## 🔁 Synchronous — `process_message()`

Kept for backward compatibility. Internally calls `stream_message()` and collects all events into a single dict:

```python
def process_message(message: str, user: dict | None = None) -> dict:
    ...
```

**Returns one of:**
```python
{"type": "text",             "message": "Full reply text"}
{"type": "chart",            "chart_type": ..., "labels": ..., "data": ...}
{"type": "attendance",       "status": "success", "message": "..."}
{"type": "attendance_table", "data": [...]}
{"type": "error",            "message": "...", "suggestions": [...]}
```

---

## 🔐 User Context & Permissions

Every session receives a `user` dict that scopes tool behaviour:

```python
user = {
    "employee_id": 42,
    "name": "Prashant",
    "role": "admin",          # or "employee"
    "department": "Sales"
}
```

| Behaviour | Employee | Admin |
|---|---|---|
| Mark own attendance | ✅ | ✅ |
| View own attendance | ✅ | ✅ |
| View all attendance | ❌ | ✅ |
| SQL queries | ✅ | ✅ |
| PDF search | ✅ | ✅ |

---

## 🧠 Agent Rules (System Prompt)

The agent is instructed to:

1. Use `sql_query` for all data questions — **SELECT only**
2. Use `ORDER BY … LIMIT N` for rankings
3. Use `MONTH()`, `YEAR()`, `CURDATE()`, `DATE_SUB()` for date filtering
4. Use `search_pdf_library` for policy / document questions and cite the source
5. Use `mark_attendance` / `get_attendance_report` exclusively for attendance (never raw SQL)
6. Call `sql_query` first, then `generate_chart` for visualization requests
7. Be concise

---

## ⚠️ Error Handling

`_friendly_error()` maps raw exceptions to human-readable messages:

| Error | User Message |
|---|---|
| Rate limit (429) | "Rate limit reached… try again in Xm." |
| Failed tool call | "Model failed to generate a valid tool call. Try rephrasing." |
| Quota exhausted | "API credits exhausted. Please top up." |
| Timeout | "Request timed out. Model may be overloaded." |

---

## 🔗 Dependencies

```python
from config      import get_db_uri, get_llm       # LLM + DB URI provider
from database    import mark_attendance, \
                        get_attendance_report      # DB write/read helpers
from pdf_handler import query_pdfs                 # ChromaDB vector search
```

### External packages required

```
langchain
langchain-community
langgraph
pydantic
```

---

## 💡 Example Usage

```python
from chatbot import stream_message, process_message

user = {"employee_id": 1, "name": "Aditya", "role": "admin", "department": "IT"}

# Streaming (SSE)
for event in stream_message("Show top 5 products by sales", user=user):
    print(event)

# Synchronous
result = process_message("Show my attendance this month", user=user)
print(result)
```

---

## 📝 Notes
<img width="1513" height="896" alt="a" src="https://github.com/user-attachments/assets/cc72ad6b-b303-4b1b-ab31-dafe01c8a237" />


- `SQLDatabase` is a **singleton** — initialized once and reused across all requests to avoid repeated connection overhead.
- `sample_rows_in_table_info=0` is set intentionally to **save ~1000 tokens per call** — column info is sufficient.
- The token buffer logic handles **Llama 3's tendency** to leak tool-call syntax as plain text before a tool is invoked — only flushing text tokens when the step is confirmed as a plain reply.
- Chart priority order on `done`: **chart → attendance action → attendance table → plain text**.
