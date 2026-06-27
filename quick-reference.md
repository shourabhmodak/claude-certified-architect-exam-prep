# CCA-F One-Page Quick Reference

All 5 domains. Everything you need in the last 30 minutes before the exam.

→ [Full cheat sheets](./cheat-sheet/README.md) | [Sample questions](./sample-questions.md) | [Simulator](./cca-f-simulator.html)

---

## Exam Format

| Questions | Time | Pass | Retry fee |
|-----------|------|------|-----------|
| 60 scenario-based | 90 min | 720/1000 (72%) | $99 |

---

## Domain 1 — Agentic Architecture (27%)

**The 4 stop_reason values and what each means:**

| Value | Meaning | Action |
|-------|---------|--------|
| `end_turn` | Task complete | Return response |
| `tool_use` | Claude wants a tool | Execute tool, append result, loop |
| `max_tokens` | Response truncated | Increase max_tokens or compact context |
| `stop_sequence` | Custom stop string hit | Return response |

**Minimum correct loop:**
```python
for _ in range(MAX_ITERATIONS := 15):         # never while True
    response = client.messages.create(...)
    messages.append({"role": "assistant", "content": response.content})  # full content, not just text

    if response.stop_reason == "end_turn":
        return response.content

    if response.stop_reason == "tool_use":
        tool_block = next(b for b in response.content if b.type == "tool_use")
        result = run_tool(tool_block.name, tool_block.input)
        messages.append({"role": "user", "content": [
            {"type": "tool_result", "tool_use_id": tool_block.id, "content": result}
        ]})
```

**Hub-and-spoke:** use when single agent has 10+ tools → context saturation → routing failures.
Each spoke = its own agent, narrow tool surface (3–4 tools), own message history.

**Subagent isolation:** each sub-call has empty history — pass state explicitly via system prompt.

---

## Domain 2 — Tool Design & MCP (18%)

**Schema must-haves:**
- `required` array — omitting it = all params optional
- Property `description` with format example: `"e.g. ORD-12345"`
- Tool `description` = when to call + when NOT to call

**Hooks vs Prompts:**

| If it MUST happen | If it SHOULD happen |
|------------------|---------------------|
| PreToolCall hook | System prompt |
| PostToolCall hook | CLAUDE.md |
| Deterministic | Probabilistic |

**PreToolCall:** auth, rate limiting, input sanitization, operation ordering enforcement
**PostToolCall:** PII scrubbing, audit logging, output transformation

**Tool error handling:** return error as string in `tool_result` content — don't raise exceptions.

---

## Domain 3 — Claude Code Config (20%)

**File precedence (high → low):**
1. `~/.claude/settings.json` (user)
2. `.claude/settings.json` (project)
3. `~/.claude/CLAUDE.md` (user context)
4. `/repo/CLAUDE.md` (project context)

**Hook types:**

| Hook | Fires | Blocks on exit code |
|------|-------|---------------------|
| `PreToolUse` | Before tool | Yes — exit 1 blocks |
| `PostToolUse` | After tool | Yes — exit 1 signals error |
| `UserPromptSubmit` | User sends prompt | Yes — exit 1 blocks |
| `Stop` | Claude finishes turn | No — informational only |

**Critical distinction:** CLAUDE.md = advisory context. Hooks + permissions = enforcement.

**Slash commands:** `.claude/commands/filename.md` → available as `/filename` to all devs on repo.

**Deny rule example:** `"Bash(rm -rf*)"` in `permissions.deny` blocks all matching commands.

---

## Domain 4 — Prompt Engineering (20%)

**Structured output reliability ranking (best → worst):**
1. Tool definition + `tool_choice: {"type": "any"}` ← only structurally enforced
2. XML `<output_format>` block in system prompt ← advisory
3. "Respond with JSON" in system prompt ← unreliable
4. "Respond with JSON" in user message ← least reliable

**Explicit criteria pattern:**
```
# WRONG: "Classify as appropriate"
# RIGHT:
Categories: BUG | FEATURE_REQUEST | QUESTION | BILLING | ACCOUNT
Pick exactly one. If unclear, pick most likely.
```

**System prompt layer order:**
1. `<persona>` — who Claude is
2. `<task>` — what it must do
3. `<rules>` — numbered constraints
4. `<output_format>` — schema, last (trailing instructions have highest weight)

**Prompt injection defense:** label all external/retrieved content as `[UNTRUSTED]`. Put behavioral rules in `<security>` block. Never rely on system prompt alone for sensitive rules — use PreToolCall hooks.

**Temperature guide:**
- `0.0` — extraction, classification, code generation
- `0.3–0.5` — summarization
- `0.7–1.0` — creative, brainstorming

---

## Domain 5 — Context Management (15%)

**"Lost in the middle":** content at the start and end of context gets more attention than the middle. Affects agents with 20+ turns or large retrieved documents.

**Compaction trigger:** message count > threshold → summarize old turns with Haiku → replace with `[summary] + last N turns`.

**Prompt caching math:**
- Cache write: **1.25x** normal input cost
- Cache hit: **0.1x** normal input cost
- Break-even: 2nd request
- TTL: 5 minutes (reset on each hit)
- Min cacheable size: ~1,024 tokens

**When to use streaming:**
- ✅ Long responses in user-facing UI
- ❌ Tool-use agentic loops (need complete response for `stop_reason`)
- ❌ Programmatic response parsing

**Retry pattern:** exponential backoff on `RateLimitError` and 5xx. Don't retry 4xx (client error).

---

## The One Frame That Unlocks This Exam

Every question is really asking: **"Is this deterministic or probabilistic?"**

| Deterministic (guaranteed) | Probabilistic (advisory) |
|---------------------------|-------------------------|
| PreToolCall / PostToolCall hooks | System prompt instructions |
| Tool schema `required` fields | CLAUDE.md project context |
| `stop_reason` API field | "Always do X" in prompts |
| `permissions.deny` rules | Temperature=0 |
| `tool_choice: any` | Few-shot examples |

When the exam shows you a bug, ask: "Is enforcement happening in code (good) or in a prompt (not enough)?"

---

## Top 10 Exam Traps

1. **Text-parsing tool detection** → use `stop_reason == "tool_use"`
2. **`while True` loop** → always add `MAX_ITERATIONS` guard
3. **Appending only `.text`** → append full `response.content`
4. **No `required` in tool schema** → all params become optional
5. **Security in system prompt** → move to PreToolCall hook
6. **CLAUDE.md as enforcement** → it's advisory; hooks enforce
7. **Subagents share hub state** → each starts with empty history
8. **"Output JSON" in prompt** → use tool definition + `tool_choice: any`
9. **Caching short blocks** → min ~1,024 tokens to cache
10. **Ignoring `max_tokens` stop_reason** → handle all 4 values

---

👉 **[Free Udemy Enrollment — 100 slots](https://www.udemy.com/course/claude-certified-architect-cca-f-6-practice-tests-2026/?couponCode=FFA33A1350C02189AC15)**
