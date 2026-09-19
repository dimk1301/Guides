# Kiro CLI: Practical Guide

Commands, workflow, customization (AGENTS.md, skills, agents), and saving credits. Based on the Kiro docs as of September 2026. Your company may run an older CLI (2.x) or restrict features, so check with `kiro-cli version` and `/help`. Behavior differs between 2.x and 3.x; where docs disagree, verify with `/context show`.

---

## 1. Getting started

```bash
cd my-project
kiro-cli            # opens the terminal UI chat
kiro-cli login      # if auth fails or expires
kiro-cli doctor     # diagnose common problems
kiro-cli --v3       # opt in to CLI 3.0 (runs alongside 2.x)
```

Type `/guide` inside a session to talk to the built-in Guide agent. It knows the docs for your installed version and can generate agents, prompts, and steering files. `Shift+Tab` returns to your main agent.

---

## 2. Commands

### Terminal commands

| Command | Purpose |
|---|---|
| `kiro-cli chat "question"` | Chat with a first prompt |
| `kiro-cli chat --resume` / `--resume-picker` | Continue last session / choose one |
| `kiro-cli chat --list-sessions` | Sessions for this directory |
| `kiro-cli chat --list-models` | Available models and multipliers |
| `kiro-cli chat --agent <name>` | Start with a custom agent |
| `kiro-cli chat --effort low` | Set reasoning effort at launch |
| `kiro-cli chat --no-interactive "..."` | One-shot output (scripting) |
| `kiro-cli translate "find py files changed this week"` | Natural language to shell command |
| `kiro-cli agent list / create / edit` | Manage agents |
| `kiro-cli mcp list / add / status` | Manage MCP servers |
| `kiro-cli settings list` | View/change settings |

### In-session slash commands

| Command | Purpose |
|---|---|
| `/help`, `/guide` | Command list / ask Kiro about Kiro |
| `/context` (`/context show`) | What is loaded and token usage |
| `/usage` | Usage limits and credits |
| `/model`, `/effort` | Switch model / reasoning depth |
| `/agent swap <name>` | Switch agents |
| `/chat new`, `/chat resume`, `/chat save <file>` | Session management |
| `/compact` | Summarize history to free context |
| `/tangent` | Branch a side conversation, then return |
| `/spec new <name>` | Spec-driven development |
| `/tools`, `/mcp`, `/hooks`, `/prompts` | Inspect tools, MCP servers, hooks, prompts |

### Shortcuts and syntax

- `@path/to/file` references a file (tab completion)
- `!npm test` runs a shell command without the AI (no credits)
- `Shift+Tab` toggles Plan mode; `Shift+Enter` new line; `Esc` cancels

---

## 3. Best practices

- **Plan first.** Plan mode (`Shift+Tab`) outlines the approach before any edits. For larger features use `/spec new` (requirements, design, tasks).
- **Be specific.** Name files with `@`, state the goal and constraints, and say how to verify ("run the tests"). Vague prompts cause wandering exploration, which costs credits.
- **Write project instructions once** (section 4) instead of retyping conventions.
- **Use custom agents** for recurring jobs like code review or test writing.
- **Approve deliberately.** Kiro asks before writing files or running commands. Trust routine patterns (e.g. `npm run`), not everything. In 3.x, trust flags are replaced by capability rules in `permissions.yaml`.
- **Commit before big agent runs** so you can diff and revert.
- **Don't paste secrets** and follow your company's AI usage policy.

Example flow: `kiro-cli` → `/context` → Plan mode with `@files` → approve → implement at `/effort medium` → run `!pnpm test` yourself and paste only failures back → `/compact` when history grows → `/chat new` for the next task.

---

## 4. Customization

### Claude Code to Kiro mapping

