# AI Token Efficiency Playbook for Developers

A practical guide for reducing token usage and improving output quality in **Claude Code**, **Kiro CLI**, and **GitHub Copilot CLI**. Token efficiency comes down to keeping context small and relevant, choosing the right model for the task, and constraining both prompts and outputs so the model processes only what is needed.

> **Commands verified as of August 2026.** All three tools ship updates frequently — exact flags, thresholds, and slash commands can change between releases. Run `--help` or check official docs before relying on an exact flag name in this guide. Where a feature is explicitly marked **Experimental** by its vendor (e.g., Kiro's `/knowledge`), that status is called out below — expect rougher edges and possible breaking changes between releases.

**Jump to:** [Core Mental Model](#core-mental-model) · [Progressive Knowledge Indexing](#progressive-knowledge-indexing) · [10 Golden Rules](#10-golden-rules) · [Good vs. Bad Examples](#good-vs-bad-prompting-examples) · [Prompt Templates](#copy-paste-prompt-templates) · [Developer Workflow](#developer-workflow) · [When to Stop Optimizing](#when-to-stop-optimizing) · [Cross-Tool Equivalents](#cross-tool-equivalents) · [Claude Code](#claude-code-playbook) · [Kiro CLI](#kiro-cli-playbook) · [GitHub Copilot CLI](#github-copilot-cli-playbook) · [Config Files](#configuration-files-summary) · [Third-Party Tools](#optional-third-party-tools) · [Troubleshooting](#troubleshooting) · [Checklist](#team-rollout-checklist) · [Cheat Sheet](#one-page-cheat-sheet) · [5-Minute Plan](#5-minute-adoption-plan)

---

## Core Mental Model

Think of an LLM request like a function call:

```text
output = model(instructions + history + code + logs + tool_output)

```

LLMs are effectively stateless across turns, so each new request is evaluated against the full active context window rather than a compact internal memory. That means every extra message, file, log block, and repeated instruction is paid for again on later turns, which increases cost and often reduces answer quality once the context becomes noisy.

### Agentic Loops Make This Worse Than It Looks

A standard chatbot mostly pays for what it *says* — output tokens, generated step-by-step, are the expensive part of a normal conversation. Coding agents flip this. Each loop iteration — read context, call a tool, append the result, decide the next step — resends the *entire* prior history as input, because the model carries no memory between calls:

```text
[Context Window] → [LLM Evaluates] → [Tool Called] → [Result Appended] → [Repeat]

```

Because history is resent in full on every turn, the *cumulative* input volume across a session grows much faster than the number of turns — a 50-turn debugging session isn't 50x the cost of a 1-turn question, it's worse, since turn 50 re-pays for everything said in turns 1–49. In practice this means the large majority of an agent's bill comes from input tokens rather than output. That's the real reason Rules 1, 3, 7, and 9 below (fresh sessions, minimal snippets, proactive resets, inspecting context) matter more for agentic tools than they would for a one-shot chatbot question.

> Exact input/output cost splits vary by task and provider — treat "input dominates" as the general shape of agentic costs, not a fixed percentage to cite.

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

### Why Token Count ≠ Word Count

Billing and context limits are based on tokens, not words, and the split isn't intuitive:

* **Punctuation and symbols usually cost their own token.** A comma, brace, or colon typically isn't free just because it's short.
* **Code and structured formats (JSON, YAML) tokenize less efficiently than plain prose.** All the braces, quotes, and colons around a handful of meaningful words add real overhead — the same information expressed as prose is usually cheaper.
* **Rough estimate for sanity-checking:** plain English runs about ~4 characters per token; code runs closer to ~3 characters per token (more symbols, less compression). Exact ratios vary by model and tokenizer, so treat this as a quick gut-check, not a billing calculation — run an actual tokenizer on representative samples before projecting costs for a new feature.

This is the mechanism behind several of the Golden Rules below: pasting minimal snippets (Rule 3), asking for diffs instead of full rewrites (Rule 4), and summarizing logs before debugging all work specifically because code, structured data, and verbose output are the most token-expensive things you can put in context — cutting them helps more than cutting an equivalent amount of plain prose would.

### A Quick Note on Images and Other Non-Text Input

If you're pasting screenshots of errors, UI diffs, or architecture diagrams into a session, know that image tokens are counted and billed differently from text — typically derived from image dimensions rather than something you can eyeball the way you can with a paragraph. The same "minimal snippet" instinct from Rule 3 still applies: crop to the relevant region instead of pasting a full-screen screenshot, and don't leave large images sitting in a long-running session's history once you're done referencing them — they get resent on every subsequent turn just like everything else.

### How Prompt Caching Breaks (and How to Keep It Intact)

All three tools get an automatic discount on tokens that hit the cache. For Claude models this is well-documented: a cache hit costs 10% of the standard input price — a 90% discount (see the *Prompt Caching* row in the Claude Code Context Control Commands table below). Kiro CLI and Copilot CLI pass through comparable caching economics from their underlying model providers, but the exact discount, minimum cacheable size, and cache lifetime depend on which model you've selected inside each tool — don't assume the Claude numbers transfer 1:1 to a GPT or Gemini model running inside Kiro or Copilot.

That discount only applies if the new prompt shares an exact, unbroken prefix with something already cached. Models process prompts strictly top-to-bottom: if a single character changes at token 10 of a 50,000-token prompt, everything from token 10 onward has to be recomputed, even though the rest of the prompt is unchanged. Think of it as a sandwich — the cache only holds as long as the bread (the fixed top portion) stays exactly the same.

The practical fix is to **order context from static to volatile**:

1. **Top (rarely changes):** System instructions, `CLAUDE.md` / `AGENTS.md` / steering rules, tool and schema definitions.
2. **Middle (append-only):** Conversation history and prior tool outputs — new turns get added, old ones aren't edited.
3. **Bottom (changes every turn):** Timestamps, live session metadata, current file contents, anything genuinely dynamic.

This is also *why* editing `CLAUDE.md`, switching models, or changing effort level mid-session forces a full cache rebuild in the tool-specific sections below — those live at the top of the stack, so any change invalidates everything cached beneath them.

---

## Progressive Knowledge Indexing

*(This is a term coined for this guide, not vendor terminology — you won't find it by this name in any tool's docs. The underlying pattern is real, though, and maps directly to what each tool calls Skills, `/knowledge`, or path-scoped instructions below.)*

Most token waste from documentation comes from an all-or-nothing choice: either load every doc/skill/instruction file into context "just in case," or don't give the model access at all. Progressive knowledge indexing is the middle path, and it works the same way across tools even though the names differ:

1. **Metadata-only at startup.** Only a name and short description of each doc/skill loads at session start — essentially a table of contents. This costs next to nothing.
2. **Full content loaded on demand.** The agent pulls in the full text of a doc only when your current prompt actually matches its description.
3. **Searched, not loaded, for large corpora.** For big doc sets or whole codebases, nothing sits in context at all — it's indexed and queried like search, only when needed.

The result: context cost scales with what's *relevant* to the task in front of you, not with the total size of your documentation.

**Where to apply it:** treat it as the deciding line between two buckets —

* **Always-needed, small rules** → put these in your always-on instruction file (`CLAUDE.md`, `AGENTS.md`/steering, `copilot-instructions.md`). These load every turn by design, so keep them short.
* **Large or situational reference material** → structure it for progressive loading instead of always-on. See the tool-specific mechanics in each Playbook section below:
  - **Kiro CLI** — Skills (metadata-only until relevant) + `/knowledge` (an **Experimental** feature, must be enabled) for indexed, searchable doc sets. See *Selective Indexing via Skills + Knowledge Bases*.
  - **Claude Code** — Skills, loaded only when the task matches the skill's description.
  - **GitHub Copilot CLI** — Path-scoped `*.instructions.md`, which loads by file-type match rather than semantic relevance — a coarser proxy for the same idea, not a full equivalent.

The common failure mode is a vague skill/doc description: too vague and the agent either never triggers it or triggers it for everything, which cancels out the savings. Write descriptions as specific as the task they should match.

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
8. **Use configuration files:** Store persistent rules in tool-specific instruction files instead of repeating them every time. Keep them static — editing them mid-session breaks the cache (see *How Prompt Caching Breaks*, above).
9. **Inspect active context:** Use built-in context tools to inspect, compact, or clear context rather than guessing.
10. **Index large repos:** Prefer indexed or searchable knowledge stores for big codebases and docs instead of loading them permanently into context.

*(These are levers, not a checklist to run on every prompt — see [When to Stop Optimizing](#when-to-stop-optimizing) below for when applying them isn't worth the overhead.)*

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

For teams, standardize four things:

1. A shared task-brief template.
2. A shared instruction file per tool.
3. A shared rule that long sessions must be summarized and reset instead of endlessly extended.
4. A budget or turn-count cap for autonomous/agentic runs, so a runaway loop fails loudly and cheaply instead of quietly burning tokens.

### Monitoring Token Usage

The built-in `/context` commands (see the Cross-Tool Equivalents table below) are enough for day-to-day use. If you need cost visibility across a team, or are building your own tooling around these agents, track raw token counters rather than dollar totals — they're the more stable unit as pricing changes. Five numbers cover it per request or loop iteration:

* `input_tokens` — uncached input, full price
* `cache_read_tokens` — cached input, heavily discounted
* `cache_write_tokens` — one-time fee to populate the cache
* `output_tokens` — generated text
* `thinking_tokens` — reasoning/extended-thinking tokens, where applicable

A high `input_tokens`-to-`cache_read_tokens` ratio over time is usually the clearest sign that the context-ordering advice above, or Rules 1 and 7, aren't being followed in practice.

**Mapping counters to dollars, roughly:** if you need an actual budget number rather than just a trend line, multiply each counter by your model's current per-token rate — input, cache read, cache write, and output are usually four different rates, and the ratios between them change more often than the raw numbers do, so pull current rates from your provider's pricing page rather than hardcoding them. If you're building a dashboard or spreadsheet around this, add a reminder to refresh the rates; stale per-token prices are the most common source of wrong budget projections on long-lived internal tools.

---

## When to Stop Optimizing

Token efficiency has diminishing returns, and over-applying these rules can cost more in developer time than it saves in tokens or dollars.

* **Don't fragment a task just to keep a session "clean."** If you're mid-debug and genuinely need the full history to reason about a regression, restarting the session and re-explaining everything from scratch can burn more tokens — and more of your time — than just letting the session run long and compacting once.
* **Don't hand-craft a full task brief for a one-line question.** The Universal Task Brief template is for substantive work; writing four lines of constraints to ask "what does this regex do" is optimization theater.
* **Don't drop to the cheapest model for something that actually needs reasoning.** A wrong answer from a fast/cheap model that you have to catch, correct, and re-prompt almost always costs more total tokens — and time — than getting it right once with a stronger model.
* **Don't obsess over `/context` percentages mid-task.** Checking usage is useful at natural breakpoints; checking it every few turns is its own overhead and a distraction from the actual work.
* **Weigh your time, not just the token bill.** If a team optimizes to save a few cents of API cost while an engineer spends ten minutes reformatting prompts, the optimization is net-negative. These rules pay off most on repeated, large-scale, or team-wide usage — not on every single interaction.

Treat the rest of this guide as levers to pull when the task and context genuinely warrant it, not as a checklist to run against every prompt.

---

## Cross-Tool Equivalents

The three tools converge on the same five levers, just with different commands. This table is a scannable index only — full mechanics, config keys, and caveats live in each tool's own Playbook section below, so look here first, then jump to your tool.

| Lever | Claude Code | Kiro CLI | GitHub Copilot CLI |
| --- | --- | --- | --- |
| **Reasoning depth control** | `/effort` | `--effort` (launch) / `/effort` (mid-session) | Reasoning effort via `/model` picker |
| **Manual compaction** | `/compact` | `/compact` | `/compact` |
| **Automatic compaction** | None — manual only | On context overflow | On approaching context limit |
| **Scoped file reading** | `@file` references | Built-in read/grep tools | `#file`, `#selection`, `#editor` |
| **Lazy-loaded documentation** | Skills | Skills + `/knowledge` *(Experimental)* | Path-scoped `*.instructions.md` (coarser — file-type match, not relevance) |
| **Non-destructive usage check** | `/context` | `/context show` | `/context` |
| **Autonomous-run budget cap** | Not native — enforce via a wrapper/turn count you track yourself | Not native — same | Native: per-session AI-credit limit, stops cleanly and asks before exceeding |

The practical takeaway: whichever tool you're on, the same rule applies — **tune reasoning depth to the task, compact proactively rather than waiting for a wall, read only what's needed, and let large docs load lazily instead of upfront.** Details and exact syntax: see each tool's Playbook section.

---

## Tool-Specific Workflows

### Claude Code Playbook

Claude Code relies on **Prompt Caching** to reuse unchanged context prefixes. Certain actions invalidate the cache and force a complete context re-evaluation.

#### Context Control Commands

| Tool / Command | What It Does | Cache Impact |
| --- | --- | --- |
| **Prompt Caching** | Reuses previously processed prompt prefixes automatically. | Saves ~90% on input costs (cache reads are billed at 10% of standard input price), but only for tokens that hit the cache. |
| `/context` | Shows a visual breakdown of current context usage without modifying it. | **Preserves cache prefix** (Preferred for checking progress). |
| `/compact` | Summarizes history and rebuilds the active session context. | Resets the cached prefix. |
| `/clear` | Wipes conversation history completely. | Rebuilds cache from scratch. |
| `/effort` | Sets reasoning depth (`low` through `xhigh`/`max`). | Likely affects cache reuse, since effort level is part of the request configuration — avoid changing it mid-task. |

#### Practical Rules for Claude Code

* Keep `CLAUDE.md` short (under ~200 lines) and placed at the project root; move details into separate files and pull them in with `@filename`.
* Avoid changing `CLAUDE.md`, tool permissions, or effort level mid-session — each can force a cache rebuild (see *How Prompt Caching Breaks*, above, for why).
* Prefer `/context` over `/compact` when you just want to check usage, to keep your cache warm.
* Use `/clear` between unrelated tasks; use `/compact` when continuing the same task with less context pressure.
* Caching has a minimum cacheable prefix size — a two-line `CLAUDE.md` or short system prompt may sit below that floor and never actually hit the cache. Don't expect savings on trivially small always-on instructions; the payoff shows up on larger, stable prefixes (tool definitions, longer steering files, loaded documents).
* Cache breakpoints are limited per request. If you're layering system prompt + tool definitions + a large loaded document + conversation history, keep the boundaries between them in static-to-volatile order rather than leaving the split to be inferred.

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
│ SEARCHED: /knowledge — Experimental (RAG index — 0 passive   │
│           context cost once enabled)                         │
└─────────────────────────────────────────────────────────────┘

```

#### Official Tools & Context Routing

| Context Category | Best Choice | Why Use It |
| --- | --- | --- |
| **Core Standards** | `AGENTS.md` / Steering Files | Loaded at startup for persistent guidance. |
| **Large Doc Sets** | `/knowledge` Knowledge Base *(Experimental — must be enabled)* | Indexed on-demand search; consumes no context until queried. |
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

Set it at launch (`kiro-cli chat --effort high`, a shell command) or mid-session (`/effort high`, typed inside the interactive chat REPL). Kiro remembers your choice automatically for future sessions via `~/.kiro/settings/cli.json`, and you can set different default effort levels per model under `chat.modelDefaults` (e.g., keep Sonnet at `high` but Opus at `max`). You can also drop a `.kiro/settings/cli.json` in a project root to set workspace-level defaults for everyone working in that repo — Kiro resolves effort in this priority order: session override → workspace defaults → user defaults → model's built-in default. Practical rule: bump down for quick lookups, bump up only when the agent is missing edge cases or the task is genuinely hard.

#### Context Compaction: `/compact`

Kiro's direct answer to Claude Code's `/compact`. It summarizes older messages while preserving key information and recent turns, freeing up context window space. It also triggers **automatically** if your context window overflows, so you're not forced to babysit it. Two settings tune retention precisely, and Kiro applies whichever is more conservative (larger) when they disagree:

* `compaction.excludeMessages` — minimum number of message pairs to always keep uncompacted.
* `compaction.excludeContextWindowPercent` — minimum percentage of the window to retain.

Compaction spins up a *new* session — jump back to the original untouched history anytime with `/chat resume`. Compaction is one-way within a session: if you need the full pre-compaction history for something other than resuming, export it before compacting.

#### Selective Indexing via Skills + Knowledge Bases

This is where Kiro pulls ahead of most competitors on documentation cost. Skills use progressive context loading: instead of loading full documentation upfront, the agent reads only a short YAML frontmatter description of each skill file, and loads the full content only when it determines that skill is actually relevant to the current task. Combined with `/knowledge` for persistent, searchable storage, this gives a two-layer system: broad project facts live as lightweight, lazily-loaded skill descriptions, and deep documentation only gets pulled in on-demand rather than repeated every prompt.

Note that `/knowledge` is currently labeled **Experimental** in Kiro's own documentation — it has to be turned on explicitly before it's available (see the `kiro-cli settings` command below), and as with any experimental feature, treat it as more likely to change behavior or break between releases than the stable Skills mechanism.

#### Key Kiro Commands

> Flag and config-key names below match current docs as of this writing — Kiro ships frequently, so confirm with `kiro-cli --help` or `/help` before scripting against them.

```bash
# Enable searchable knowledge bases (Experimental feature — off by default)
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
* Treat `/knowledge` as Experimental in practice, not just in name — enable it deliberately, and don't build a critical workflow around it without a fallback (Skills, or plain file references) in case behavior changes between releases.

---

### GitHub Copilot CLI Playbook

Copilot CLI works best when constrained using concise instruction files and explicit context scoping. Note: the reasoning-effort and auto-compaction features below are specific to Copilot **CLI** — behavior may differ in the IDE/editor extension.

#### Official Features & Best Practices

| Tool / Feature | What It Does | Efficiency Benefit |
| --- | --- | --- |
| **Auto Model Selection** | Routes tasks to an appropriate model based on intent. | Avoids running lightweight prompts through costly reasoning models. |
| **Custom Instructions** | Project rules defined in `.github/copilot-instructions.md`. | Persistent guidance across sessions. |
| **Context Scoping** | Direct reference variables like `#file`, `#selection`, or `#editor`. | Prevents unnecessary codebase context loading. |
| **Path-Scoped Rules** | Files like `*.instructions.md` matching specific file patterns. | Loads rules only when matching file types are edited. |
| **Reasoning Effort (Copilot CLI)** | For reasoning models that support it, open the `/model` picker, select the model, then pick a level from the "Thinking Effort" submenu. | Balances response speed against reasoning depth per task. |
| **Auto-Compaction (Copilot CLI)** | Automatically compresses history in the background as the session approaches the context limit (current docs cite ~80%, with earlier releases at ~95% — this threshold has moved between versions); `/compact` also available manually. | Enables long sessions without manual cleanup; `/context` shows the token-usage breakdown. |
| **Session AI-Credit Limit** | Cap the amount of work Copilot performs on a single session before it stops and asks. | Direct, native implementation of Golden Rule 10's "budget cap" for autonomous/long-running runs. |

#### Practical Rules for GitHub Copilot CLI

* Avoid switching models or reasoning-effort levels mid-session — it can break cache reuse and force a rebuild.
* Use `/context` to check usage before deciding whether to compact manually.
* Prefer `#file` / `#selection` references over asking Copilot to explore the whole repo.
* Cache lifetime is provider-dependent, not fixed: roughly 24 hours of inactivity for OpenAI-hosted models, about 1 hour for most others. If you're returning to an old session after a break, starting fresh — or running `/compact` so what rebuilds is a short summary rather than the full history — is often cheaper than hoping the cache is still warm.
* Set a session AI-credit limit before kicking off a long autonomous run rather than relying on manual attention to notice it's spinning.

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
| **GitHub Copilot CLI** | `copilot-instructions.md` | `.github/` | Global workspace coding standards. |

> **Before committing:** these are plain text files that typically live in the repo and get committed to version control. Don't put credentials, internal URLs, customer data, or anything else you wouldn't want sitting in `git log` into them — treat them with the same review care as any other source file, not as a private scratchpad.

---

## Optional Third-Party Tools

Reach for anything in this section only after you've implemented the native rules above and identified a *specific* remaining bottleneck — logs/tool-output volume is still your dominant cost driver, or large-codebase exploration alone is expensive. These are solutions to a diagnosed problem, not a first step.

Everything above this section is a native, vendor-shipped feature of Claude Code, Kiro CLI, or Copilot CLI. What follows is a mix of community/commercial projects that sit on top of those tools, plus one emerging open spec (OKF) — none of them ship natively inside these three CLIs, so vet each one (data flow, maintenance activity, added overhead, maturity) before a team-wide rollout. Treat this section as "things to evaluate," not "things to adopt by default."

| Tool | What it does | Which tool(s) it plugs into | Key caveat |
| --- | --- | --- | --- |
| **Headroom** | A proxy/library/MCP server that compresses tool outputs, logs, files, and RAG chunks before they reach the model. Vendor claims 60–95% token reduction. | Claude Code, Codex, Copilot, Gemini, Bedrock, Vertex | Works by rerouting your API base URL through a local proxy — review what it caches/logs locally before wiring it into a shared team config. |
| **KiroGraph** | An MCP server exposing a semantic, tree-sitter-based code knowledge graph — symbol lookups, call graphs, and impact analysis in one query instead of several read/grep tool calls. | Kiro CLI (full support); ~34 other MCP-capable tools (experimental) | Enabling every optional module adds several thousand tokens of tool definitions to *every* call (exact figures vary by version) — enable only the modules you actually use, or the tool overhead can outweigh the savings. |
| **Graphify** | A `/graphify` skill that builds a local, tree-sitter-based knowledge graph of code, docs, PDFs, and more; the agent queries the graph (`graphify query`, `graphify path`) instead of reading or grepping files. Because it replaces raw file/log reads with small queried graph nodes, it shrinks the underlying input volume itself — a saving that stacks with prompt caching rather than depending on it. | Claude Code, Copilot CLI, Cursor, Codex, Gemini CLI, and 15+ more | Code parsing is fully local and free (no LLM call). Docs/PDF/image ingestion routes through your assistant's model API, so it isn't zero-cost for non-code content. |
| **OKF (Open Knowledge Format)** | An open, vendor-neutral spec from Google Cloud (v0.1, launched June 2026) for representing curated agent context as a directory of markdown files with YAML frontmatter — one concept per file, cross-linked, file path as identity. | Format, not a tool — consumable by any agent that can read files; not tied to a specific CLI | Very new (v0.1) and still evolving. Unlike Skills or `/knowledge`, nothing loads full content only-when-relevant by default — you still need to design (or bolt on) your own progressive-loading layer on top of an OKF bundle. |

### Where each one fits into this playbook

* **Headroom** is the closest thing to an automated version of Golden Rules 3 and 5 (minimal snippets, output limits) — it compresses what's already flowing through the pipe rather than requiring you to hand-curate it. Consider it if your logs/tool-output volume is the dominant cost driver and manual summarization isn't scaling.
* **KiroGraph** extends the *Selective Indexing via Skills + Knowledge Bases* pattern in the Kiro Playbook above: instead of Kiro's agent reading files to build understanding, the graph is pre-built and queried. Good fit if you're already leaning on Kiro's Skills/`/knowledge` split and want the codebase itself indexed the same way.
* **Graphify** fills the gap in the *Progressive Knowledge Indexing* section's "searched, not loaded, for large corpora" tier for **Claude Code and Copilot CLI** specifically — right now that tier has a native answer in Kiro (`/knowledge`) but not in the other two. If your team's biggest token sink is large-codebase exploration (not logs or docs), this is the most directly applicable of the three.
* **OKF** is less a tool and more a portable version of the idea behind Skills and `AGENTS.md`/steering files: one markdown file per concept, machine-readable frontmatter, human-readable body. The difference is portability — an OKF bundle isn't tied to Claude Code, Kiro, or Copilot specifically, so it's worth piloting if you want one knowledge base that several agents/tools can read without reformatting per tool. Being v0.1, treat it as something to trial on a non-critical doc set, not something to standardize your team's whole knowledge layer on yet.

None of these replace the native commands in the Cross-Tool Equivalents table — they're additive, and each introduces its own dependency (a local proxy, an MCP server, a generated graph file, or a new context format) that a team needs to own and update going forward.

---

## Troubleshooting

A short list of the problems people actually hit, and where to look first.

**"My context is huge and I don't know why."**
Run the usage-inspection command for your tool (`/context` in Claude Code and Copilot CLI, `/context show` in Kiro) before doing anything else — guessing wastes more time than checking. Common culprits: too many optional MCP modules/tool definitions enabled (see the KiroGraph caveat above), a long conversation history that hasn't been compacted, or a file you loaded early in the session and forgot was still there.

**"Caching doesn't seem to be working — costs aren't dropping."**
The most common cause is a broken prefix: something at the top of your context changed between requests (a timestamp, a reordered tool list, a mid-session model or effort-level switch, or inconsistent formatting). Re-check *How Prompt Caching Breaks* above and the static-to-volatile ordering. Also confirm you're not just between cache windows — Claude's default ephemeral cache lasts a few minutes, and Copilot CLI's provider-dependent cache windows range from about an hour to about a day of inactivity before they expire — so a long pause between messages can look like "caching stopped working" when it's actually just expired.

**"Auto-compaction fired and now the agent seems to have forgotten something important."**
This is closer to an expected tradeoff than a bug: compaction deliberately compresses intermediate tool output and exploratory discussion first, and is generally one-way once it happens. If something needs to definitely survive, state it explicitly as a decision or constraint before compaction happens, rather than leaving it buried in a tool result. Know your tool's recovery path — Kiro's `/chat resume` returns to the pre-compaction session; check whether your version of Claude Code or Copilot CLI offers an equivalent before assuming history is gone for good.

**"An agentic loop seems to be spinning / burning tokens with no progress."**
This is what the budget/turn-count cap in the Team Pattern section is for. If you don't have one set yet, that's the first fix — Copilot CLI has this natively via session AI-credit limits; Claude Code and Kiro currently need it enforced externally. In the moment, stop the session rather than letting it continue, and start a fresh one with a narrower task brief once you understand what it got stuck on.

---

## Team Rollout Checklist

* [ ] Add a short instruction file (`CLAUDE.md`, `AGENTS.md`, or `copilot-instructions.md`) to the codebase root. *(highest impact — do this first)*
* [ ] Save the **Universal Task Brief** in team snippet managers. *(highest impact — do this second)*
* [ ] Confirm instruction files don't contain secrets, internal URLs, or sensitive data before they're committed — they're plain tracked files, not a private scratchpad.
* [ ] Establish a policy: require patch/diff outputs for routine code edits.
* [ ] Train developers to reset or summarize sessions when context drifts.
* [ ] Utilize searchable knowledge bases or RAG indexes for large repos rather than dumping raw directories into context.
* [ ] Set a default reasoning-effort level per tool (Claude Code `/effort`, Kiro `--effort`, Copilot reasoning effort) and reserve the top tier for genuinely hard tasks.
* [ ] Prefer proactive manual compaction over waiting for automatic triggers when a big context-heavy task is coming up.
* [ ] Order any custom context injection (RAG chunks, retrieved docs, injected snippets) static-to-volatile so it doesn't silently break prompt caching.
* [ ] Set a budget cap or max-turn limit for autonomous agent runs, so a runaway loop fails safely instead of burning tokens silently.
* [ ] If tracking spend across the team, log raw token counters (input / cache_read / cache_write / output / thinking) rather than relying on dollar totals alone.
* [ ] If evaluating a third-party tool or spec (Headroom, KiroGraph, Graphify, OKF, or similar), review its data flow, update cadence, and maturity before rolling it out beyond a single volunteer's machine.

---

## One-Page Cheat Sheet

Everything else in the guide explains *why* these work — this is the scannable version.

**Before you start a task**
- New session for a new task (Rule 1).
- Task brief: task / files / constraints / output format (Rule 2).
- Pick your effort/reasoning level for the task, not just the default (Rule 6).

**While you work**
- Paste snippets, not whole files or full logs (Rule 3).
- Ask for diffs/patches, not rewrites (Rule 4).
- Add an output limit — word count, bullet count, "patch only" (Rule 5).
- Don't edit your instruction file or switch models/effort mid-session — both break the cache.

**When it gets messy**
- Check usage first, don't guess: `/context` (Claude Code, Copilot CLI) or `/context show` (Kiro).
- Summarize and reset before the session rots (Rule 7), or compact proactively before a big context-heavy task (Rule 10).
- Set a turn/budget cap on autonomous runs so a runaway loop fails loudly, not silently.

**Command quick-reference**

| | Claude Code | Kiro CLI | Copilot CLI |
| --- | --- | --- | --- |
| Check usage | `/context` | `/context show` | `/context` |
| Compact | `/compact` (manual only) | `/compact` (also automatic) | `/compact` (also automatic, ~80% currently) |
| Set effort | `/effort` | `--effort` / `/effort` | `/model` picker → Thinking Effort |
| Scoped file read | `@file` | built-in read/grep | `#file` / `#selection` / `#editor` |
| Lazy docs | Skills | Skills + `/knowledge` *(Experimental)* | `*.instructions.md` (path-scoped) |
| Wipe history | `/clear` | new session | new session |
| Budget cap | manual/external | manual/external | native session AI-credit limit |

**Don't:**
- Don't fragment tiny questions into a full task-brief ritual — see [When to Stop Optimizing](#when-to-stop-optimizing).
- Don't commit secrets into `CLAUDE.md` / `AGENTS.md` / `copilot-instructions.md` — they're regular tracked files.
- Don't cite a vendor-marketing token-reduction percentage without checking the source and framing it as a claim, not a fact.

---

## 5-Minute Adoption Plan

1. **Minute 1:** Add a 10-line instruction file to your repository.
2. **Minute 2:** Save the universal task brief prompt snippet.
3. **Minute 3:** Practice requesting `diff only` on your next task.
4. **Minute 4:** Check your active context status (`/context show` in Kiro, `/context` in Claude Code, or `/context` in Copilot CLI).
5. **Minute 5:** Start clearing or resetting chats between tasks instead of running multi-issue sessions.
