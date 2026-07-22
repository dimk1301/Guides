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

## Cross-Tool Equivalents

The three tools converge on the same four levers, just with different names. Use this as a quick lookup before diving into the tool-specific sections below.

| Lever | Claude Code | Kiro CLI | GitHub Copilot CLI |
| --- | --- | --- | --- |
| **Reasoning depth control** | `/effort` (toggles reasoning depth; keyed per session, invalidates cache) | `--effort` at launch or `/effort` mid-session — `low / medium / high / xhigh / max`, remembered per model in `chat.modelDefaults` | Configurable reasoning effort for models that support it (e.g. GPT reasoning models); set via `/model`, `--reasoning-effort`, or `COPILOT_MODEL_EFFORT` |
| **Manual compaction** | `/compact` (summarizes and rebuilds context, resets cache) | `/compact` — summarizes older turns while preserving recent messages; tunable via `compaction.excludeMessages` and `compaction.excludeContextWindowPercent` | `/compact` — manual compression anytime; `/context` shows a token-usage breakdown |
| **Automatic compaction** | Not automatic — must be triggered manually via `/compact` or `/clear` | Triggers automatically on context-window overflow, in addition to manual `/compact` | Auto-compacts in the background at ~80–95% of the token limit; creates a recovery checkpoint each time |
| **Scoped file reading (vs. dumping whole files)** | `@file` references pull in only the referenced file/section | Built-in read/grep-style tools pull only the specific lines or matches needed, rather than ingesting whole files or directory trees | `#file`, `#selection`, `#editor` context variables scope references explicitly |
| **Lazy-loaded documentation** | Skills load only when the task matches (progressive loading) | **Skills**: only a short YAML frontmatter description loads at startup; full skill content loads on demand when relevant | Path-scoped `*.instructions.md` files load only for matching file types |
| **Session recovery after compaction** | Original history is gone once compacted | Compaction spins up a *new* session; the untouched original is always reachable via `/chat resume` | Compaction creates a numbered checkpoint file; original detail may not be fully recoverable once summarized |

The practical takeaway: whichever tool you're on, the same rule applies — **tune reasoning depth to the task, compact proactively rather than waiting for a wall, read only what's needed, and let large docs load lazily instead of upfront.**

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
│ ON-DEMAND: Skills (metadata loads at startup, full content   │
│            loads only when the agent needs it)               │
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
| **Specialized Guides** | Skills | Only name + description load at startup; full content loads only when the agent decides the skill is relevant. |
| **File Lookups** | Built-in read/grep tools | Pull specific lines or matches instead of loading whole files or directory trees. |

#### Reasoning Depth: `--effort` / `/effort`

Effort controls how much reasoning the model spends per prompt — lower effort means faster, cheaper, shorter responses; higher effort spends more tokens on deep multi-step reasoning.

| Level | Best for |
| --- | --- |
| `low` | Quick lookups, simple questions |
| `medium` | Standard day-to-day dev tasks |
| `high` | Complex refactoring, architecture decisions |
| `xhigh` | Multi-file changes, nuanced problems |
| `max` | Security reviews, hard debugging, intricate logic |

Set it at launch (`kiro-cli chat --effort high`) or mid-session (`/effort high`). Kiro remembers your choice automatically for future sessions via `~/.kiro/settings/cli.json`, and you can set different default effort levels per model under `chat.modelDefaults` (e.g., keep Sonnet at `high` but Opus at `max`). Practical rule: bump down for quick lookups, bump up only when the agent is missing edge cases or the task is genuinely hard.

#### Context Compaction: `/compact`

Kiro's direct answer to Claude Code's `/compact`. It summarizes older messages while preserving key information and recent turns, freeing up context window space. It also triggers **automatically** if your context window overflows, so you're not forced to babysit it. Two settings tune retention precisely:

* `compaction.excludeMessages` — minimum number of message pairs to always keep uncompacted.
* `compaction.excludeContextWindowPercent` — minimum percentage of the window to retain.

Compaction spins up a *new* session — jump back to the original untouched history anytime with `/chat resume`.

#### Selective Indexing via Skills + Knowledge Bases

This is where Kiro pulls ahead of most competitors on documentation cost. Skills use progressive context loading: instead of loading full documentation upfront, the agent reads only a short YAML frontmatter description of each skill file, and loads the full content only when it determines that skill is actually relevant to the current task. Combined with `/knowledge` for persistent, searchable storage, this gives a two-layer system: broad project facts live as lightweight, lazily-loaded skill descriptions, and deep documentation only gets pulled in on-demand rather than repeated every prompt.

#### Key Kiro Commands

```bash
# Enable searchable knowledge bases
kiro-cli settings chat.enableKnowledge true

# Add codebase for lexical search (zero passive token cost)
/knowledge add --name "src-code" --path ./src --include "**/*.ts" --index-type Fast

# Set an initial effort level at launch
kiro-cli chat --effort high

# Adjust reasoning effort mid-session
/effort medium

# Inspect active token usage by source
/context show

# Compact proactively before a big context-heavy task
/compact

# Return to the untouched pre-compaction session
/chat resume

```

#### Practical Rules for Kiro CLI

* Set `--effort medium` as your default at launch; reserve `high`/`max` for genuinely hard problems.
* Let automatic compaction handle overflow, but run `/compact` proactively before a big context-heavy task rather than waiting for the automatic trigger.
* Rely on built-in read/grep-style tools for file lookups instead of asking the agent to "explore the codebase."
* Structure large documentation as Skills with clear YAML descriptions so Kiro only pulls full content in when relevant.
* Remember compaction creates a new session — if you're mid-debug and need the full original context back, `/chat resume` is your safety net, not trying to undo a compaction.

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
| **Reasoning Effort (Copilot CLI)** | Configurable reasoning effort for supported reasoning models, set via `/model`, a reasoning-effort flag, or an environment variable. | Balances response speed against reasoning depth per task. |
| **Auto-Compaction (Copilot CLI)** | Automatically compresses history in the background at roughly 80–95% of the token limit; `/compact` also available manually. | Enables long sessions without manual cleanup; `/context` shows the token-usage breakdown. |

#### Practical Rules for GitHub Copilot CLI

* Avoid switching models or reasoning-effort levels mid-session — it can break cache reuse and force a rebuild.
* Use `/context` to check usage before deciding whether to compact manually.
* Prefer `#file` / `#selection` references over asking Copilot to explore the whole repo.

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
* [ ] Set a default reasoning-effort level per tool (Claude Code `/effort`, Kiro `--effort`, Copilot reasoning effort) and reserve the top tier for genuinely hard tasks.
* [ ] Prefer proactive manual compaction over waiting for automatic triggers when a big context-heavy task is coming up.

---

## 5-Minute Adoption Plan

1. **Minute 1:** Add a 10-line instruction file to your repository.
2. **Minute 2:** Save the universal task brief prompt snippet.
3. **Minute 3:** Practice requesting `diff only` on your next task.
4. **Minute 4:** Check your active context status (`/context show` in Kiro, `/recap` in Claude Code, or `/context` in Copilot CLI).
5. **Minute 5:** Start clearing or resetting chats between tasks instead of running multi-issue sessions.
