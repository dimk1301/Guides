# AI Token Efficiency Playbook for Developers

A practical, guide for reducing token usage and improving output quality in **Claude Code**, **Kiro CLI**, and **GitHub Copilot**. Token efficiency comes down to keeping context small and relevant, choosing the right model for the task, and constraining both prompts and outputs so the model processes only what is needed.

---

## Core Mental Model

Think of an LLM request like a function call:

```text
output = model(instructions + history + code + logs + tool_output)

```

LLMs are effectively stateless across turns, so each new request is evaluated against the full active context window rather than a compact internal memory. That means every extra message, file, log block, and repeated instruction is paid for again on later turns, which increases cost and often reduces answer quality once the context becomes noisy.

### What Actually Affects Token Usage

```
                  ┌──────────────────────────────┐
                  │  Context Size (Inputs)       │
                  │  - Active conversation       │
                  │  - System instructions       │
                  │  - Loaded files & terminal   │
                  └──────────────┬───────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                          LLM Request                            │
└──────────────┬──────────────────────────────────┬───────────────┘
               │                                  │
               ▼                                  ▼
┌──────────────────────────────┐  ┌──────────────────────────────┐
│  Model Choice                │  │  Output Constraints          │
│  - Routine tasks → Fast      │  │  - Patches/diffs only        │
│  - Heavy math → Reasoning    │  │  - Strict word & bullet limits│
└──────────────────────────────┘  └──────────────────────────────┘

```

* **Context size:** Conversation history, loaded files, logs, agent instructions, tool output.
* **Model choice:** Stronger models are useful for hard reasoning, but are often wasteful for simple search, formatting, or file-reading tasks.
* **Output length:** Long answers, full rewrites, and unnecessary commentary consume tokens and increase review time.
* **Session length:** Long sessions accumulate stale context and are more likely to repeat mistakes or ignore buried instructions.

---

## 10 Golden Rules

Use these rules even if you are new to AI coding tools.

1. **One session per task:** Start a fresh chat for each bug, feature, or refactor.
2. **Begin with a short task brief:** Task, files, constraints, expected output.
3. **Paste minimal snippets:** Include only the relevant code or log excerpt, not the whole file or full terminal dump.
4. **Demand patches/diffs:** Ask for a patch or diff, not a full rewrite, unless a rewrite is actually needed.
5. **Add output limits:** Use rules such as `patch only`, `under 200 words`, or `max 5 bullets`.
6. **Tier your models:** Use smaller or automatic models for routine work; reserve stronger models for complex reasoning.
7. **Summarize & reset:** Summarize long sessions before they get messy, then continue in a new chat.
8. **Use configuration files:** Store persistent rules in tool-specific instruction files instead of repeating them every time.
9. **Inspect active context:** Use built-in context tools to inspect, compact, or clear context rather than guessing.
10. **Index large repos:** Prefer indexed or searchable knowledge stores for big codebases and docs instead of loading them permanently into context.

---

## Copy-Paste Prompt Templates

### Universal Task Brief

Use this at the start of almost every AI coding session.

```text
Task: Fix the 500 error on /login.
Files: src/auth/login.ts, src/auth/validateUser.ts.
Constraints: No new dependencies, minimal diff, preserve existing logging.
Output: Patch only, no explanation.

```

> **Why it works:** It narrows scope, limits unnecessary exploration, and tells the model when to stop.

### Bug Investigation Template

```text
Task: Find the root cause of the failure.
Relevant area: src/auth/validateUser.ts
Known symptom: 500 error when the DB returns null.
Constraints: Do not redesign architecture. Do not modify unrelated files.
Output: 1) Root cause, 2) Minimal patch, 3) Test case.

```

### Refactor Template

```text
Task: Refactor registerUser() for readability.
Scope: Only registerUser() in src/controllers/userController.ts.
Constraints: Keep behavior identical, no new dependencies, no file moves.
Output: Unified diff only.

```

### Log Summarization Template

```text
Summarize the key errors from this log in under 10 bullet points.
Focus only on root-cause indicators and repeated failures.
Ignore startup noise and successful operations.

```

> **Why it works:** Summarizing large logs first reduces token load for the follow-up debugging step.

### Architecture Question Template

```text
Task: Evaluate two implementation options.
Context: Existing stack is Spring Boot + React.
Goal: Recommend the safer and simpler option for session management.
Constraints: No greenfield redesign, account for existing auth middleware.
Output: 5 bullets max, include trade-offs and recommendation.

```

---

## Developer Workflow

### Phase 1: Start Small

* Open a new session for one task only.
* Provide the task brief.
* Include only the minimum relevant code, config, or logs.

### Phase 2: Keep the Session Clean

* If the assistant starts repeating itself or drifts into unrelated changes, stop and summarize.
* Ask for short outputs by default.
* Do not allow speculative broad rewrites unless you explicitly want redesign work.

### Phase 3: Reset Before Session Rot

* Ask for a summary of decisions and remaining issues.
* Start a new session with that summary.
* Continue only with the current objective and relevant files.

### Team Pattern

For teams, standardize three things:

1. A shared task-brief template.
2. A shared instruction file per tool.
3. A shared rule that long sessions must be summarized and reset instead of endlessly extended.

---

## Tool-Specific Workflows

### Claude Code Playbook

Claude Code relies on **Prompt Caching** to reuse unchanged context prefixes. Certain actions invalidate the cache and force a complete context re-evaluation.

#### Context Control Commands