| Claude Code | Kiro CLI |
|---|---|
| `CLAUDE.md` | `AGENTS.md`, or `.kiro/steering/*.md` |
| `~/.claude/CLAUDE.md` | `~/.kiro/steering/` |
| `.claude/skills/<name>/SKILL.md` | `.kiro/skills/<name>/SKILL.md` |
| `.claude/agents/*.md` | `.kiro/agents/*.md` or `*.json` |
| Custom slash commands | Prompts (`/prompts`, `@name`) and skills |
| Hooks in settings | `.kiro/hooks/*.json` |
| `.mcp.json` | `.kiro/settings/mcp.json` |

Kiro does not natively read a file named `CLAUDE.md`. Use `AGENTS.md`.

### 4.1 Instruction files

**Option A: `AGENTS.md`** (closest to CLAUDE.md). Place it in the workspace root or `~/.kiro/steering/`. Kiro also discovers AGENTS.md files in subdirectories (e.g. `services/api/`), each loaded as steering context. They are always included and have no inclusion modes.

```markdown
# AGENTS.md
## Stack
- TypeScript, Node 20, pnpm. Tests: Vitest.
## Commands
- Build: `pnpm build`   Test: `pnpm test`   Lint: `pnpm lint`
## Conventions
- Follow the error-handling pattern in src/lib/errors.ts
- Never edit generated files in /dist
```

**Option B: steering files** in `.kiro/steering/`, one topic per file (`testing.md`, `api-conventions.md`), optionally with front matter:

```markdown
---
inclusion: fileMatch
fileMatchPattern: "**/*.test.ts"
---
# Testing rules
- Use Vitest; mock only at module boundaries
```

**Caveat:** the CLI 3.0 notes list inclusion modes (always, fileMatch, manual, auto), but the main steering page says the CLI loads every file in `.kiro/steering/` regardless, and there are user reports of `manual` files loading anyway. Check `/context show` before relying on conditional loading.

**Shortcut:** run `/guide` and ask, "Analyze this repo and create an AGENTS.md with build/test commands and conventions." Then review and trim it.

**Reusing an existing CLAUDE.md:** add `file://CLAUDE.md` to an agent's `resources`. If your team also uses Claude Code, keep `AGENTS.md` as the source of truth and import it from `CLAUDE.md` with `@AGENTS.md`.

### 4.2 Skills

Skills follow the open Agent Skills standard, so skills from other compatible tools (including Claude Code) can be reused. Locations: `.kiro/skills/` (workspace, wins on name clash) and `~/.kiro/skills/` (global). The default agent loads both automatically.

```
.kiro/skills/pr-review/
├── SKILL.md
└── references/checklist.md
```

```markdown
---
name: pr-review
description: "Review pull requests for security issues, test coverage, and code quality. Use when reviewing PRs or preparing code for review."
---
## Checklist
1. Check for injection risks and exposed secrets
2. Confirm new code has tests
Details: see references/checklist.md
```

**Why skills save tokens:** only each skill's name and description load at startup; the full instructions load when your request matches; reference files load only as needed. Move long, situational material out of `AGENTS.md` into skills.

**Tips**
- Write a specific description using the words you'd naturally use in a request; this decides when the skill activates.
- Keep `SKILL.md` focused; put detail in `references/`.
- Quote descriptions containing a colon (a reported bug can silently drop them from discovery).
- Verify with `/context show`; some bug reports say skills loaded fully at startup instead of progressively.

### 4.3 Custom agents

An agent bundles its own prompt, tools, model, MCP servers, and resources. Create with `/guide Create an agent for code review`, `/agent create`, or `kiro-cli agent create <name>`. Switch with `/agent swap <name>`; set a default with `/agent set-default <name>`.

Configs can be JSON or Markdown with identical fields. Markdown agents are a 3.x feature; on 2.x use JSON.

`.kiro/agents/reviewer.md` (front matter is config, body is the system prompt):

```markdown
---
name: reviewer
description: Read-only code reviewer
model: claude-sonnet-5
tools: [read, shell]
allowedTools: [read]
includeMcpJson: false
resources:
  - file://AGENTS.md
  - skill://.kiro/skills/pr-review/SKILL.md
---
You are a meticulous code reviewer. Focus on correctness, security, and
maintainability. Do not modify files. Report findings by severity.
```

