# Technical Guide: Prompt Engineering for Developers

This section covers advanced techniques for developers working with LLMs for code generation, agentic workflows, and deterministic outputs.

---

## Context Files for Coding Agents

Every major AI coding tool reads configuration files from your repository. These are the single most important factor for deterministic code output.

### The Complete Landscape

| File | Tool | Format |
|------|------|--------|
| `CLAUDE.md` | Claude Code | Markdown (~300 lines max recommended) |
| `AGENTS.md` | Codex CLI, Cursor, Claude Code (fallback) | Markdown (Linux Foundation standard, 60K+ projects) |
| `GEMINI.md` | Gemini CLI | Markdown |
| `.cursor/rules/*.mdc` | Cursor (current) | MDC (Markdown+) with scoped rules |
| `.github/copilot-instructions.md` | GitHub Copilot | Markdown |
| `.windsurf/rules/*.md` | Windsurf | Markdown |

All tools converged on the same idea: a markdown file in your repo that the AI reads before doing anything.

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

**Key constraint:** Frontier LLMs can reliably follow ~150-200 instructions. Keep context files concise.

---

## Agentic Coding Best Practices

### 1. Plan First, Then Code

Separate research/planning from implementation.

**Cursor's Plan Mode (Shift+Tab):**
1. Agent researches your codebase
2. Asks clarifying questions
3. Creates a detailed implementation plan
4. Waits for approval before building

**Claude Code's 4-phase workflow:**
1. **Explore** — Read files, understand architecture
2. **Plan** — Create detailed implementation plan
3. **Implement** — Code against the plan with verification
4. **Commit** — Descriptive commit + PR

### 2. Give the Agent a Way to Verify Its Work

The single highest-leverage thing you can do.

**Bad:** `implement a function that validates email addresses`
**Good:** `write a validateEmail function. Test cases: user@example.com → true, invalid → false, user@.com → false. Run the tests after implementing.`

### 3. Manage Context Window Aggressively

*"Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills."* — Anthropic

**Start a new conversation when:** moving to a different task, agent seems confused, finished one logical unit of work.

### 4. Reference Files, Don't Copy Content

```
See components/Button.tsx for canonical component structure
```

---

## Compiled AI — LLMs as Compilers

A paradigm from arXiv:2604.05150 that achieves true determinism by using LLMs once during compilation, then executing as static code.

### The Three Properties

1. **One-time LLM invocation** — Model runs once at generation time
2. **Zero-token deterministic execution** — Deployed workflows run as static code
3. **Mandatory multi-stage validation** — Security, syntax, execution, accuracy checks

### Results

| Metric | Result |
|--------|--------|
| Task completion (function-calling) | 96% |
| Token reduction at 1K transactions | 57× vs Direct LLM |
| Break-even point | ~17 transactions |

---

## TDD Governance for Multi-Agent Systems

From arXiv:2604.26615 — operationalizes Test-Driven Development as prompt-level governance for multi-agent code generation.

### Pipeline

1. **Planning agent** — analyzes requirements, writes test cases first
2. **Generation agent** — writes code to pass the tests
3. **Repair agent** — fixes failing code
4. **Validation agent** — independently verifies output

Tests provide objective pass/fail criteria — no ambiguity about correctness.

---

## Prompt Patterns for Deterministic Code

### Specify the Exact File and Function

```
In src/auth/session.ts, modify the createSession function to:
- Accept a second parameter: options: { ephemeral?: boolean }
- If ephemeral is true, set cookie maxAge to 0
- Run: npm run typecheck
```

### Reference Existing Code

```
Create a new API route at app/api/users/route.ts following the 
same pattern as app/api/posts/route.ts.
```

### Constrain the Solution Space

```
Refactor this function. Constraints:
- Do NOT change the function signature
- Do NOT add new dependencies
- Keep existing tests passing
```

### Specify What NOT to Do

```
Add logging. Rules:
- Do NOT modify existing function signatures
- Do NOT add new files
- Follow existing pattern in logger.ts
- Do NOT import new libraries
```

---

## Configuration Reference

| Parameter | Value | Use Case |
|-----------|-------|----------|
| `temperature` | 0.0 | Code generation, refactoring, debugging |
| `temperature` | 0.3-0.5 | Code explanation, documentation |
| `top_p` | 1.0 | With temperature=0, no effect |
| `seed` | Fixed | Best-effort determinism |
| `response_format` | `json_object` | Structured code review output |

---

*See main [README.md](./README.md) for the general guide.*
