# Domain 5: Context Management & Reliability

**Exam weight: 15%**

→ [Cheat Sheet Index](./README.md) | [Main README](../README.md)

---

## Key Concepts Table

| Concept | What to Know |
|---------|-------------|
| Context window | Total token budget shared by system + messages + tools + response |
| "Lost in the middle" | Model attention degrades for content in the middle of long context |
| Compaction | Summarize old turns to free up token budget without losing state |
| Prompt caching | Mark static content with `cache_control`; cache hits ~10% of full cost |
| Cache TTL | 5 minutes (ephemeral cache); refreshed on each cache hit |
| Token counting | Use `client.messages.count_tokens()` before sending |
| Streaming | Use for long responses to improve perceived latency |

---

## Token Budget Planning

```python
import anthropic

client = anthropic.Anthropic()

# Models and their context windows (as of mid-2026)
CONTEXT_LIMITS = {
    "claude-opus-4-8":        200_000,
    "claude-sonnet-4-6":      200_000,
    "claude-haiku-4-5-20251001": 200_000,
}

def estimate_available_output(system: str, messages: list, tools: list, model: str) -> int:
    token_count = client.messages.count_tokens(
        model=model,
        system=system,
        tools=tools,
        messages=messages
    )
    limit = CONTEXT_LIMITS[model]
    input_tokens = token_count.input_tokens
    safe_max_output = min(limit - input_tokens - 500, 8192)  # 500 token safety margin
    return safe_max_output
```

---

## Compaction Pattern

For agentic loops running 20+ iterations — prevents "lost in the middle" degradation:

```python
COMPACTION_THRESHOLD = 20  # compact when message count exceeds this
KEEP_RECENT = 6            # always keep last N turns verbatim

def compact_if_needed(messages: list) -> list:
    if len(messages) < COMPACTION_THRESHOLD:
        return messages

    to_compact = messages[:-KEEP_RECENT]
    recent = messages[-KEEP_RECENT:]

    # Use cheap model for summarization
    summary_response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=2048,
        system=(
            "You are summarizing an AI agent's work session. "
            "Produce a dense state summary: decisions made, tool results, "
            "key findings, current task state. Preserve all factual data. "
            "Use bullet points. Be comprehensive but concise."
        ),
        messages=to_compact
    )

    summary_text = summary_response.content[0].text

    return [
        {
            "role": "user",
            "content": f"[SESSION SUMMARY — prior conversation compacted]\n{summary_text}"
        },
        {
            "role": "assistant",
            "content": "Understood. I have the session context and will continue from this state."
        },
        *recent
    ]
```

**When to compact:**
- Long-running research or coding agents (20+ turns)
- Any agent that processes large documents repeatedly
- Before making expensive API calls on nearly-full context

---

## Prompt Caching

Reduces cost by ~90% on repeated identical prefixes. Critical for RAG systems and high-volume agents.

```python
# Static content (documents, system rules) — cache it
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": large_legal_document,       # 80k tokens — expensive per call
            "cache_control": {"type": "ephemeral"}  # cache this block
        },
        {
            "type": "text",
            "text": "You are a legal analyst. Answer questions about the document above."
            # No cache_control — this is short, not worth caching alone
        }
    ],
    messages=[{"role": "user", "content": user_question}]
)

# Inspect cache usage
print(response.usage.cache_read_input_tokens)   # tokens served from cache
print(response.usage.cache_creation_input_tokens)  # tokens written to cache
```

**Caching rules:**
- Cache block must be at least ~1,024 tokens to be eligible
- 5-minute TTL — each hit resets the timer
- Cache is per-account, per-model, per-exact-content
- Cost: write = 1.25x normal; read = 0.1x normal

**ROI calculation:**
```
normal_cost = input_tokens * per_token_price
cache_write_cost = input_tokens * per_token_price * 1.25  # first call
cache_hit_cost = input_tokens * per_token_price * 0.10   # subsequent calls

# Break even after 2nd call. Huge win at volume.
```

---

## Streaming Responses

Use for long generations to improve perceived latency:

```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    messages=[{"role": "user", "content": "Write a detailed architecture doc for..."}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)  # stream to user in real-time

# Get final message after stream completes
final_message = stream.get_final_message()
```

**Use streaming when:**
- Response is long (>500 tokens) and user is waiting
- Displaying output progressively (chat UI, terminal)
- You want to cancel early if you detect the response is going wrong

**Don't use streaming when:**
- Processing response programmatically (wait for complete response)
- Tool use agentic loops (need complete response to parse stop_reason)

---

## Circuit Breaker Pattern for Reliability

```python
import time
from dataclasses import dataclass, field

@dataclass
class CircuitBreaker:
    failure_threshold: int = 5
    reset_timeout: int = 60  # seconds
    failures: int = field(default=0, init=False)
    last_failure_time: float = field(default=0, init=False)
    state: str = field(default="closed", init=False)  # closed, open, half-open

    def call(self, fn, *args, **kwargs):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.reset_timeout:
                self.state = "half-open"
            else:
                raise RuntimeError("Circuit open — API calls suspended")

        try:
            result = fn(*args, **kwargs)
            if self.state == "half-open":
                self.state = "closed"
                self.failures = 0
            return result

        except Exception as e:
            self.failures += 1
            self.last_failure_time = time.time()
            if self.failures >= self.failure_threshold:
                self.state = "open"
            raise

breaker = CircuitBreaker(failure_threshold=5, reset_timeout=60)

def safe_api_call(messages):
    return breaker.call(
        client.messages.create,
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=messages
    )
```

---

## Retry with Exponential Backoff

```python
import time
import anthropic

def create_with_retry(client, max_retries=3, **kwargs):
    for attempt in range(max_retries):
        try:
            return client.messages.create(**kwargs)

        except anthropic.RateLimitError:
            if attempt == max_retries - 1:
                raise
            wait = (2 ** attempt) + 0.5  # 0.5s, 1.5s, 3.5s
            time.sleep(wait)

        except anthropic.APIStatusError as e:
            if e.status_code in (500, 529):  # server errors — retry
                if attempt == max_retries - 1:
                    raise
                time.sleep(2 ** attempt)
            else:
                raise  # 4xx errors — don't retry
```

---

## Exam Anti-Patterns

| Anti-Pattern | Why Wrong | Fix |
|-------------|-----------|-----|
| No token budget check before long agent loops | Context overflow mid-loop = truncation errors | Count tokens; compact before hitting 80% of limit |
| `while True` without context size monitoring | Long loops fill context silently | Track message count; compact at threshold |
| Caching 200-token blocks | Below minimum threshold — no cache benefit | Ensure cached blocks are 1,024+ tokens |
| Streaming in tool-use agentic loops | `stop_reason` only reliable on complete response | Use non-streaming for loops; streaming for final output |
| No retry on `RateLimitError` | API rate limits are transient | Exponential backoff with jitter |
| Compacting to single summary message | Loses conversation structure the model needs | Keep 2 messages (user summary + assistant ack) + recent turns |

---

## Quick Reference: Context Usage Headers

```python
# After each API call, log context efficiency:
usage = response.usage
print(f"Input: {usage.input_tokens} | "
      f"Output: {usage.output_tokens} | "
      f"Cache write: {usage.cache_creation_input_tokens} | "
      f"Cache read: {usage.cache_read_input_tokens}")
```

---

👉 **[Get 360 practice questions — Free Enrollment](https://www.udemy.com/course/claude-certified-architect-cca-f-6-practice-tests-2026/?couponCode=FFA33A1350C02189AC15)**

← [Domain 4](./domain4-prompt-engineering.md) | [Back to Index](./README.md)
