# Domain 2: Tool Design & MCP Integration

**Exam weight: 18%**

→ [Cheat Sheet Index](./README.md) | [Main README](../README.md)

---

## Key Concepts Table

| Concept | What to Know |
|---------|-------------|
| Tool description | Primary routing signal — more important than tool name |
| `required` array | Missing = model treats all params as optional |
| PreToolCall hook | Deterministic enforcement: auth, validation, rate limiting |
| PostToolCall hook | Deterministic output: scrubbing, logging, audit trail |
| MCP server | Exposes tools over a protocol; Claude Code connects as client |
| Tool result format | String or array of content blocks |
| Error propagation | Return error as tool_result content, not by raising exception |

---

## Tool Schema — Complete Pattern

```python
tools = [
    {
        "name": "query_customer_database",
        "description": (
            "Retrieves customer account information by customer ID or email. "
            "Use when the user asks about account details, subscription status, "
            "billing history, or contact information for a specific customer. "
            "Do NOT use for bulk queries — this fetches one record at a time."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "lookup_type": {
                    "type": "string",
                    "enum": ["customer_id", "email"],
                    "description": "Whether to look up by customer ID or email address"
                },
                "lookup_value": {
                    "type": "string",
                    "description": "The customer ID (e.g. CUST-12345) or email address to look up"
                },
                "fields": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "Optional list of specific fields to return. Omit for all fields.",
                    "default": []
                }
            },
            "required": ["lookup_type", "lookup_value"]
        }
    }
]
```

**Schema rules:**
- `description` at tool level = when to call this tool (routing)
- `description` at property level = what to put in each field (format hints)
- Always set `required` — omitting it means the model can skip any field
- Use `enum` for constrained string values — prevents hallucinated values
- Use concrete examples in descriptions: `e.g. ORD-12345`

---

## Hooks vs Prompts — The Core Distinction

```
Probabilistic ──────────────────── Deterministic
     │                                    │
System Prompt                      Code Hooks
 "Always check auth"          PreToolCall validates token
 "Never log PII"              PostToolCall strips PII
 "Run fraud check first"      Hook enforces sequencing
```

**Exam rule:** If it *must* happen (security, compliance, ordering), use a hook. If it *should* happen (tone, style, scope), use a prompt.

---

## PreToolCall Hook Implementation

```python
from functools import wraps

SENSITIVE_TOOLS = {"payment_approve", "user_delete", "admin_escalate"}

def pre_tool_call_hook(tool_name: str, tool_input: dict, session: dict) -> dict:
    """
    Returns tool_input (possibly modified) or raises to block the call.
    """
    # 1. Authentication
    if not session.get("authenticated"):
        raise PermissionError(f"Unauthenticated call to {tool_name}")

    # 2. Authorization
    if tool_name in SENSITIVE_TOOLS:
        required_role = TOOL_ROLE_MAP[tool_name]
        if session["role"] not in required_role:
            raise PermissionError(f"Role {session['role']} cannot call {tool_name}")

    # 3. Input sanitization
    if "query" in tool_input:
        tool_input["query"] = sanitize_sql(tool_input["query"])

    # 4. Rate limiting
    check_rate_limit(session["user_id"], tool_name)

    return tool_input  # return (possibly modified) input to proceed


def post_tool_call_hook(tool_name: str, result: str, session: dict) -> str:
    """
    Returns (possibly modified) result or raises to block delivery.
    """
    # Strip PII before result enters context window
    result = mask_pii(result)

    # Audit log every sensitive tool call
    if tool_name in SENSITIVE_TOOLS:
        audit_log(session["user_id"], tool_name, result)

    return result
```

---

## Tool Result Formatting

```python
def dispatch_tool(tool_name: str, tool_input: dict) -> str:
    try:
        result = TOOL_REGISTRY[tool_name](**tool_input)
        return str(result)  # always return string

    except KeyError:
        # Return error as content — don't raise; let the model handle it
        return f"Error: tool '{tool_name}' not found"

    except Exception as e:
        return f"Error executing {tool_name}: {str(e)}"

# Feed result back as tool_result content block:
messages.append({
    "role": "user",
    "content": [{
        "type": "tool_result",
        "tool_use_id": tool_block.id,
        "content": dispatch_tool(tool_block.name, tool_block.input),
        "is_error": False  # set True if result represents an error
    }]
})
```

---

## MCP Server — Minimal Setup

```python
# server.py — MCP server exposing tools to Claude Code
from mcp.server import Server
from mcp.server.stdio import stdio_server
import mcp.types as types

server = Server("my-tools")

@server.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="get_weather",
            description="Get current weather for a city. Use when user asks about weather.",
            inputSchema={
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "City name, e.g. 'London'"},
                    "units": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "Temperature unit"
                    }
                },
                "required": ["city"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name == "get_weather":
        result = fetch_weather(arguments["city"], arguments.get("units", "celsius"))
        return [types.TextContent(type="text", text=str(result))]

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream, server.create_initialization_options())
```

**Claude Code settings.json — connect to MCP server:**
```json
{
  "mcpServers": {
    "my-tools": {
      "command": "python",
      "args": ["server.py"],
      "cwd": "/path/to/server"
    }
  }
}
```

---

## Exam Anti-Patterns

| Anti-Pattern | Why Wrong | Fix |
|-------------|-----------|-----|
| Tool description: `"Searches the database"` | Too vague — model can't decide when to call it | Add when-to-use and when-NOT-to-use |
| No `required` array in schema | Model treats all params as optional | Always set `required` explicitly |
| Security check in system prompt | Probabilistic — can be skipped | PreToolCall hook |
| Raising exception on tool error | Crashes the loop | Return error string as tool_result content |
| One giant tool for all DB operations | Poor routing accuracy | Split by operation type and entity |
| `"content": json.dumps(result)` always | JSON string works, but can be noisy | Return clean string representation |

---

👉 **[Get 360 practice questions — Free Enrollment](https://www.udemy.com/course/claude-certified-architect-cca-f-6-practice-tests-2026/?couponCode=FFA33A1350C02189AC15)**

← [Domain 1](./domain1-agentic-architecture.md) | → [Domain 3](./domain3-claude-code-config.md)
