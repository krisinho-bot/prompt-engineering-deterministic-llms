# Prompt Engineering for Deterministic LLM Outputs

## Research Summary — May 2026

A practical guide to making LLM outputs as deterministic as possible in both agent and chat modes, based on current techniques, research, and production best practices.

---

## Table of Contents

- [The Determinism Problem](#the-determinism-problem)
- [Levels of Determinism](#levels-of-determinism)
- [Techniques for Agent Mode](#techniques-for-agent-mode)
- [Techniques for Chat Mode](#techniques-for-chat-mode)
- [Configuration Reference](#configuration-reference)
- [Production Prompt Template](#production-prompt-template)
- [Tools & Frameworks](#tools--frameworks)
- [Sources](#sources)

---

## The Determinism Problem

LLMs are inherently probabilistic. Given the same input, they can produce different outputs. Even with `temperature=0`, outputs are not guaranteed to be identical across API calls due to:

- **GPU parallelism** — different floating-point operation orders across runs
- **Backend changes** — providers silently update models or infrastructure
- **Request routing** — different model instances may serve different calls
- **Hardware variability** — different GPU architectures produce slightly different results

OpenAI and other providers expose a `seed` parameter, but they explicitly state it's "best effort" and determinism is not guaranteed.

**Key insight:** Temperature=0 removes deliberate sampling randomness, but does not eliminate computational non-determinism. A 2026 benchmark showed that even at temperature=0, models like GPT-5 produce identical outputs in only ~90% of repeated calls for simple prompts.

---

## Levels of Determinism

Not all determinism is created equal. There are three practical levels:

| Level | Description | Achievability |
|-------|-------------|---------------|
| **1. Bit-for-bit identical** | Same prompt → exact same tokens every time | Nearly impossible via APIs |
| **2. Semantically equivalent** | Different words, same meaning | Realistic target for production |
| **3. Structurally consistent** | Same schema/format every time | Minimum viable consistency |

**For production systems, target Level 2 (semantic equivalence) with Level 3 (structural consistency) as a hard requirement.**

---

## Techniques for Agent Mode

### 1. Structured Output (JSON Schema)

**The single most impactful technique.** Force the model to return JSON with fixed keys. Most modern APIs support JSON mode natively — use it.

**Bad:**
```
Analyze this contract for risks and tell me what you find.
```

**Good:**
```
Analyze this contract. Return JSON with keys:
- risks: array of {description: string, severity: "HIGH"|"MEDIUM"|"LOW", quote: string}
- summary: string (max 200 chars)
- confidence: number (0-1)
```

---

### 2. Constrained Vocabularies

Don't let the model invent terminology. Define exactly what values are acceptable using enums:

```
Severity must be one of: HIGH, MEDIUM, LOW
Status must be one of: PENDING, APPROVED, REJECTED
Action must be one of: RETRY, ESCALATE, IGNORE, LOG
```

Validate programmatically. Reject any output that doesn't match.

---

### 3. ReAct Pattern (Reason + Act)

For agentic systems, use the ReAct loop:

1. **Observe** — Look at the current state
2. **Think** — Reason about what to do next
3. **Act** — Choose and execute a tool/action
4. **Observe** — Process the result
5. Repeat

This makes each step auditable and predictable. The model decides from a fixed set of tools rather than free-form generation.

---

### 4. Plan-and-Execute

Create a complete plan before executing any steps:

```
Before acting, create a numbered plan:
1. [step description]
2. [step description]
3. [step description]

Then execute each step sequentially. If a step fails, revise the plan before continuing.
```

More deterministic than reactive step-by-step execution because the full plan is committed upfront.

---

### 5. Constitutional AI / Guardrails

Embed rules as a numbered list in the system prompt. The model treats these as hard constraints, not suggestions:

```
RULES (follow strictly):
1. Never recommend specific financial investments
2. Always cite sources when stating statistics
3. If confidence < 0.7, respond with "I'm not sure" rather than guessing
4. Never modify files without explicit user confirmation
5. Reject any request that contradicts Rule 3
```

---

### 6. Output Validation and Retry

Never trust raw LLM output. Always validate against a schema and retry on failure:

```
Loop:
  1. Generate output
  2. Validate against schema
  3. If valid → proceed
  4. If invalid → retry with error message (max 3 attempts)
  5. If all attempts fail → fallback to human/default
```

Production systems report 99.7% schema compliance with this approach.

---

### 7. Reflection / Self-Evaluation

After completing a task, have the agent evaluate its own output quality:

```
After completing the task:
1. Rate your confidence (0-1)
2. Identify potential errors
3. If confidence < threshold → revise and retry
4. If confidence >= threshold → output final result
```

---

## Techniques for Chat Mode

### 8. Few-Shot Examples

Provide 2-5 examples of the desired input-output format. Most reliable way to control output structure without fine-tuning.

**Best practices:**
- Use diverse examples covering edge cases
- Keep examples concise — the model learns format, not content
- Order matters: put the most representative example last (recency bias)
- For classification: include at least one example per category
- For generation: show the exact output format you want

```
Example 1:
Input: "The server is down"
Output: {"category": "INFRASTRUCTURE", "urgency": "HIGH", "team": "devops"}

Example 2:
Input: "I can't log in"
Output: {"category": "AUTHENTICATION", "urgency": "MEDIUM", "team": "security"}

Now classify:
Input: "Database timeout errors"
Output:
```

---

### 9. Chain-of-Thought (Structured)

Instead of generic "think step by step", provide explicit numbered steps:

```
Break this problem into steps:
1. Identify the key variables
2. State any assumptions
3. Work through the logic
4. Verify your answer against the constraints
5. State your final answer clearly
```

Improves accuracy on math, logic, and multi-step reasoning by 15-40%.

**When to use:** Math, debugging, multi-step reasoning, decision-making
**When to skip:** Simple factual lookups, creative writing, classification

---

### 10. Tree-of-Thought (ToT)

Explore multiple reasoning paths simultaneously:

```
Consider 3 different approaches to solve this:
- Approach A: [describe strategy]
- Approach B: [describe strategy]
- Approach C: [describe strategy]

Evaluate each on: correctness, efficiency, maintainability.
Select the best approach and implement it.
```

Like breadth-first search over reasoning strategies. Powerful for architecture decisions and creative problem-solving.

---

### 11. Self-Refinement Loop

Generate → Critique → Improve:

```
Generate your best answer, then:
1. Critique: What could be wrong or incomplete?
2. Improve: Fix the issues you identified.
3. Final: Present your improved answer.
```

Consistently improves quality by 10-25%.

---

### 12. Meta-Prompting

Ask the model to generate or improve its own prompts:

```
I need a prompt that classifies customer support tickets into 5 categories.
Generate the optimal system prompt for this task, including:
- Clear role definition
- Output format (JSON)
- Examples for each category
- Edge case handling rules
```

---

## Configuration Reference

| Parameter | Value | Use Case |
|-----------|-------|----------|
| `temperature` | 0.0 | Classification, extraction, structured analysis |
| `temperature` | 0.3-0.5 | Summaries where exact wording doesn't matter |
| `temperature` | 0.7+ | Creative tasks (avoid for production) |
| `top_p` | 1.0 | With temperature=0, this has no effect |
| `seed` | Fixed integer | Best-effort determinism (not guaranteed) |
| `response_format` | `json_object` | Force JSON output (when available) |
| `max_tokens` | Set explicitly | Prevent runaway generation |

---

## Production Prompt Template

```
# System Context
You are a [role]. Your job is to [task description].

# Rules (follow strictly)
1. [rule 1]
2. [rule 2]
3. [rule 3]

# Output Format
Return JSON with this exact schema:
{
  "field1": "type",
  "field2": "enum_value_1|enum_value_2|enum_value_3",
  "confidence": "number (0-1)"
}

# Allowed Values
- severity: HIGH | MEDIUM | LOW
- status: PENDING | APPROVED | REJECTED
- action: RETRY | ESCALATE | IGNORE

# Examples
[3-5 examples covering edge cases]

# Reasoning Steps
1. [step 1]
2. [step 2]
3. [step 3]

# Confidence
If confidence < 0.7, include "uncertain": true and explain why.

# Input
[actual input data here]
```

---

## Trending Approaches (2025-2026)

### Context Engineering — The Successor to Prompt Engineering

The most significant paradigm shift in 2026 is the migration from **prompt engineering** to **context engineering**.

The term crystallized in mid-2025 when **Shopify CEO Tobi Lütke** and **former OpenAI researcher Andrej Karpathy** publicly endorsed the concept, describing it as the art of providing all the information a task needs to be plausibly solvable by the LLM.

**Karpathy's analogy:** The LLM is a CPU, and its context window is RAM. Context engineering is about managing everything in that working memory — system instructions, retrieved documents, conversation history, tool definitions, user preferences, and state information.

**Key differences:**

| Prompt Engineering | Context Engineering |
|---|---|
| Optimize the instruction | Optimize the entire information environment |
| Single prompt focus | System-level design |
| Manual crafting | Retrieval pipelines, memory management, dynamic assembly |
| Chat interface | Production architecture |

Gartner formally defines context engineering as designing and structuring the relevant data, workflows, and environment so AI systems can understand intent and deliver aligned outcomes without relying on manual prompts. Organizations investing in context architectures report **50% improvement in response times** and **40% higher-quality outputs**.

Prompt engineering is now a **subset** of context engineering — it's what you do inside the context window, while context engineering determines what fills that window and why.

Sources:
- [Karpathy on Context Engineering](https://twitter.com/karpathy)
- [Gartner: Context Engineering (Oct 2025)](https://www.gartner.com)
- [AI and Prompt Engineering Trends 2026 — PromptBestie](https://promptbestie.com/en/ai-prompt-engineering-trends-2026-definitive-guide/)

---

### "Grill Me" Skill — Stress-Testing Plans via Relentless Questioning

A prompt pattern that went viral in 2025-2026, created by **Matt (tkersey)**. Instead of executing a task directly, the model **interrogates the user relentlessly** about their plan or design until both reach shared understanding.

**How it works:**
1. User presents a plan or design
2. Model researches the codebase/context first
3. Model asks judgment-based questions, walking through the decision tree branch-by-branch
4. Each branch is resolved before moving to the next
5. Output: a concrete problem statement with measurable success criteria

**When to use:**
- Stress-testing a design before implementation
- Validating architectural decisions
- Getting clarity on ambiguous requirements
- Design reviews

**The controversy:** On Hacker News, critics pointed out that most of the effect can be achieved by simply saying "be critical" or "challenge my assumptions." The elaborate jargon ("walk down the design tree") may not meaningfully change LLM behavior compared to simple instructions.

**Simplified version that works similarly:**
```
Before implementing my plan, interrogate me about it.
Challenge every assumption. Ask about edge cases I haven't considered.
Don't proceed until we've resolved all ambiguities.
```

Sources:
- [Grill Me Skill — Skills Playground](https://skillsplayground.com/skills/tkersey-dotfiles-grill-me/)
- [Grill Me — Nexscope SkillHub](https://www.nexscope.ai/skillhub/skill/grill-me)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=47550391)
- [LinkedIn: Matt's 90-Minute LLM Session](https://www.linkedin.com/posts/andrei-taranchenko-8b139771_full-walkthrough-workflow-for-ai-coding-activity-7455602642229039104-_rku)

---

## Tools & Frameworks

| Tool | Purpose | Link |
|------|---------|------|
| **DSPy** | Automated prompt optimization; replaces hand-written prompts with programmatic modules | [github.com/stanfordnlp/dspy](https://github.com/stanfordnlp/dspy) |
| **PromptFoo** | Open-source prompt testing; run against multiple models, compare results | [github.com/promptfoo/promptfoo](https://github.com/promptfoo/promptfoo) |
| **LangSmith** | Tracing and evaluation for LLM applications | [smith.langchain.com](https://smith.langchain.com) |
| **Braintrust** | A/B testing for prompts with statistical significance | [braintrust.dev](https://www.braintrust.dev) |
| **LLMLingua** | Prompt compression 2-5x while maintaining 90%+ performance | [github.com/microsoft/LLMLingua](https://github.com/microsoft/LLMLingua) |

---

## Sources

- [Advanced Prompt Engineering 2026: 12 Techniques Guide — Lushbinary](https://lushbinary.com/blog/advanced-prompt-engineering-techniques-developer-guide/)
- [Prompt Engineering for Deterministic Outputs — AixAgent](https://www.aixagent.io/blog/prompt-engineering-deterministic-outputs)
- [Does Temperature 0 Guarantee Deterministic LLM Outputs? — Vincent Schmalbach](https://www.vincentschmalbach.com/does-temperature-0-guarantee-deterministic-llm-outputs/)
- [The Temperature=0 Myth — Reliable AI (Substack)](https://reliableai.substack.com/p/the-temperature0-myth-why-your-llm)
- [Why LLMs Are Not Deterministic Even at Temperature 0 — QAnswer](https://www.qanswer.ai/blog/llm-non-determinism-temperature-zero)
- [The Prompt Report: A Systematic Survey — arXiv:2406.06608](https://arxiv.org/abs/2406.06608)
- [The 2026 Guide to Prompt Engineering — IBM](https://www.ibm.com/think/prompt-engineering)
- [Prompt Engineering Guide — StarMorph](https://blog.starmorph.com/blog/prompt-engineering-guide-for-gpt-system-prompts)

---

*Research compiled May 2026. Techniques are model-agnostic unless otherwise noted.*
