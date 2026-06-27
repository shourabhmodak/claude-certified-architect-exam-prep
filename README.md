# Claude Certified Architect – Foundations (CCA-F) Exam Prep

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Udemy](https://img.shields.io/badge/Udemy-6%20Practice%20Tests-EC5252?logo=udemy&logoColor=white)](https://www.udemy.com/course/claude-certified-architect-cca-f-6-practice-tests-2026/?referralCode=D8A4535C0858621CB62B)
[![Stars](https://img.shields.io/github/stars/shourabhmodak/claude-certified-architect-exam-prep?style=social)](https://github.com/shourabhmodak/claude-certified-architect-exam-prep/stargazers)

Free study guide, domain cheat sheets, interactive practice exam, and scenario-based questions for the **Anthropic Claude Certified Architect – Foundations (CCA-F)** certification.

> If this repo saves you prep time, a ⭐ helps others find it and takes 2 seconds.

---

## Exam Format at a Glance

| Detail | Value |
|--------|-------|
| Platform | Skilljar (proctored) |
| Questions | 60 scenario-based |
| Time limit | 90 minutes |
| Passing score | 720 / 1000 (72%) |
| Retry fee | $99 |
| Format | Multiple choice, scenario-driven |
| Focus | Production architecture judgment, not definitions |

---

## 🎁 100 Free Udemy Enrollment Slots

Giving away **100 free enrollments** to the full practice test course in exchange for honest reviews.

**What you get:**
- 6 full-length simulated exams (360 unique scenario-based questions)
- Exact domain weights matching official Anthropic blueprint
- Deep-dive explanations — *why* each option is correct or an anti-pattern
- Direct links to official Anthropic docs and Cookbook samples

👉 **[Claim Free Slot — Coupon Applied at Link](https://www.udemy.com/course/claude-certified-architect-cca-f-6-practice-tests-2026/?couponCode=FFA33A1350C02189AC15)**

*Single-use coupon. First 100 only. Leave an honest review — that's the only ask.*

---

## Table of Contents

1. [Exam Domain Breakdown](#exam-domain-breakdown)
2. [Domain 1: Agentic Architecture & Orchestration (27%)](#domain-1-agentic-architecture--orchestration-27)
3. [Domain 2: Tool Design & MCP Integration (18%)](#domain-2-tool-design--mcp-integration-18)
4. [Domain 3: Claude Code Configuration & Workflows (20%)](#domain-3-claude-code-configuration--workflows-20)
5. [Domain 4: Prompt Engineering & Structured Output (20%)](#domain-4-prompt-engineering--structured-output-20)
6. [Domain 5: Context Management & Reliability (15%)](#domain-5-context-management--reliability-15)
7. [Exam Anti-Patterns Cheatsheet](#exam-anti-patterns-cheatsheet)
8. [Sample Exam Questions](#sample-exam-questions)
9. [Interactive Practice Exam](#interactive-practice-exam)
10. [One-Page Quick Reference](#one-page-quick-reference)
11. [Course Curriculum](#course-curriculum)

---

## Exam Domain Breakdown

| # | Domain | Weight | Cheat Sheet |
|---|--------|--------|-------------|
| 1 | Agentic Architecture & Orchestration | **27%** | [→ domain1](./cheat-sheet/domain1-agentic-architecture.md) |
| 2 | Tool Design & MCP Integration | **18%** | [→ domain2](./cheat-sheet/domain2-tool-design-mcp.md) |
| 3 | Claude Code Configuration & Workflows | **20%** | [→ domain3](./cheat-sheet/domain3-claude-code-config.md) |
| 4 | Prompt Engineering & Structured Output | **20%** | [→ domain4](./cheat-sheet/domain4-prompt-engineering.md) |
| 5 | Context Management & Reliability | **15%** | [→ domain5](./cheat-sheet/domain5-context-management.md) |

→ **[Full Cheat Sheet Index](./cheat-sheet/README.md)**

---

## Domain 1: Agentic Architecture & Orchestration (27%)

Largest exam domain. Tests ability to design **deterministic code loops** around probabilistic LLM responses.

### The 4-Step Agentic Loop

1. **Execute** — send user request + system context to Claude
2. **Evaluate** — parse `stop_reason` (structural metadata, not model text)
3. **Call Tool** — if `stop_reason == "tool_use"`, execute deterministically
4. **Respond** — append tool result to messages and loop

### Exam Trap: Never Parse Model Text for Flow Control

```python
import anthropic

client = anthropic.Anthropic()

def run_agentic_loop(user_prompt: str, tools_schema: list) -> list:
    messages = [{"role": "user", "content": user_prompt}]
    MAX_ITERATIONS = 10

    for _ in range(MAX_ITERATIONS):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            tools=tools_schema,
            messages=messages
        )

        # Always append full content block — not just text
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason == "end_turn":
            return response.content

        elif response.stop_reason == "tool_use":
            tool_block = next(b for b in response.content if b.type == "tool_use")
            tool_result = execute_local_tool(tool_block.name, tool_block.input)

            messages.append({
                "role": "user",
                "content": [{
                    "type": "tool_result",
                    "tool_use_id": tool_block.id,
                    "content": tool_result
                }]
            })

    raise RuntimeError("Agent exceeded max iterations — possible cycle detected")
```

**Key exam points:**
- `stop_reason` values: `end_turn`, `tool_use`, `max_tokens`, `stop_sequence`
- Always include max iteration guard — `while True` without cap is an exam anti-pattern
- Tool results go in `user` turn as `tool_result` content blocks, not assistant turn
- Append full `response.content` (includes tool_use blocks), not just text

→ [Full Domain 1 Cheat Sheet](./cheat-sheet/domain1-agentic-architecture.md)

---

## Domain 2: Tool Design & MCP Integration (18%)

### Strict Schema Requirements

```python
tools = [{
    "name": "get_customer_order",
    "description": "Retrieves a customer order by ID. Call when the user references a specific order number or asks about order status.",
    "input_schema": {
        "type": "object",
        "properties": {
            "order_id": {
                "type": "string",
                "description": "Alphanumeric order ID, e.g. ORD-12345"
            },
            "include_history": {
                "type": "boolean",
                "description": "Set true to include full status history timeline"
            }
        },
        "required": ["order_id"]
    }
}]
```

### Hooks vs Prompts — Core Exam Concept

| Concern | Wrong approach | Correct approach |
|---------|---------------|-----------------|
| Input validation | System prompt instruction | PreToolCall hook |
| Security checks | "Never access X" in prompt | PreToolCall hook + schema |
| Output scrubbing | "Remove PII before responding" | PostToolCall hook |
| Audit logging | System prompt reminder | PostToolCall hook |

**Rule:** Anything that must be guaranteed goes in hooks (deterministic). Anything advisory goes in prompts (probabilistic).

→ [Full Domain 2 Cheat Sheet](./cheat-sheet/domain2-tool-design-mcp.md)

---

## Domain 3: Claude Code Configuration & Workflows (20%)

### CLAUDE.md — Project Context File

Loaded automatically by Claude Code for every session in that directory:

```markdown
# Project: Payment API

## Commands
- Build: `npm run build`
- Test: `npm test`
- Lint: `npm run lint`

## Architecture
- Services in `/src/services/` — each owns its DB table
- Never commit directly to main — always branch + PR

## Key Constraints
- All payment operations require dual approval in staging
- PII must be masked in logs — use `maskPii()` from `/src/utils/privacy.ts`
```

### Hook Types (settings.json)

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{"type": "command", "command": "bash /scripts/validate-cmd.sh"}]
    }],
    "PostToolUse": [{
      "matcher": "Write",
      "hooks": [{"type": "command", "command": "npm run lint --fix"}]
    }],
    "UserPromptSubmit": [{
      "hooks": [{"type": "command", "command": "bash /scripts/log-prompt.sh"}]
    }],
    "Stop": [{
      "hooks": [{"type": "command", "command": "bash /scripts/notify-done.sh"}]
    }]
  }
}
```

→ [Full Domain 3 Cheat Sheet](./cheat-sheet/domain3-claude-code-config.md)

---

## Domain 4: Prompt Engineering & Structured Output (20%)

### System Prompt Architecture

```python
system_prompt = """
You are a senior code reviewer for Acme Corp's Python codebase.

<rules>
- Flag security issues as CRITICAL, performance issues as MEDIUM, style issues as LOW
- Never suggest changes outside the diff provided
- Always include the line number with each finding
</rules>

<output_format>
Respond with a JSON array. Each item: {"severity": "CRITICAL|MEDIUM|LOW", "line": int, "issue": str, "fix": str}
</output_format>
"""
```

### Structured Output — Use Tools, Not Prompts

```python
# WRONG: asking Claude to "output JSON"
# Unreliable — Claude may add prose, wrap in markdown, etc.

# RIGHT: use a tool definition to enforce schema
extraction_tool = [{
    "name": "extract_entities",
    "description": "Extract named entities from the text",
    "input_schema": {
        "type": "object",
        "properties": {
            "people": {"type": "array", "items": {"type": "string"}},
            "organizations": {"type": "array", "items": {"type": "string"}},
            "locations": {"type": "array", "items": {"type": "string"}}
        },
        "required": ["people", "organizations", "locations"]
    }
}]
```

→ [Full Domain 4 Cheat Sheet](./cheat-sheet/domain4-prompt-engineering.md)

---

## Domain 5: Context Management & Reliability (15%)

### Compaction Pattern for Long-Running Agents

```python
COMPACTION_THRESHOLD = 20  # messages before compacting

def compact_if_needed(messages: list, client) -> list:
    if len(messages) < COMPACTION_THRESHOLD:
        return messages

    to_compact = messages[:-5]  # keep last 5 turns fresh
    summary_response = client.messages.create(
        model="claude-haiku-4-5-20251001",  # cheap model for summarization
        max_tokens=1024,
        system="Summarize this conversation history as a compact state block. Preserve all decisions, tool results, and key findings.",
        messages=to_compact
    )

    return [
        {"role": "user", "content": f"[Compacted history]: {summary_response.content[0].text}"},
        {"role": "assistant", "content": "Understood. Continuing from that state."},
        *messages[-5:]
    ]
```

### Prompt Caching — High-Volume Cost Reduction

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": large_static_document,  # 50k tokens — cache this
            "cache_control": {"type": "ephemeral"}
        },
        {
            "type": "text",
            "text": "You are a document analyst. Answer questions about the document above."
        }
    ],
    messages=[{"role": "user", "content": user_question}]
)
# Cache hit = ~10% of input token cost. 5-minute TTL.
```

→ [Full Domain 5 Cheat Sheet](./cheat-sheet/domain5-context-management.md)

---

## Exam Anti-Patterns Cheatsheet

| Anti-Pattern | Why It Fails | Correct Fix |
|-------------|-------------|-------------|
| String-match Claude output to detect tool calls | Non-deterministic; exact phrase varies | Parse `response.stop_reason == "tool_use"` |
| `while True` loop with no iteration cap | Tool returning ambiguous data causes infinite cycle | Add `for _ in range(MAX_ITERATIONS)` guard |
| "Please output valid JSON" in system prompt | Prompt enforcement is probabilistic | Use tool definition schema; validate with Pydantic |
| Security validation inside system prompt | Advisory only — can be overridden | Move to PreToolCall hook |
| Passing raw 50k-row logs into context | "Lost in the middle" degradation | Preprocess; use compaction pipeline |
| Single mega-agent with 15+ tools | Context saturation, wrong tool selection | Hub-and-spoke pattern |
| Hardcoded `claude-3-5-sonnet-20241022` | Breaks on model deprecation | Use `-latest` aliases or parameterize |
| Putting all instructions in system prompt | Ignores hooks for guaranteed enforcement | Split: advisory → prompt, guaranteed → hook |

---

## Sample Exam Questions

10 full scenario-based questions with answers and explanations: **[→ sample-questions.md](./sample-questions.md)**

Two preview questions included below.

---

**Q: Flow Control (Domain 1)**

An engineer's agentic loop uses this flow control:

```python
text = " ".join(b.text for b in response.content if hasattr(b, "text"))
if "I'll now call the tool" in text:
    execute_tool(...)
```

Root cause and fix?

<details>
<summary>Answer</summary>

**Parse `response.stop_reason` instead of scanning text.**

`stop_reason == "tool_use"` is deterministic API metadata. The model's phrasing varies — it may not produce "I'll now call the tool" every time, causing missed or false-positive tool calls.

```python
if response.stop_reason == "tool_use":
    tool_block = next(b for b in response.content if b.type == "tool_use")
```

</details>

---

**Q: Hooks vs Prompts (Domain 2 & 3)**

A fintech's agentic pipeline has the rule "always run fraud check before payment approval" in the system prompt. In production, the agent occasionally approves payments without completing the fraud check under high token load. Fix?

<details>
<summary>Answer</summary>

**Move enforcement to a PreToolCall hook.**

System prompt rules are probabilistic — under high context load, the model can violate them. A PreToolCall hook is deterministic code that runs before the tool executes:

```python
def pre_tool_hook(tool_name: str, session: dict) -> None:
    if tool_name == "payment_approve":
        if not session.get("fraud_check_passed"):
            raise SecurityViolation("Payment blocked: fraud check not completed")
```

This cannot be bypassed by the model regardless of context state.

</details>

---

## Interactive Exam Simulator

Self-contained browser quiz — 20 questions, instant feedback, domain-score breakdown.

**[→ Open cca-f-simulator.html](./cca-f-simulator.html)**

*(Download and open in any browser — no server needed)*

---

## One-Page Quick Reference

All 5 domains compressed into a single cram sheet: **[→ quick-reference.md](./quick-reference.md)**

---

## Course Curriculum

### Module 1: Agentic Loop Mastery
- 4-step loop architecture and code
- `stop_reason` parsing — all 4 values
- Stateful vs stateless design
- Loop termination and cycle prevention
- Multi-turn conversation state

### Module 2: Tool Design & MCP Integration
- JSON schema — types, descriptions, required fields
- Tool selection accuracy — why descriptions matter more than names
- MCP server setup and client integration
- PreToolCall / PostToolCall hook implementation
- Tool result formatting and error propagation

### Module 3: Claude Code Configuration
- CLAUDE.md structure and loading behavior
- settings.json — hooks, permissions, env vars
- All 4 hook types — when each fires
- Path prefix configuration for tool restrictions
- Custom slash commands / skills

### Module 4: Prompt Engineering & Structured Output
- System prompt architecture for production
- XML tags for structural separation
- Explicit categorical criteria vs vague instructions
- Structured output via tool definitions (not "output JSON")
- Few-shot examples and when to use them

### Module 5: Context Management & Reliability
- Token budgeting strategies
- Compaction pipeline for long agents
- Prompt caching — setup and cost math
- Streaming vs batch response handling
- Circuit breakers and retry patterns

### Module 6: Enterprise Integration
- Authentication in agentic systems
- Rate limiting and backoff
- Observability — tracing agentic loops
- Cost optimization — caching + model routing
- Human-in-the-loop integration

### 6 Full-Length Practice Exams
- Exams 1–4: Domain-focused
- Exam 5: Mixed timed simulation
- Exam 6: Advanced scenario marathon

---

## Contributing

Errors, outdated content, new questions? PRs welcome. Open an issue for significant changes.

---

*Independent study resource. Not affiliated with or endorsed by Anthropic.*
