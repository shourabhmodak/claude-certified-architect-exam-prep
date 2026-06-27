# Domain 1: Agentic Architecture & Orchestration

**Exam weight: 27% — largest domain**

→ [Cheat Sheet Index](./README.md) | [Main README](../README.md)

---

## Key Concepts Table

| Concept | What to Know |
|---------|-------------|
| 4-step agentic loop | Execute → Evaluate stop_reason → Call tool → Respond |
| `stop_reason` values | `end_turn`, `tool_use`, `max_tokens`, `stop_sequence` |
| Loop termination guard | Always use `for _ in range(MAX)`, never `while True` |
| Tool result format | `user` turn, `tool_result` content block with `tool_use_id` |
| Message history | Append full `response.content`, not just text blocks |
| Multi-agent patterns | Hub-and-spoke, pipeline, parallel fan-out |
| Subagent isolation | Each subagent gets its own message history; no shared state by default |

---

## Production Agentic Loop — Complete Pattern

```python
import anthropic

client = anthropic.Anthropic()
MAX_ITERATIONS = 15

def run_agent(user_prompt: str, tools: list) -> list:
    messages = [{"role": "user", "content": user_prompt}]

    for iteration in range(MAX_ITERATIONS):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )

        # Append full content (includes tool_use blocks, not just text)
        messages.append({"role": "assistant", "content": response.content})

        match response.stop_reason:
            case "end_turn":
                return response.content

            case "tool_use":
                tool_results = []
                for block in response.content:
                    if block.type != "tool_use":
                        continue
                    result = dispatch_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })
                messages.append({"role": "user", "content": tool_results})

            case "max_tokens":
                raise RuntimeError("Response truncated — increase max_tokens or reduce context")

            case "stop_sequence":
                return response.content  # custom stop sequence hit

    raise RuntimeError(f"Agent cycle detected after {MAX_ITERATIONS} iterations")
```

---

## Multi-Tool Response Handling

Claude can request multiple tools in one response. Always iterate all blocks:

```python
# WRONG — only handles first tool call
tool_block = response.content[0]

# RIGHT — handle all tool_use blocks in one turn
tool_results = []
for block in response.content:
    if block.type == "tool_use":
        result = dispatch_tool(block.name, block.input)
        tool_results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": str(result)
        })
```

---

## Hub-and-Spoke Pattern

Use when: single agent needs 10+ tools, routing failures in production, context saturation.

```
         [ User Input ]
               │
               ▼
    ┌──────────────────┐
    │    Hub Agent     │ ← manages state, intent, routing
    └──────────────────┘
         │    │    │
    ┌────┘    │    └────┐
    ▼         ▼         ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│ Spoke 1 │ │ Spoke 2 │ │ Spoke 3 │
│  CRM    │ │ Refunds │ │Shipping │
│ 3 tools │ │ 4 tools │ │ 3 tools │
└─────────┘ └─────────┘ └─────────┘
```

```python
def hub_agent(user_input: str) -> str:
    # Hub classifies intent and delegates
    intent = classify_intent(user_input)

    match intent:
        case "crm":
            return crm_spoke.run(user_input, session_state)
        case "refund":
            return refund_spoke.run(user_input, session_state)
        case "shipping":
            return shipping_spoke.run(user_input, session_state)

# Each spoke is its own agent with its own tools and message history
```

---

## Parallel Agent Fan-Out

Use when: subtasks are independent (no data dependency between them).

```python
import asyncio

async def parallel_research(topics: list[str]) -> list:
    tasks = [research_agent(topic) for topic in topics]
    results = await asyncio.gather(*tasks)
    return results

# Then aggregate in hub:
async def hub(query: str):
    subtopics = decompose_query(query)                    # deterministic
    results = await parallel_research(subtopics)          # parallel
    return synthesize_agent(query, results)               # sequential
```

---

## Subagent Context Isolation — Exam Trap

Each spawned agent starts with a **clean message history**. No automatic state sharing.

```python
# WRONG — subagent cannot see hub's conversation
subagent_response = client.messages.create(
    model="claude-sonnet-4-6",
    messages=[{"role": "user", "content": "Continue what we were doing"}],
    # subagent has NO context from hub's messages
)

# RIGHT — pass relevant context explicitly
subagent_response = client.messages.create(
    model="claude-sonnet-4-6",
    system=f"Context from hub: {hub_state_summary}",
    messages=[{"role": "user", "content": task_for_subagent}],
)
```

---

## Exam Anti-Patterns

| Anti-Pattern | Why Wrong | Fix |
|-------------|-----------|-----|
| `if "TOOL:" in response.content[0].text` | Text parsing is non-deterministic | `response.stop_reason == "tool_use"` |
| `while True:` agentic loop | No termination guarantee | `for _ in range(MAX_ITERATIONS)` |
| Appending only text from response | Loses tool_use blocks from context | Append full `response.content` |
| Single agent with 15 tools | Context saturation, wrong selections | Hub-and-spoke decomposition |
| Subagent assumes shared state | Each agent context is isolated | Pass state explicitly in system/messages |
| Ignoring `max_tokens` stop_reason | Silent truncation = corrupted state | Handle all 4 stop_reason values |

---

## Quick-Reference: stop_reason Handling

```python
handlers = {
    "end_turn": lambda r: r.content,                    # task complete
    "tool_use": lambda r: process_tool_calls(r),        # call tools, continue loop
    "max_tokens": lambda r: raise_truncation_error(r),  # increase budget or compact
    "stop_sequence": lambda r: r.content,               # custom stop hit
}
result = handlers[response.stop_reason](response)
```

---

👉 **[Get 360 practice questions — Free Enrollment](https://www.udemy.com/course/claude-certified-architect-cca-f-6-practice-tests-2026/?couponCode=FFA33A1350C02189AC15)**

→ Next: [Domain 2 — Tool Design & MCP](./domain2-tool-design-mcp.md)
