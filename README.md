# Prompt Engineering for Deterministic LLM Code Generation

## Research Summary — May 2026

A practical investigation into making LLM outputs deterministic and reliable specifically for **software development**: code generation, refactoring, debugging, and agentic coding workflows.

---

## Table of Contents

- [The Problem: Non-Determinism in Code Generation](#the-problem-non-determinism-in-code-generation)
- [Context Files: The Foundation of Deterministic Coding Agents](#context-files-the-foundation-of-deterministic-coding-agents)
- [Agentic Coding Best Practices (Cursor, Claude Code, Copilot)](#agentic-coding-best-practices-cursor-claude-code-copilot)
- [Compiled AI: Generating Deterministic Code from LLMs](#compiled-ai-generating-deterministic-code-from-llms)
- [TDD Governance for Multi-Agent Code Generation](#tdd-governance-for-multi-agent-code-generation)
- [Prompt Patterns for Deterministic Code Output](#prompt-patterns-for-deterministic-code-output)
- [Configuration Reference for Code Tasks](#configuration-reference-for-code-tasks)
- [Trending: Context Engineering & Grill Me](#trending-context-engineering--grill-me)
- [Tools & Frameworks](#tools--frameworks)
- [Sources](#sources)

---

## The Problem: Non-Determinism in Code Generation

LLMs generating code is inherently risky because of stochastic output. The same prompt can produce:

- Different variable names, function signatures, or file structures
- Different error handling strategies
- Different library choices or API patterns
- Subtle logic differences that pass code review but behave differently

**Key data points:**
- Even at `temperature=0`, accuracy varies up to **15% across runs** (Atil et al., 2024)
- Output variance of **18-75%** due to Mixture-of-Experts routing (Ouyang et al., 2023)
- 79% of multi-agent failures stem from **specification and coordination issues**, not infrastructure (Cemri et al., 2025)
- Agent success rates degrade from **58% single-turn to 35% multi-turn** (Salesforce CRMArena-Pro, 2025)

For code, this means: the same "refactor this function" prompt can produce working code on one run and broken code on the next.

### Levels of Determinism for Code

| Level | Description | Example |
|-------|-------------|---------|
| **1. Bit-for-bit identical** | Same prompt → exact same code | Nearly impossible via APIs |
| **2. Semantically equivalent** | Different variable names, same behavior | Realistic target |
| **3. Structurally consistent** | Same file structure, exports, patterns | Minimum viable |

**Target for production code: Level 2 (semantic equivalence) with Level 3 (structural consistency).**

---

## Context Files: The Foundation of Deterministic Coding Agents

Every AI coding tool now reads configuration files from your repository. These are the single most important factor for deterministic code output — they give the model your project's conventions, patterns, and constraints before it writes a single line.

### The Complete Landscape

| File | Tool | Format |
|------|------|--------|
| `CLAUDE.md` | Claude Code | Markdown (~300 lines max recommended) |
| `AGENTS.md` | Codex CLI, Cursor, Claude Code (fallback) | Markdown (Linux Foundation standard, 60K+ projects) |
| `GEMINI.md` | Gemini CLI | Markdown |
| `.cursor/rules/*.mdc` | Cursor (current) | MDC (Markdown+) with scoped rules |
| `.github/copilot-instructions.md` | GitHub Copilot | Markdown |
| `.windsurf/rules/*.md` | Windsurf | Markdown |

**All tools converged on the same idea:** a markdown file in your repo that the AI reads before doing anything. The differences are naming, discovery order, and hierarchy.

### What to Put in Context Files

```markdown
# Commands
- `npm run build`: Build the project
- `npm run typecheck`: Run the typechecker
- `npm run test`: Run tests (prefer single test files for speed)

# Code Style
- Use ES modules (import/export), not CommonJS (require)
- Destructure imports when possible: `import { foo } from 'bar'`
- See `components/Button.tsx` for canonical component structure

# Workflow
- Always typecheck after making a series of code changes
- API routes go in `app/api/` following existing patterns
- Never modify migration files after they've been committed
```

**What NOT to put:**
- Entire style guides (use a linter instead)
- Every possible command (the agent knows common tools)
- Edge case instructions that rarely apply

**Key constraint:** Frontier LLMs can reliably follow ~150-200 instructions. Claude Code's system prompt already uses ~50. Keep your context file concise.

### AGENTS.md — The Cross-Tool Standard

`AGENTS.md` is the closest thing to a universal standard, now stewarded by the Linux Foundation with adoption across 60,000+ open-source projects. If you maintain a single `AGENTS.md`, it works with Codex CLI, Cursor, Claude Code, Continue.dev, Aider, and OpenHands.

---

## Agentic Coding Best Practices (Cursor, Claude Code, Copilot)

### 1. Plan First, Then Code

The most impactful pattern: separate research/planning from implementation.

**Cursor's Plan Mode (Shift+Tab):**
1. Agent researches your codebase
2. Asks clarifying questions
3. Creates a detailed implementation plan with file paths
4. Waits for your approval before building

**Claude Code's 4-phase workflow:**
1. **Explore** — Read files, understand architecture (plan mode)
2. **Plan** — Create detailed implementation plan
3. **Implement** — Code against the plan with verification
4. **Commit** — Descriptive commit + PR

**When to skip planning:** If you could describe the diff in one sentence (typos, log lines, renames).

### 2. Give the Agent a Way to Verify Its Work

The single highest-leverage thing you can do (per Anthropic's own teams).

**Bad:**
```
implement a function that validates email addresses
```

**Good:**
```
write a validateEmail function. Test cases: user@example.com → true, 
invalid → false, user@.com → false. Run the tests after implementing.
```

Verification can be: test suite, linter, typechecker, screenshot comparison, or a bash command that checks output.

### 3. Manage Context Window Aggressively

Claude Code's own docs state: *"Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills."*

**When to start a new conversation:**
- Moving to a different task or feature
- Agent seems confused or keeps making same mistakes
- Finished one logical unit of work

**When to continue:**
- Iterating on the same feature
- Agent needs context from earlier in the discussion
- Debugging something it just built

### 4. Start Over From a Plan When Things Go Wrong

Instead of trying to fix a bad agent output through follow-up prompts, revert changes, refine the plan to be more specific, and run again. This is often faster and produces cleaner results.

### 5. Reference Files, Don't Copy Content

In context files and prompts, reference canonical files rather than copying their contents:
```
See components/Button.tsx for canonical component structure
```
This keeps instructions short and prevents them from becoming stale as code changes.

---

## Compiled AI: Generating Deterministic Code from LLMs

A paradigm from a April 2026 paper (Trooskens et al., arXiv:2604.05150) that achieves true determinism by **using LLMs as compilers, not interpreters**.

### Core Idea

Instead of invoking the LLM at runtime for every transaction, use it **once** during a compilation phase to generate executable code artifacts. Workflows then execute deterministically without further model calls.

### The Three Properties of Compiled AI

1. **One-time LLM invocation** — Model runs once at generation time, not at transaction time
2. **Zero-token deterministic execution** — Deployed workflows run as static code with no further model calls
3. **Mandatory multi-stage validation** — Every artifact passes security, syntax, execution, and accuracy checks

### Four-Stage Validation Pipeline

1. **Security analysis** — Detect prompt injection, unsafe patterns
2. **Syntactic verification** — Static analysis, type checking
3. **Execution testing** — Run the generated code against test cases
4. **Output accuracy** — Verify results match expected behavior

### Results

| Metric | Result |
|--------|--------|
| Task completion (function-calling) | 96% |
| Execution tokens | Zero (after compilation) |
| Token reduction at 1K transactions | 57× vs Direct LLM |
| Break-even point | ~17 transactions |
| Prompt injection detection | 96.7% accuracy |
| Static code safety | 87.5% accuracy, zero false positives |

### When to Use

- High-volume, repetitive workflows (form processing, API transformations, data pipelines)
- Compliance-sensitive environments (healthcare, finance)
- When you need auditability and predictable latency
- NOT for exploratory or creative coding tasks

Source: [Deterministic Code Generation for LLM-Based Workflow Automation (arXiv:2604.05150)](https://arxiv.org/html/2604.05150v1)

---

## TDD Governance for Multi-Agent Code Generation

A framework from an April 2026 paper (Hasanli et al., arXiv:2604.26615) that operationalizes Test-Driven Development as **prompt-level and workflow-level governance** for multi-agent code generation.

### Core Idea

Instead of letting agents generate code freely, enforce TDD principles through the prompt architecture itself:

1. **Planning stage** — Agent analyzes requirements and writes test cases first
2. **Generation stage** — Agent writes code to pass the tests
3. **Repair stage** — Agent fixes code that fails tests
4. **Validation stage** — Independent agent verifies the output

### Key Innovation

TDD principles are formalized in a **machine-readable manifesto** and distributed across the agent pipeline. This separates model-specific behavior from governance rules — the TDD constraints survive even when you swap models.

### Why It Matters for Determinism

- Tests provide **objective pass/fail criteria** — no ambiguity about whether code is correct
- The multi-agent architecture means a **planning agent** defines scope and a **validation agent** checks it
- Reduces the "79% of multi-agent failures from specification issues" problem

Source: [TDD Governance for Multi-Agent Code Generation via Prompt Engineering (arXiv:2604.26615)](https://arxiv.org/abs/2604.26615)

---

## Prompt Patterns for Deterministic Code Output

### 1. Specify the Exact File and Function

```
In src/auth/session.ts, modify the createSession function to:
- Accept a second parameter: options: { ephemeral?: boolean }
- If ephemeral is true, set cookie maxAge to 0
- Keep all existing behavior unchanged
- Run: npm run typecheck
```

### 2. Show, Don't Tell — Reference Existing Code

```
Create a new API route at app/api/users/route.ts following the 
same pattern as app/api/posts/route.ts. Use the same error handling, 
authentication middleware, and response format.
```

### 3. Constrain the Solution Space

```
Refactor this function. Constraints:
- Do NOT change the function signature
- Do NOT add new dependencies
- Keep the existing test cases passing
- Use the same error handling pattern as the rest of this file
```

### 4. Verification-First Prompts

```
Write a function that parses CSV headers. Before implementing:
1. Write test cases for: empty input, single column, quoted headers, 
   headers with spaces, headers with special characters
2. Run the tests (they should fail)
3. Implement the function
4. Run the tests again (they should pass)
5. Show me the test output
```

### 5. Specify What NOT to Do

```
Add logging to this service. Rules:
- Do NOT modify any existing function signatures
- Do NOT add any new files
- Do NOT change the log format — follow the existing pattern in logger.ts
- Do NOT import any new libraries
```

### 6. Structured Output for Code Review

```
Review this PR. Return JSON:
{
  "issues": [{"file": string, "line": number, "severity": "HIGH"|"MEDIUM"|"LOW", "description": string}],
  "summary": string,
  "approved": boolean
}
```

---

## Configuration Reference for Code Tasks

| Parameter | Value | Use Case |
|-----------|-------|----------|
| `temperature` | 0.0 | Code generation, refactoring, debugging |
| `temperature` | 0.3-0.5 | Code explanation, documentation |
| `top_p` | 1.0 | With temperature=0, no effect |
| `seed` | Fixed | Best-effort determinism |
| `response_format` | `json_object` | When structured code review output is needed |
| `max_tokens` | Set explicitly | Prevent runaway code generation |

### Model-Specific Tips for Code

| Model | Best For | Tip |
|-------|----------|-----|
| Claude Opus 4.7 | Complex refactors, multi-file changes | Use XML tags for constraints, detailed CLAUDE.md |
| GPT-5.5 | Function calling, structured output | Concise JSON schemas, avoid verbose instructions |
| Gemini 3.1 Pro | Long context, full codebase analysis | Provide grounding documents |

---

## Trending: Context Engineering & Grill Me

### Context Engineering — The Successor to Prompt Engineering

Popularized by **Andrej Karpathy** (ex-OpenAI) and **Tobi Lütke** (Shopify CEO) in mid-2025. The idea: don't optimize individual prompts — optimize the **entire information environment** surrounding the model.

**Karpathy's analogy:** The LLM is a CPU, context window is RAM. Context engineering manages everything in that working memory.

For software development, this means:
- Well-structured `CLAUDE.md` / `AGENTS.md` files
- Codebase indexing and retrieval pipelines
- Persistent memory systems across sessions
- Dynamic context assembly based on the current task

Organizations using context architectures report **50% better response times** and **40% higher quality** (Gartner, 2025).

### "Grill Me" Skill — Stress-Testing Designs Before Coding

A viral pattern from **Matt (tkersey)**: instead of letting the agent implement immediately, it **interrogates you** about your plan until shared understanding is reached. Walks the decision tree branch-by-branch.

**Simplified version:**
```
Before writing any code, challenge my design:
- What edge cases haven't I considered?
- What could go wrong with this approach?
- Are there existing patterns in the codebase I should follow?
Don't proceed until we've resolved all ambiguities.
```

**HN Criticism:** Most of the effect can be achieved with "be critical" — the elaborate jargon may not meaningfully change LLM behavior.

Sources:
- [Grill Me — Skills Playground](https://skillsplayground.com/skills/tkersey-dotfiles-grill-me/)
- [HN Discussion](https://news.ycombinator.com/item?id=47550391)

---

## Tools & Frameworks

| Tool | Purpose | Link |
|------|---------|------|
| **Cursor** | Agentic IDE with Plan Mode, Rules, Skills | [cursor.com](https://cursor.com) |
| **Claude Code** | Terminal-based agentic coding | [code.claude.com](https://code.claude.com) |
| **Codex CLI** | OpenAI's terminal coding agent | [github.com/openai/codex](https://github.com/openai/codex) |
| **Gemini CLI** | Google's terminal coding agent | [github.com/google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli) |
| **DSPy** | Automated prompt optimization for code pipelines | [github.com/stanfordnlp/dspy](https://github.com/stanfordnlp/dspy) |
| **PromptFoo** | Test prompts across models with code evals | [github.com/promptfoo/promptfoo](https://github.com/promptfoo/promptfoo) |
| **LLMLingua** | Compress code context 2-5x | [github.com/microsoft/LLMLingua](https://github.com/microsoft/LLMLingua) |
| **Continue.dev** | Open-source AI coding assistant | [github.com/continuedev/continue](https://github.com/continuedev/continue) |

---

## Sources

### Papers
- [Compiled AI: Deterministic Code Generation for LLM-Based Workflow Automation (arXiv:2604.05150)](https://arxiv.org/html/2604.05150v1) — Trooskens et al., April 2026
- [TDD Governance for Multi-Agent Code Generation via Prompt Engineering (arXiv:2604.26615)](https://arxiv.org/abs/2604.26615) — Hasanli et al., April 2026
- [The Prompt Report: A Systematic Survey of Prompting Techniques (arXiv:2406.06608)](https://arxiv.org/abs/2406.06608)

### Guides & Documentation
- [Best Practices for Coding with Agents — Cursor](https://cursor.com/blog/agent-best-practices)
- [Best Practices for Claude Code — Anthropic](https://code.claude.com/docs/en/best-practices)
- [CLAUDE.md, AGENTS.md & Copilot Instructions Guide — DeployHQ](https://www.deployhq.com/blog/ai-coding-config-files-guide)
- [Agentic Coding with Claude Code and Cursor — Softcery](https://softcery.com/lab/softcerys-guide-agentic-coding-best-practices)

### Articles
- [Advanced Prompt Engineering 2026 — Lushbinary](https://lushbinary.com/blog/advanced-prompt-engineering-techniques-developer-guide/)
- [Prompt Engineering for Deterministic Outputs — AixAgent](https://www.aixagent.io/blog/prompt-engineering-deterministic-outputs)
- [AI and Prompt Engineering Trends 2026 — PromptBestie](https://promptbestie.com/en/ai-prompt-engineering-trends-2026-definitive-guide/)
- [The Temperature=0 Myth — Reliable AI](https://reliableai.substack.com/p/the-temperature0-myth-why-your-llm)
- [Why LLMs Are Not Deterministic Even at Temperature 0 — QAnswer](https://www.qanswer.ai/blog/llm-non-determinism-temperature-zero)

---

*Research compiled May 2026. Focused on software development applications.*