Confirm model IDs with `kiro-cli chat --list-models` and tool names with `/tools`.

**Permissions (3.x)** replace the deprecated `toolsSettings`:

```json
"permissions": { "rules": [
  { "capability": "shell", "match": ["npm *", "node *"], "effect": "allow" }
] }
```

**Steering inheritance (docs conflict):** the steering page says custom agents don't include steering automatically, while the 3.x configuration reference says they inherit default resources (steering, skills, AGENTS.md); the `chat.disableInheritingDefaultResources` setting turns that off. Run `/context show` under your agent. If steering is missing, add `"file://.kiro/steering/**/*.md"` to `resources`.

**Token savings via agents:** give each agent only the tools it needs, set `includeMcpJson: false` where MCP isn't needed, and use cheaper models for exploration.

### 4.4 Recommended team repo layout

```
repo/
├── AGENTS.md                  # short: stack, commands, hard rules
└── .kiro/
    ├── steering/              # optional per-domain rules
    ├── skills/                # long/situational playbooks (on demand)
    ├── agents/                # reviewer, test-writer, ...
    └── settings/mcp.json      # shared MCP servers
```

Commit it so everyone gets the same behavior. Keep `AGENTS.md` under a page: it loads every session, so you pay for it every time.

---

## 5. Saving tokens and credits

Kiro bills in credits. A simple prompt can cost under one credit; complex work (e.g. executing a spec task) usually costs more, and models consume credits at different rates. Run `/usage` to see what applies to you (enterprise accounts may be managed differently).

1. **Pick the model deliberately.** Start with Auto. Use Opus when stuck on hard problems (about 2.2x), Sonnet for strong agentic work at lower cost (about 1.3x), Haiku for quick fixes, and open-weight models (0.05x to 0.5x) for long or high-volume sessions. Check `--list-models` for current multipliers.
2. **Lower reasoning effort for easy work.** `/effort low` for renames, boilerplate, and small edits; raise it for design and tricky debugging. Higher effort spends more tokens even at the same multiplier. The setting persists across sessions, so reset it when the task changes.
3. **Keep context lean.** Run `/context` regularly. Start `/chat new` when switching tasks. Reference files with `@` instead of making Kiro search the repo. Keep `AGENTS.md` and steering short.
4. **Use `/compact` proactively.** Compaction runs automatically near the limit and keeps goals, decisions, modified files, and constraints while compressing tool output and exploration. It is one-way, so `/chat save` first if you need the full history.
5. **Trim MCP overhead.** Each connected server adds tool definitions to context. Disconnect unused servers. With 5+ servers or context overflow errors, enable Tool Search: `kiro-cli settings toolSearch.enabled true`.
6. **Do mechanical work yourself.** Use `!` for builds, tests, and git, and `kiro-cli translate` for shell one-liners.
7. **Use `/tangent`** for side questions so detours don't pollute the main conversation.

---

## 6. Caveats and security

- 3.x breaks compatibility in places: session format, hooks (now standalone `.kiro/hooks/*.json`), tool IDs, and the trust model. Back up `~/.kiro/sessions/` before switching, and use `/upgrade-agent` to migrate agent configs.
- Agents with write tools can modify anything under `~/.kiro` (skills, steering, MCP config), and skills run with the agent's permissions. Review third-party skills before installing.
- Enterprise admins may restrict models, MCP servers, and settings.

## 7. References

- Setup: kiro.dev/docs/cli/setup
- CLI commands: kiro.dev/docs/reference/cli-commands
- Slash commands: kiro.dev/docs/reference/slash-commands
- Steering, skills, custom agents: kiro.dev/docs/steering, /docs/skills, /docs/custom-agents
- Models and effort: kiro.dev/docs/models
- What's new in 3.0: kiro.dev/docs/cli/v3