| Tool / Command | What It Does | Cache Impact |
| --- | --- | --- |
| **Prompt Caching** | Reuses previously processed prompt prefixes automatically. | Saves ~90% input costs on cached tokens. |
| `/recap` | Produces a session summary in terminal output without modifying context. | **Preserves cache prefix** (Preferred). |
| `/compact` | Summarizes history and rebuilds the active session context. | Resets the cached prefix. |
| `/clear` | Wipes conversation history completely. | Rebuilds cache from scratch. |
| `/effort` | Toggles reasoning depth. | **Invalidates cache** (Keyed by effort level). |

#### Practical Rules for Claude Code

* Keep `CLAUDE.md` short (under 100 lines) and placed at the project root.
* Avoid changing `CLAUDE.md`, tool permissions, or effort level mid-session.
* Prefer `/recap` over `/compact` when checking progress to keep your cache warm.

#### Starter `CLAUDE.md`

```markdown
# Project Rules
- Return minimal diffs by default.
- Never modify production configuration without explicit confirmation.
- Prefer existing patterns and libraries over introducing new ones.
- Keep explanations under 3 sentences unless asked for detailed reasoning.

```

---

### Kiro CLI Playbook

Kiro organizes context into three distinct tiers: **Always-on steering**, **On-demand skills**, and **Indexed Knowledge Bases**.

```
┌─────────────────────────────────────────────────────────────┐
│ ALWAYS-ON: AGENTS.md / .kiro/steering/*.md                   │
├─────────────────────────────────────────────────────────────┤
│ ON-DEMAND: Skills.md (Loaded when invoked)                  │
├─────────────────────────────────────────────────────────────┤
│ SEARCHED: /knowledge (RAG index — 0 passive context cost)   │
└─────────────────────────────────────────────────────────────┘

```

#### Official Tools & Context Routing

| Context Category | Best Choice | Why Use It |
| --- | --- | --- |
| **Core Standards** | `AGENTS.md` / Steering Files | Loaded at startup for persistent guidance. |
| **Large Doc Sets** | `/knowledge` Knowledge Base | Indexed on-demand search; consumes no context until queried. |
| **Task Files** | `/context add <file>` | Temporary, session-only context that can be cleared with `/context clear`. |
| **Specialized Guides** | Skills | Loaded into active context only when specifically invoked. |

#### Key Kiro Commands

```bash
# Enable searchable knowledge bases
kiro-cli settings chat.enableKnowledge true

# Add codebase for lexical search (zero passive token cost)
/knowledge add --name "src-code" --path ./src --include "**/*.ts" --index-type Fast

# Inspect active token usage by source
/context show

# Compact when session context becomes large
/compact

```

---

### GitHub Copilot Playbook

Copilot works best when constrained using concise instruction files and explicit context scoping.

#### Official Features & Best Practices

| Tool / Feature | What It Does | Efficiency Benefit |
| --- | --- | --- |
| **Auto Model Selection** | Routes tasks to an appropriate model based on intent. | Avoids running lightweight prompts through costly reasoning models. |
| **Custom Instructions** | Project rules defined in `.github/copilot-instructions.md`. | Persistent guidance across sessions. |
| **Context Scoping** | Direct reference variables like `#file`, `#selection`, or `#editor`. | Prevents unnecessary codebase context loading. |
| **Path-Scoped Rules** | Files like `*.instructions.md` matching specific file patterns. | Loads rules only when matching file types are edited. |

#### Starter `.github/copilot-instructions.md`

```markdown
# Copilot Directives
- Default to minimal unified diffs.
- Do not expand scope beyond the explicitly referenced file or function.
- Prefer existing project patterns and libraries.
- Omit conversational explanations unless requested.

```

---

## Configuration Files Summary

| Tool | Default File Name | Location | Primary Purpose |
| --- | --- | --- | --- |
| **Claude Code** | `CLAUDE.md` | Root or `.claude/` | Project guidelines and cached system rules. |
| **Kiro CLI** | `AGENTS.md` / `.kiro/steering/*.md` | Root / `.kiro/steering/` | Domain-specific steering directives. |
| **GitHub Copilot** | `copilot-instructions.md` | `.github/` | Global workspace coding standards. |

---

## Good vs. Bad Prompting Examples

### ❌ Bad Example

```text
Can you review my auth system and improve it?

```

> **Why it fails:** Too broad, unclear scope, lacks stopping conditions, and invites expensive full-repo analysis.

### ✅ Efficient Example

```text
Task: Review verifyToken() in src/auth/verify.ts.
Goal: Identify the bug causing null sessions.
Constraints: No architecture changes, no new dependencies.
Output: 3 bullets max + minimal patch.

```

> **Why it works:** Narrow scope, explicit goal, constrained output budget, and clear definition of done.

---

## Team Rollout Checklist

* [ ] Add a short instruction file (`CLAUDE.md`, `AGENTS.md`, or `copilot-instructions.md`) to the codebase root.
* [ ] Save the **Universal Task Brief** in team snippet managers.
* [ ] Establish a policy: require patch/diff outputs for routine code edits.
* [ ] Train developers to reset or summarize sessions when context drifts.
* [ ] Utilize searchable knowledge bases or RAG indexes for large repos rather than dumping raw directories into context.

---

## 5-Minute Adoption Plan

1. **Minute 1:** Add a 10-line instruction file to your repository.
2. **Minute 2:** Save the universal task brief prompt snippet.
3. **Minute 3:** Practice requesting `diff only` on your next task.
4. **Minute 4:** Check your active context status (`/context show` in Kiro or `/recap` in Claude Code).
5. **Minute 5:** Start clearing or resetting chats between tasks instead of running multi-issue sessions.
