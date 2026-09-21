# Structural Code Graphs for AI Coding Agents
**CodeGraph, KiroGraph, and Graphify**

---

## Table of Contents

1. [Key Concepts and Definitions](#1-key-concepts-and-definitions)
2. [The Problem](#2-the-problem)
3. [How the Graph is Built](#3-how-the-graph-is-built)
4. [Tool Comparison: CodeGraph vs KiroGraph vs Graphify](#4-tool-comparison-codegraph-vs-kirograph-vs-graphify)
5. [Which Tool Should You Learn?](#5-which-tool-should-you-learn)
6. [When to Use Which](#6-when-to-use-which)
7. [Setup: CodeGraph in Claude Code](#7-setup-codegraph-in-claude-code)
8. [Setup: KiroGraph in Kiro CLI](#8-setup-kirograph-in-kiro-cli)
9. [Common Workflows](#9-common-workflows)
10. [Evidence and Benchmarks](#10-evidence-and-benchmarks)
11. [CLI and MCP Cheat Sheets](#11-cli-and-mcp-cheat-sheets)
12. [Pitfalls and Tips](#12-pitfalls-and-tips)
13. [Learning Path](#13-learning-path)
14. [Appendix: Graphify Quick Start](#14-appendix-graphify-quick-start)

---

## 1. Key Concepts and Definitions

Before comparing tools, here are the core terms used throughout this guide.

| Term | Definition |
| :--- | :--- |
| **AST (Abstract Syntax Tree)** | A tree representation of source code that captures its structural syntax, such as functions, classes, and expressions. |
| **Tree-sitter** | A parser generator that builds ASTs from source code. It enables fast, incremental structural analysis across many languages. |
| **Knowledge Graph** | A graph of nodes (functions, classes, files) and edges (calls, imports, extends) that represents code structure and relationships. |
| **MCP (Model Context Protocol)** | A standard protocol that allows AI agents to call external tools—such as a knowledge graph server—directly from their environment. |
| **Blast Radius** | The set of all code that could be affected by a change to a given function, class, or module. |
| **Incremental Indexing** | Updating only the parts of the graph that changed, rather than rebuilding the entire index from scratch. |
| **FTS5** | SQLite’s full-text search extension, used for fast text search over indexed symbols. |
| **Vector Engine** | A system for semantic search using embeddings, allowing queries based on meaning rather than exact text. |

---

## 2. The Problem

Imagine you are new to a huge library with millions of books. Someone asks you, “Where is the book that explains how to fix a car engine?” Without a catalog, you would have to walk through every aisle, pull out books one by one, and skim them until you find the right one. That could take hours.

That is exactly what traditional AI coding agents do when they explore a codebase. They use simple text searches (like `grep`) and read files one at a time. For a small project, this works. For a large project with thousands of files, it becomes painfully slow and expensive.

### Two Big Problems

**1. Wasting Money and Time (Token & Tool Call Waste)**  
Every time the AI reads a file or runs a search, it uses “tokens.” Tokens are like words—the AI pays for each one it reads and writes. If the AI has to read 50 files to find one function, that’s 50 separate actions (called “tool calls”), and thousands of tokens spent. The bigger the codebase, the more it costs, and the more likely you’ll hit rate limits (like a speed limit for AI requests).

**2. Missing the Big Picture (Lack of Structural Context)**  
Text searches only look for exact words. They don’t understand how pieces of code are connected. For example, if function `A` calls function `B`, and `B` calls `C`, a simple search for `C` won’t tell you that `A` eventually leads to `C`. It also can’t track things like: which classes inherit from which, which interfaces are implemented by which classes, or how a web request flows through different layers. This means the AI might miss important connections and give you incomplete answers.

### A Simple Example

**Without a Graph (The Old Way):**  
You ask the AI: “How does a user request reach the database?”  
The AI has to:
1. Search for a function name (e.g., `handleRequest`).
2. Read the file where it’s defined.
3. Manually figure out what that function calls next.
4. Search for that next function.
5. Read that file.
6. Repeat until it finds the database call.

This could take dozens of steps and thousands of tokens.

**With a Knowledge Graph (The New Way):**  
The AI makes a single tool call to a special graph database. The graph already knows all the connections. In milliseconds, it returns the entire path: `handleRequest → validateUser → queryDatabase`. It also gives you the exact lines of code and tells you what else might break if you change something.

### Why This Matters

A knowledge graph turns a slow, expensive, error-prone exploration into a fast, cheap, and accurate lookup. It’s like having a magical library catalog that not only tells you where every book is, but also shows you how all the books reference each other.

---

## 3. How the Graph is Built

Building a knowledge graph for code is like creating a detailed map of a city. You need to know where every building is, what roads connect them, and how people travel between them. Here’s how it works, step by step.

### Step 1: Parsing – Reading the Code Like a Human

First, the tool reads your source code. But instead of just treating it as plain text, it uses a special program called **Tree-sitter**. Think of Tree-sitter as a robot that understands programming languages. It reads a file and breaks it down into a tree structure called an **Abstract Syntax Tree (AST)**.

An AST is like a family tree for code. For example, a function called `calculateTax` might have child nodes for its parameters, its local variables, and the statements inside it. This tree captures the *structure* of the code, not just the words.

- **KiroGraph** recognizes about 24–26 different types of nodes (like functions, classes, etc.).
- **CodeGraph** supports 66 programming languages and can parse them in parallel (many at once).
- **Graphify** also uses Tree-sitter but can additionally read documents, PDFs, and images.

### Step 2: Nodes and Edges – The Building Blocks

Once the code is parsed, the tool extracts two things:

- **Nodes:** These are the “things” in your code. Examples: a function, a class, an interface, a variable, a route, or even a security vulnerability.
- **Edges:** These are the relationships between nodes. Examples: “function A calls function B,” “class X extends class Y,” “file M imports file N,” “interface I is implemented by class C.”

Think of nodes as people and edges as friendships or family relationships. The graph captures who knows whom and how they interact.

### Step 3: Storage – Keeping the Map Safe

The graph is saved locally on your computer, usually in a small database file.

- **CodeGraph** uses a single SQLite file (`.codegraph/db.sqlite`) with special features for fast text search.
- **KiroGraph** also uses SQLite (`.kirograph/kirograph.db`) but adds support for 9 different vector engines, which allow searching by meaning rather than exact words.
- **Graphify** exports the graph as a JSON file, an HTML visualization, and a markdown report.

Because everything is stored locally, your code never leaves your machine. You can add these files to `.gitignore` so they don’t get committed to version control.

### Step 4: MCP Exposure – Talking to Your AI Agent

Finally, the graph is exposed to your AI coding agent through something called **MCP (Model Context Protocol)**. MCP is like a walkie-talkie between your AI and the graph. When the AI needs to know something, it sends a query over MCP, and the graph responds instantly.

For example, the AI might ask: “Show me all functions that call `loginUser`.” The graph looks it up and returns the answer in milliseconds. No more grepping through files!

This is why a knowledge graph is so powerful: it turns a slow, manual search into a fast, automated conversation.

---

## 4. Tool Comparison: CodeGraph vs KiroGraph vs Graphify

| Feature / Dimension | CodeGraph | KiroGraph | Graphify |
| :--- | :--- | :--- | :--- |
| **Origin** | Original project by Colby McHenry for Claude Code. | Ported and expanded natively from CodeGraph by Davide Desio for Kiro. | Independent multimodal knowledge graph tool. |
| **Primary Target** | Claude Code; also Cursor, Codex, opencode, Hermes, Antigravity CLI. | Kiro IDE & CLI; experimental support for 34+ other tools. | Claude Code, Cursor, and other MCP-compatible agents. |
| **Language & Runtime** | Rust. Single self-contained binary (~5 MB), zero runtime dependencies. | TypeScript. Runs via Node runtime. | Python 3.10+. Requires Python runtime and Tree-sitter grammars. |
| **Storage & Indices** | SQLite (`.codegraph/db.sqlite`) with FTS5. | SQLite (`.kirograph/kirograph.db`) + 9 pluggable vector engines (PGlite, Qdrant, Typesense, etc.). | NetworkX graph; exports `graph.json`, `graph.html`, `GRAPH_REPORT.md`. |
| **Semantic & Extra Layers** | Text search via FTS5. Lacks built-in vector engines, wiki, or memory layers. | Vector engines (`nomic-embed-text-v1.5`), architecture metrics (Ca/Ce/instability), LLM Wiki, Watchmen memory, security/vulnerability scanning, tabular data, PDF parsing, SIMD/quantization. | Multimodal ingestion (docs, PDFs, images, video, audio); edge provenance (`EXTRACTED`, `INFERRED`, `AMBIGUOUS`). |
| **Sync Mechanism** | Built-in file watcher with XXH3 content hashing/polling for incremental re-indexing. | Kiro Hooks triggered on `agentStop` to sync index when files change; zero overhead during active editing. | Git hooks (`post-commit`, `post-checkout`) or manual `--update`. Graph can become stale between commits. |
| **Tool Design Strategy** | Exposes 1 primary tool (`codegraph_explore`) to reduce mis-picks and save context (or 9 tools in specific builds). | Exposes up to 126 claimed MCP tools; earlier configs list 18 core tools plus sub-domain tools (wiki, tabular data, docs). | 24+ MCP tools. |
| **Setup Effort** | Single binary installation via npm: `npm install -g @colbymchenry/codegraph`. | Interactive installer: `kirograph install` configures MCP, hooks, and steering files. | `pip install graphifyy` (note double “y”), then `graphify claude install`. |
| **Best For** | Fast, local code navigation; Claude Code and multi-agent CLI setups. | Kiro IDE/CLI users needing advanced semantic, security, and wiki layers. | Multimodal projects where code, docs, PDFs, and diagrams must be understood together. |

> **Marketing Claims vs. Evidence**
> - **CodeGraph languages:** The paper claims 66 languages. The Rust README notes Kotlin support is blocked upstream on a Tree-sitter grammar upgrade; Dart, Pascal, and Luau are listed under “Coming back.”
> - **KiroGraph tool count:** Documentation claims “126 MCP Tools,” but earlier auto-approve configs list 18 core tools. The high count includes sub-domain tools beyond pure AST parsing.
> - **Graphify token reduction:** Vendor claims of “71.5x” token reduction are not independently verified. Treat as marketing figures.

---

## 5. Which Tool Should You Learn?

**Start with CodeGraph if you are learning.**

| Goal | Recommendation |
| :--- | :--- |
| Learn the core concept of structural code graphs with minimal setup. | **CodeGraph** |
| Work primarily in Claude Code or multi-agent CLI setups. | **CodeGraph** |
| Need a lightweight, fast, self-contained binary with no Node.js or Python runtime. | **CodeGraph** |
| Use Kiro IDE/CLI and need advanced semantic, security, or wiki layers. | **KiroGraph** |
| Analyze multimodal repositories (code + PDFs + diagrams + docs). | **Graphify** |
| Need edge provenance and an audit trail for inferred relationships. | **Graphify** |

**Verdict:** Learn **CodeGraph first**. It teaches the core ideas—AST parsing, graph queries, blast radius—without the overhead of Python environments or multimodal complexity. Once comfortable, move to **KiroGraph** for Kiro-specific workflows or **Graphify** for multimodal projects.

---

## 6. When to Use Which

### Use CodeGraph if:
- You prioritize a lightweight, fast, self-contained binary with zero Node.js or external runtime dependencies.
- You work primarily in Claude Code or multi-agent CLI setups (Cursor, Codex, Hermes).
- You prefer a single, streamlined tool call (`codegraph_explore`) that returns verbatim source, call flows, and blast radius in one payload.
- You need background file-watching that updates automatically during editing sessions.

### Use KiroGraph if:
- You use the Kiro AI IDE or Kiro CLI as your primary environment.
- You require advanced, non-code indexing layers: LLM Wiki, session synthesis, security/CVE reachability analysis, CycloneDX SBOM exports, tabular data, or PDF processing.
- You require pluggable local semantic search / vector engines (e.g., PGlite for large repos, SQLite cosine, Qdrant).
- You prefer zero-overhead editing, where re-indexing occurs only when the agent stops (`agentStop` hook).

### Use Graphify if:
- Your repository includes non-code artifacts: documentation, PDFs, images, video, or audio.
- You need to understand design rationale, not just code structure.
- You want edge provenance (`EXTRACTED`, `INFERRED`, `AMBIGUOUS`) for auditing how relationships were derived.
- You are willing to manage a Python environment and git-hook-based syncing.

### Use Neither if:
- You work in unsupported language edge-cases (e.g., C projects heavily dependent on preprocessor macros where ASTs cannot reflect structure).
- You are debugging dynamic dispatch runtime behaviors, reflection, or unindexed runtime state that static AST parsing cannot capture.
- Your task requires exact line-level source code inspection across unindexed files where raw file reading is necessary.

---

## 7. Setup: CodeGraph in Claude Code

### Prerequisites
- Node.js and npm installed.
- Claude Code installed and configured.
- A local project directory.

### Step-by-Step Setup Checklist

1. **Install Global CLI:**
   ```bash
   npm install -g @colbymchenry/codegraph
   ```

2. **Wire Agent MCP:**
   ```bash
   codegraph install
   ```
   Or manually add `codegraph serve --mcp` to `~/.claude.json`.

3. **Configure Permissions:**  
   Add to `~/.claude/settings.json`:
   ```json
   {
     "permissions": {
       "allow": ["mcp__codegraph__*"]
     }
   }
   ```

4. **Navigate to Project:**
   ```bash
   cd my-react-project
   ```

5. **Initialize Graph:**
   ```bash
   codegraph init
   ```
   This generates `.codegraph/db.sqlite`.

6. **Confirm Integration:**  
   Open Claude Code (`claude`), ask “How does index.tsx render components?”, and verify in logs/output that `codegraph_explore` is executed.

### Exposed MCP Tools
- **`codegraph_explore`:** Unified tool returning verbatim source, call paths, dynamic dispatch, and blast radius.
- **Unlisted Core Tools** (re-enabled via `CODEGRAPH_MCP_TOOLS`):  
  `codegraph_node`, `codegraph_search`, `codegraph_callers`, `codegraph_callees`, `codegraph_impact`, `codegraph_files`, `codegraph_status`.

### Steering & Guidance
The MCP server delivers instructions automatically during the initialize response, telling the agent to answer structural queries directly using CodeGraph without re-verifying via `grep`.

---

## 8. Setup: KiroGraph in Kiro CLI

### Prerequisites
- Node.js installed.
- Kiro CLI installed.
- A fresh React project (optional, for testing).

### Step-by-Step Setup Checklist

1. **Create React Project:**
   ```bash
   npx create-react-app my-app && cd my-app
   ```

2. **Install KiroGraph Globally:**
   ```bash
   npm install -g kirograph
   ```

3. **Wire Kiro Workspace:**
   ```bash
   kirograph install
   # or
   kirograph install --target kiro
   ```

4. **Verify Generated Files:**  
   Check that `.kiro/settings/mcp.json` contains `kirograph serve --mcp` and auto-approve tool arrays.

5. **Verify Hooks:**  
   Ensure `.kiro/hooks` contains the `agentStop` hook for automated background syncing.

6. **Launch Kiro CLI:**
   ```bash
   kiro
   ```
   Switch to the `kirograph` agent.

7. **Confirm Execution:**  
   Ask “Explain App.js callers and dependencies.” Verify Kiro calls `kirograph_context` or `kirograph_callers` natively.

---

## 9. Common Workflows

| Workflow | Goal | CodeGraph | KiroGraph |
| :--- | :--- | :--- | :--- |
| **Onboarding to a New Repo** | Understand overall structure, entry points, and high-level routing. | `codegraph_explore(query="survey system architecture and main entry points")` | `kirograph_architecture()`, `kirograph_package()` |
| **Impact Analysis Before Refactoring** | Find all downstream code affected before modifying a function signature. | `codegraph_explore(query="blast radius for PaymentService.process")` or `git diff --name-only HEAD \| codegraph affected --stdin` | `kirograph_impact(symbol="PaymentService.process")` |
| **Finding Dead Code** | Locate unused functions, classes, or unreferenced exports. | `codegraph_status()` or query unreferenced nodes via `codegraph_explore` | `kirograph_dead_code()` |
| **Debugging a Call Chain** | Trace how data flows from an API route down to a database call. | `codegraph_explore(query="how does POST /checkout reach database write")` | `kirograph_path(from="handleCheckout", to="dbInsert")`, `kirograph_callees(symbol="handleCheckout")` |
| **Architecture & Coupling Review** | Identify unstable components, circular dependencies, and module coupling. | Use `codegraph_explore` for structural overview | `kirograph_circular_deps()`, `kirograph_coupling()`, `kirograph_hotspots()` |

> **Graphify alternative:** For multimodal or provenance-aware workflows, use `graphify query "..."`, `graphify path "A" "B"`, or `graphify explain "Node"`.

---

## 10. Evidence and Benchmarks

You might wonder: “Does using a knowledge graph actually make a difference?” Researchers compared two types of AI agents: one that explores files the old-fashioned way (File Exploration Agent) and one that uses a knowledge graph (Graph-Based MCP Agent). They tested them on 31 real-world code repositories. Here’s what they found, explained in simple terms.

### The Numbers

| What They Measured | Old Way (File Exploration) | New Way (Graph-Based) | What It Means |
| :--- | :--- | :--- | :--- |
| **Token Usage** | Baseline (100%) | 90% less (10x savings) | The graph agent used only 10% of the tokens. That’s like paying for 1 page instead of 10. |
| **Tool Calls** | Baseline | 2.1x fewer | The graph agent needed half as many separate actions to get the job done. |
| **Overall Quality** | 92% | 83% | The graph agent’s answers were slightly less accurate in some cases. |
| **Query Latency** | Seconds to minutes | Less than 1 millisecond | The graph agent responded almost instantly. |

> **Important:** These numbers come from the tool makers’ own research. Your results may vary depending on your codebase. Always test on your own projects.

### Why Is the Quality Slightly Lower?

The graph is fantastic for structural questions like “Who calls this function?” or “What breaks if I change this?” But it has three limitations:

1. **Line-Level Details:** If you need to see the exact code inside a function that isn’t named (like a complex expression), the graph might not store it. You’d still need to read the raw file.
2. **Preprocessor Macros (C/C++):** In languages like C, macros are expanded before the code is compiled. The graph is built from the original source, so it can’t see what the macro turns into.
3. **Dynamic Dispatch & Reflection:** Some code decides at runtime which function to call (e.g., in JavaScript or Python). The graph can’t predict these dynamic choices because it only knows static structure.

So the graph is like a brilliant map, but sometimes you still need to walk the streets yourself.

### The Bottom Line

For most everyday tasks—understanding code, finding call chains, checking impact—the graph saves enormous time and money. For highly detailed, line-by-line debugging, you may need to combine it with traditional file reading.

---

## 11. CLI and MCP Cheat Sheets

### CodeGraph CLI Reference

| Command | Description | Example |
| :--- | :--- | :--- |
| `codegraph` | Runs the interactive installer wizard. | `codegraph` |
| `codegraph install` | Wires CodeGraph MCP server into detected local agents. | `codegraph install` |
| `codegraph uninstall` | Removes CodeGraph from agent configs and CLI. | `codegraph uninstall --keep-cli` |
| `codegraph init [path]` | Initializes local `.codegraph/` directory and builds index. | `codegraph init .` |
| `codegraph uninit [path]` | Removes `.codegraph/` index from project. | `codegraph uninit . --force` |
| `codegraph index [path]` | Forces full re-indexing of the project. | `codegraph index . --force` |
| `codegraph sync [path]` | Runs incremental update on modified files. | `codegraph sync` |
| `codegraph status [path]` | Displays graph statistics and node/edge counts. | `codegraph status` |
| `codegraph unlock [path]` | Clears stale lock file blocking indexing. | `codegraph unlock` |
| `codegraph query <search>` | Searches indexed symbols by pattern. | `codegraph query UserService --kind class` |
| `codegraph explore <query>` | Traces call paths and returns source + blast radius. | `codegraph explore "auth flow"` |
| `codegraph node <symbol\|file>` | Fetches line-numbered source and callers for symbol/file. | `codegraph node AuthController` |
| `codegraph files [path]` | Displays indexed file tree structure. | `codegraph files --max-depth 2` |
| `codegraph callers <symbol>` | Finds all functions calling specified symbol. | `codegraph callers loginUser` |
| `codegraph callees <symbol>` | Finds all functions called by specified symbol. | `codegraph callees loginUser` |
| `codegraph impact <symbol>` | Analyzes impact/blast radius of changing a symbol. | `codegraph impact PaymentGateway` |
| `codegraph affected [files...]` | Identifies test files affected by changed files. | `git diff --name-only \| codegraph affected --stdin` |
| `codegraph daemon` | Manages background file-watching daemons. | `codegraph daemon` |
| `codegraph upgrade` | Updates CodeGraph CLI to latest version. | `codegraph upgrade --check` |

### CodeGraph MCP Tools Reference

| Tool | Description | Example |
| :--- | :--- | :--- |
| `codegraph_explore` | Single primary tool returning verbatim source, call flows, dynamic dispatch, and blast radius. | `codegraph_explore(query="how does App render Header")` |
| `index_repository` | Builds or updates complete repository graph index. | `index_repository(path=".")` |
| `index_status` | Polls background indexing progress. | `index_status()` |
| `list_projects` | Lists all indexed local repositories. | `list_projects()` |
| `delete_project` | Removes graph index for target repository. | `delete_project(path=".")` |
| `search_graph` | Executes symbol search across index. | `search_graph(query="User")` |
| `trace_call_path` | Traverses call chain between source and target. | `trace_call_path(start="A", end="B")` |
| `query_graph` | Performs Cypher-like structural query over graph. | `query_graph(query="MATCH (n) RETURN n")` |
| `ingest_traces` | Imports runtime trace logs into graph. | `ingest_traces(traceData="...")` |
| `detect_changes` | Analyzes git diff impact on graph nodes. | `detect_changes(commit="HEAD")` |
| `get_graph_schema` | Inspects available node and edge schema types. | `get_graph_schema()` |
| `get_architecture` | Returns high-level architecture overview. | `get_architecture()` |

### KiroGraph CLI Reference

| Command | Description | Example |
| :--- | :--- | :--- |
| `kirograph install` | Wires MCP server, hooks, steering files, and agent. | `kirograph install` |
| `kirograph install --target claude` | Installs Claude Code MCP `.mcp.json` and memory instructions. | `kirograph install --target claude` |
| `kirograph install --target codex` | Prints Codex config and updates `AGENTS.md`. | `kirograph install --target codex` |
| `kirograph uninit` | Removes Kiro integration files and local `.kirograph/` data. | `kirograph uninit --force` |
| `kirograph serve --mcp` | Starts MCP stdio server instance. | `kirograph serve --mcp` |

### KiroGraph MCP Tools Reference

| Tool | Description | Example |
| :--- | :--- | :--- |
| `kirograph_context` | Generates comprehensive task context payload. | `kirograph_context(query="fix login")` |
| `kirograph_search` | Searches graph nodes by identifier/keyword. | `kirograph_search(query="Auth")` |
| `kirograph_callers` | Lists all incoming call edges for symbol. | `kirograph_callers(symbol="validateSession")` |
| `kirograph_callees` | Lists all outgoing call edges from symbol. | `kirograph_callees(symbol="validateSession")` |
| `kirograph_impact` | Calculates downstream dependency blast radius. | `kirograph_impact(symbol="DbClient")` |
| `kirograph_node` | Inspects properties and edges of single node. | `kirograph_node(symbol="App")` |
| `kirograph_status` | Checks index freshness and total node/edge counts. | `kirograph_status()` |
| `kirograph_files` | Returns indexed file structure tree. | `kirograph_files()` |
| `kirograph_dead_code` | Scans graph for unreferenced nodes. | `kirograph_dead_code()` |
| `kirograph_circular_deps` | Identifies cyclic dependency loops. | `kirograph_circular_deps()` |
| `kirograph_path` | Traces path between two symbols. | `kirograph_path(from="A", to="B")` |
| `kirograph_type_hierarchy` | Returns inheritance/implementation trees. | `kirograph_type_hierarchy(symbol="BaseUser")` |
| `kirograph_architecture` | Summarizes component coupling and stability metrics. | `kirograph_architecture()` |
| `kirograph_coupling` | Measures Efferent/Afferent coupling between modules. | `kirograph_coupling()` |
| `kirograph_package` | Analyzes package-level dependency boundaries. | `kirograph_package()` |
| `kirograph_hotspots` | Locates most-connected load-bearing symbols. | `kirograph_hotspots()` |
| `kirograph_surprising` | Identifies non-obvious cross-file dependencies. | `kirograph_surprising()` |
| `kirograph_diff` | Diffs current graph against saved snapshot. | `kirograph_diff(snapshot="v1")` |

---

## 12. Pitfalls and Tips

Even the best tools have quirks. Here are common problems you might run into and how to avoid them.

### 1. Stale Index – Your Map Is Out of Date

**What it is:** The graph is built from a snapshot of your code. If you change your code but don’t update the graph, it becomes stale—like using a map from 10 years ago to navigate a city with new roads.

**How to avoid it:**
- **CodeGraph:** It watches your files in the background and updates automatically. But always check the “staleness banner” in tool responses after you make manual edits.
- **KiroGraph:** It updates when your AI agent stops working (the `agentStop` hook). If you edit code outside Kiro, you may need to manually re-index.
- **Graphify:** It uses Git hooks or a manual `--update` command. The graph can become stale between commits, so rebuild before a big refactor.

**Tip:** If something seems wrong, run the status command (e.g., `codegraph status` or `kirograph_status`) to check if the graph is fresh.

### 2. Too Many Tools – Remote Control Overload

**What it is:** Some graph tools offer dozens of MCP tools. If your AI agent sees 50 different tools, it might get confused about which one to use. It also wastes context window space (the AI’s short-term memory).

**How to avoid it:**
- **CodeGraph** solves this by exposing just one main tool: `codegraph_explore`. It handles almost everything.
- **KiroGraph** exposes many tools, but you can configure auto-approve lists to limit which ones are visible.
- **Graphify** has 24+ tools, so read the documentation and enable only what you need.

**Tip:** Start with the fewest tools possible. Add more only when you find a specific need.

### 3. Security & Privacy – Keep Your Code Safe

**What it is:** You might worry that your code is being sent to the cloud. Good news: all three tools store their databases locally.

- **CodeGraph** and **KiroGraph** keep everything on your machine (`.codegraph/` or `.kirograph/` folders).
- **Graphify** writes to `graphify-out/` locally.
- CodeGraph also runs automated security checks on its own binary to prevent tampering.

**How to stay safe:**
- Add the graph database folders to your `.gitignore` file. This prevents accidentally committing your local database to a public repository.
- If you’re extra cautious, you can encrypt your disk. But for most users, local storage is already very secure.

**Tip:** Never share your `.codegraph/`, `.kirograph/`, or `graphify-out/` folders with others unless you intend to. They contain a full structural map of your code.

### 4. Unsupported Languages – When the Map Doesn’t Cover Your City

**What it is:** No graph tool supports every programming language perfectly. For example, CodeGraph has trouble with Kotlin (waiting on a Tree-sitter update) and some C/C++ macros. Graphify may not parse your favorite framework’s routing.

**How to avoid it:**
- Check the tool’s documentation for supported languages.
- If your language is unsupported, fall back to traditional file reading.
- Consider using a different tool that better fits your tech stack.

**Tip:** If you work with a niche language, test the tool on a small project first before relying on it for a large codebase.

### 5. Dynamic Code – When the Map Can’t Predict Traffic

**What it is:** Static analysis (like AST parsing) can’t see runtime behavior. If your code uses reflection, dynamic imports, or runtime dependency injection, the graph might miss those connections.

**How to avoid it:**
- Use the graph for what it’s good at: structural relationships that are visible in the source code.
- For dynamic behavior, fall back to reading files, adding logging, or using runtime tracing tools.
- Some tools (like CodeGraph) allow ingesting runtime traces to augment the graph. Check if your tool supports this.

**Tip:** Don’t expect the graph to be a crystal ball. It’s a map of the static code, not a simulation of the running program.

---

## 13. Learning Path

### Step 1: Beginner — Single Symbol Lookup
**Goal:** Query a symbol without reading raw files.  
**Hands-on Exercise:** Install CodeGraph or KiroGraph on a sample repository. Ask your agent: “Where is `processOrder` defined and what calls it?” Verify the agent executes `codegraph_explore` or `kirograph_callers` without running `grep`.

### Step 2: Intermediate — Flow Tracing & Impact Analysis
**Goal:** Perform impact analysis before refactoring a function.  
**Hands-on Exercise:** Select a core utility module. Ask the agent: “What is the blast radius if I change the parameters of `calculateTax`?” Inspect the returned call tree and blast-radius summary.

### Step 3: Advanced — Architectural Analysis & CI Automation
**Goal:** Detect circular dependencies, architectural hotspots, and integrate graph checks into CI.  
**Hands-on Exercise:**
- Run architectural health commands: execute `kirograph_circular_deps` or `kirograph_coupling` to identify structural instability.
- Add a pre-commit or CI script to test modified files against affected paths:

```bash
#!/usr/bin/env bash
AFFECTED=$(git diff --name-only HEAD | codegraph affected --stdin --quiet)
if [ -n "$AFFECTED" ]; then
  npx vitest run $AFFECTED
fi
```

---

## 14. Appendix: Graphify Quick Start

### Prerequisites
- Python 3.10+ and `pip` installed.
- An MCP-compatible agent (Claude Code, Cursor, etc.).

### Installation
```bash
pip install graphifyy
```
> **Note:** The PyPI package name is `graphifyy` with a double “y”.

### Build the Graph
```bash
graphify .
# or inside Claude Code:
/graphify
```
Outputs are written to `graphify-out/`:
- `graph.json`
- `graph.html`
- `GRAPH_REPORT.md`

### Integrate with Claude Code
```bash
graphify claude install
```
This writes a `CLAUDE.md` directive and a `PreToolUse` hook that tells Claude to consult the graph before searching raw files.

### Query the Graph
```bash
graphify query "How does authentication work?"
graphify path "handleCheckout" "dbInsert"
graphify explain "PaymentService"
```

### MCP Tools
- `query_graph(question="...")`
- `find_path(start="A", end="B")`
- `explain_node(node="...")`
- `get_neighbors(node="...")`
- Plus additional tools for community detection, export, and multimodal ingestion.

### Sync
```bash
graphify hook install   # git hooks for automatic rebuilds
graphify --update       # manual incremental update
```

### When to Choose Graphify
- Multimodal repositories (code + PDFs + diagrams + docs).
- Need edge provenance (`EXTRACTED`, `INFERRED`, `AMBIGUOUS`).
- Willing to manage a Python environment and git-hook-based syncing.

> **Caution:** Vendor claims of extreme token reduction are not independently verified. Test on your own repository before relying on them.

---

*This guide is now complete, with beginner-friendly explanations for chapters 2, 3, 10, and 12, a fully linked table of contents, and all non-essential noise removed. The core valuable information remains intact for both beginners and experienced users.*
