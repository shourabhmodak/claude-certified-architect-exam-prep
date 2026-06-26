# Claude Certified Architect (CCA-F) Exam Prep & Study Guide 🚀

[![Claude Certified Architect](https://shields.io)](https://github.com)
[![Udemy Course](https://shields.io)](https://www.udemy.com/course/claude-certified-architect-cca-f-6-practice-tests-2026/?referralCode=D8A4535C0858621CB62B)

Welcome to the definitive open-source study guide and architectural blueprint repository for the **Anthropic Claude Certified Architect – Foundations (CCA-F)** examination. 

---

## 🚨 TEST YOUR EXAM READINESS UNDER REAL CONDITIONS
The official Skilljar exam relies on heavy **production-grade scenarios**, not simple trivia. Don't risk a \$99 retry fee by guessing.

Get access to our **Udemy Exam Simulator**:
* **4 Full-Length Simulated Exams** (240 unique scenario-based questions)
* **Exact Domain Weights** matching the official Anthropic blueprint
* **Deep-Dive Explanations** detailing *why* options are correct or anti-patterns
* **Direct Links** to official Anthropic documentation and cookbook samples

---

🚨 FREE BETA-ACCESS FOR DETECTORS (100 SLOTS ONLY) 🚨
Want to test your readiness under real exam conditions? 
I am gifting 100 free preparation slots to the GitHub engineering community.
All I ask is for your honest rating and review to help me improve the question bank.

👉 [CLICK HERE TO ENROLL IN THE UDEMY SIMULATOR FOR FREE](https://www.udemy.com/course/claude-certified-architect-cca-f-6-practice-tests-2026/?couponCode=FFA33A1350C02189AC15)

---

## 📋 Table of Contents
1. [Domain 1: Agentic Architecture & Orchestration](#1-agentic-architecture--orchestration)
2. [Domain 2: Tool Design & MCP Integration](#2-tool-design--mcp-integration)
3. [The Hub-and-Spoke Pattern Architecture Blueprint](#3-the-hub-and-spoke-pattern-architecture-blueprint)
4. [Exam Day Anti-Patterns to Avoid](#4-exam-day-anti-patterns-to-avoid)

---

## 1. Agentic Architecture & Orchestration

The CCA-F exam aggressively tests your ability to build **deterministic code loops** around probabilistic LLM responses. A core requirement is mastering the standard **4-Step Agentic Loop**:
1. **Execute**: Send user request + system context to Claude.
2. **Evaluate**: Parse the structural `stop_reason` property. 
3. **Call Tool**: If `stop_reason == "tool_use"`, execute your local code deterministically.
4. **Respond**: Send the tool results back to Claude to close the loop.

### 🛑 Crucial Exam Concept: Flow Control Parsing
**Exam Trap:** Never use regex or conversational strings to determine if an agent should stop or continue. Always evaluate the structural metadata.

Here is the exact production-grade Python pattern required for robust state management:

```python
import anthropic

client = anthropic.Anthropic()

def run_agentic_loop(user_prompt, tools_schema):
    messages = [{"role": "user", "content": user_prompt}]
    
    while True:
        # Step 1: Execute call to Claude 3.5 Sonnet
        response = client.messages.create(
            model="claude-3-5-sonnet-latest",
            max_tokens=4000,
            tools=tools_schema,
            messages=messages
        )
        
        # Append Claude's response to keep context intact
        messages.append({"role": "assistant", "content": response.content})
        
        # Step 2: Evaluate structural stop_reason (EXAM FOCUS)
        if response.stop_reason == "end_turn":
            # The agent has completed the task deterministically
            return response.content
            
        elif response.stop_reason == "tool_use":
            # Find the specific tool block requested by Claude
            tool_use_block = next(block for block in response.content if block.type == "tool_use")
            
            # Step 3: Call your local tool securely (Outside LLM boundaries)
            tool_result = execute_local_tool(tool_use_block.name, tool_use_block.input)
            
            # Step 4: Respond by feeding tool output back into the conversation state
            messages.append({
                "role": "user",
                "content": [
                    {
                        "type": "tool_result",
                        "tool_use_id": tool_use_block.id,
                        "content": tool_result
                    }
                ]
            })
```

---

## 2. Tool Design & MCP Integration

When building architecture for Model Context Protocol (MCP), you must construct highly rigid, strict JSON schemas. 

### Key Exam Directives:
* **PreToolCall Hooks**: Security, input sanitization, and enterprise compliance validation *must* happen here in your deterministic code layer, never inside a system prompt.
* **Property Explicitly**: Every tool property should have an explicit type and a clear string description to minimize model hallucinations during tool selection.

---

## 3. The Hub-and-Spoke Pattern Architecture Blueprint

For complex enterprise pipelines (like the exam's Customer Support Refund scenario), giving a single "Mega-Agent" access to 10+ tools degrades context and causes loops to break. 

The architecturally sound answer on the exam is always the **Hub-and-Spoke Pattern**:

Use code with caution.[ User Input ]│▼┌─────────────────┐│  Hub Agent      │ <─── Manages state & routing└─────────────────┘│     │     │┌────────┘     │     └────────┐▼              ▼              ▼┌────────────┐ ┌────────────┐ ┌────────────┐│ Spoke 1    │ │ Spoke 2    │ │ Spoke 3    ││ (CRM Tool) │ │ (Refunds)  │ │ (Shipping) │└────────────┘ └────────────┘ └────────────┘
---

## 4. Exam Day Anti-Patterns to Avoid

* ❌ **Anti-Pattern:** Mitigating JSON schema parsing errors by adding "Please output valid JSON" to the system prompt.
*  **Architectural Fix:** Enforce schema compliance programmatically using tool definitions, structural system hooks, or Pydantic validation layers.
* ❌ **Anti-Pattern:** Passing massive database logs directly into the context window without preprocessing.
*  **Architectural Fix:** Use the `/compact` pipeline pattern or abstract data layers to prevent the "lost in the middle" retrieval degradation effect.

---

## 🎯 Mastered the Basics? Pass the Exam Now.
If you understand the code snippets above, you are ready for advanced scenarios. Take our full practice bank to guarantee your certification on the first attempt.

👉 [**Get 6 Full-Length Practice Tests - 100% Off Limited Udemy Coupon**](https://www.udemy.com/course/claude-certified-architect-cca-f-6-practice-tests-2026/?couponCode=FFA33A1350C02189AC15)

---
*Disclaimer: This repository is an independent study resource and is not officially affiliated with or endorsed by Anthropic.*