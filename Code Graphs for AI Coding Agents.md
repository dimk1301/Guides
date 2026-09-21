# Structural Code Graphs for AI Coding Agents
**CodeGraph, KiroGraph, and Graphify**

> **Audience:** Experienced software engineers new to AI coding agents, MCP, and code knowledge graphs.

---

## Table of Contents

1. [Key Concepts and Definitions](#1-key-concepts-and-definitions)
2. [The Problem](#2-the-problem)
3. [Parser, AST, and Graph: How They Relate](#3-parser-ast-and-graph-how-they-relate)
4. [How the Graph is Built](#4-how-the-graph-is-built)
5. [Tool Comparison: CodeGraph vs KiroGraph vs Graphify](#5-tool-comparison-codegraph-vs-kirograph-vs-graphify)
6. [Which Tool Should You Learn?](#6-which-tool-should-you-learn)
7. [When to Use Which](#7-when-to-use-which)
8. [Setup: CodeGraph in Claude Code](#8-setup-codegraph-in-claude-code)
9. [Setup: KiroGraph in Kiro CLI](#9-setup-kirograph-in-kiro-cli)
10. [Common Workflows](#10-common-workflows)
11. [Evidence and Benchmarks](#11-evidence-and-benchmarks)
12. [CLI and MCP Cheat Sheets](#12-cli-and-mcp-cheat-sheets)
13. [Pitfalls and Tips](#13-pitfalls-and-tips)
14. [Learning Path](#14-learning-path)
15. [Appendix: Graphify Quick Start](#15-appendix-graphify-quick-start)

---

## 1. Key Concepts and Definitions

| Term | Definition |
| :--- | :--- |
| **AST (Abstract Syntax Tree)** | A tree representation of source code that captures its structural syntax, such as functions, classes, and expressions. |
| **Tree-sitter** | A parser generator that builds ASTs from source code. Enables fast, incremental structural analysis across many languages. |
| **Knowledge Graph** | A graph of nodes (functions, classes, files) and edges (calls, imports, extends) representing code structure and relationships. |
| **MCP (Model Context Protocol)** | A standard protocol allowing AI agents to call external tools—such as a knowledge graph server—directly from their environment. |
| **Blast Radius** | The set of all code that could be affected by a change to a given function, class, or module. |
| **Incremental Indexing** | Updating only the parts of the graph that changed, rather than rebuilding the entire index. |
| **FTS5** | SQLite’s full-text search extension, used for fast text search over indexed symbols. |
| **Vector Engine** | A system for semantic search using embeddings, enabling queries based on meaning rather than exact text. |

---

## 2. The Problem

Traditional LLM coding agents explore codebases through iterative file reads, `grep`, and `glob` searches. Each read or search is a separate tool call, and every tool call consumes context tokens. On large or complex tasks—e.g. “fix the auth bug” or “add rate limiting”—this creates two structural problems.

**1. Token and tool-call waste**  
Cost scales with codebase size and task depth. To trace a single call chain, the agent may need dozens of `grep`/read cycles. That inflates token usage, increases latency, and can hit model rate limits.

**2. No structural context**  
Text search has no AST-level awareness. It cannot reliably resolve:

- transitive callers/callees
- interface-to-implementation mappings
- dynamic callbacks or framework routing
- inheritance/implementation hierarchies
- cross-file dependency chains

**Concrete example**  
An agent asked “how does a request reach the database?” must grep for a function name, read matching files, manually parse the call, grep again, and repeat. Mapping one call chain can consume thousands of tokens across many tool calls.

With a knowledge graph, the agent issues one MCP call—e.g. `codegraph_explore` or `kirograph_path`. The graph pre-computes or traces the path via SQL recursive CTEs, returning the full call flow, line-numbered source, and blast radius in sub-milliseconds.

---

## 3. Parser, AST, and Graph: How They Relate

Three layers, one pipeline:

```
Source code  →  [Parser]  →  AST  →  [Extractor + Resolver]  →  Graph
```

**Parser.** A program that reads source text and produces an AST. It is per-file and syntax-only. It answers: "Is this valid, and what is its grammatical structure?" It does not know what symbols refer to.

**AST.** A tree of syntax. One AST per file. Nodes are language constructs (`FunctionDeclaration`, `CallExpression`, `ImportSpecifier`). Containment is the only relationship. No cross-file links, no type resolution.

**Graph.** A semantic model built by walking ASTs and resolving references across files. Nodes are entities (functions, classes, methods), edges are relationships (`calls`, `imports`, `extends`, `implements`). A general graph, not a tree. Spans the whole project.

**The key distinction.** Given `u.greet()`:

- The **parser** produces a `CallExpression` node wrapping a `MemberExpression`.
- The **AST** records only the syntax. It does not know what `u` is or where `greet` is defined.
- The **graph** records a `calls` edge from the enclosing function to `User.greet`—*only if* the extractor can resolve `u`'s type across files.

That `calls` edge does not exist anywhere in the AST. It is created by cross-file resolution. This is the entire value the graph adds: **it captures the semantic relationships the AST cannot represent.** Everything the graph fails to capture (dynamic dispatch, macro expansion, line-level content) is a consequence of what the AST omits and the resolver cannot infer.

---

## 4. How the Graph is Built

**Parsing**  
Source files are parsed into ASTs via Tree-sitter. KiroGraph indexes 24–26 node kinds. CodeGraph supports 66 languages using parallel worker pools. Graphify uses Tree-sitter plus multimodal extraction for docs, PDFs, and media.

**Node and edge extraction**

- **Nodes:** functions, methods, classes, interfaces, types, enums, variables, constants, routes, components, dependencies, vulnerabilities.
- **Edges:** calls, imports, exports, extends, implements, contains, references, instantiates, overrides, decorates, `type_of`, returns.

Some edges are **EXTRACTED** (visible directly in the AST), some are **INFERRED** (from type resolution or heuristics), and some are **AMBIGUOUS** (could not be resolved). Graphify tags edges with these provenance values for auditing.

**Storage**  
The structural graph is stored locally:

- **CodeGraph:** SQLite (`.codegraph/db.sqlite`) with WAL and FTS5.
- **KiroGraph:** SQLite (`.kirograph/kirograph.db`) plus 9 pluggable semantic vector engines (PGlite, Qdrant, Typesense, etc.).
- **Graphify:** NetworkX graph exported as `graph.json`, `graph.html`, and `GRAPH_REPORT.md`.

**MCP exposure**  
The graph is exposed to LLM agents as an MCP server. Agents execute typed structural queries over MCP instead of scanning raw files. This is the integration layer that makes the graph usable by an AI agent without custom tooling.

---

## 5. Tool Comparison: CodeGraph vs KiroGraph vs Graphify

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

## 6. Which Tool Should You Learn?

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

## 7. When to Use Which

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

**1. You are debugging dynamic dispatch, reflection, or runtime-resolved behavior.**  
Static AST graphs cannot see method calls resolved by variable names (`obj[methodName]()`), reflection (`Class.forName(...)`), DI container bindings, event bus topics, or dynamic imports. The graph records the call site but not the target. For these cases, use runtime tracing, debugger breakpoints, framework-specific diagnostics (e.g., Spring Actuator, `dotnet-trace`), or ingest runtime traces into the graph if the tool supports it (e.g., CodeGraph's `ingest_traces`).

**2. Your task requires exact line-level inspection of unindexed files.**  
The graph indexes supported source files and stores named nodes with verbatim source. It does not index generated code (protobuf stubs, GraphQL codegen), unsupported languages (SQL, YAML, HCL, templates), vendored dependencies, or files excluded by config. If you need to read a specific line in a generated file, compare a SQL migration against an ORM model, or inspect a C macro expansion, you must read the raw file. The graph tells you *which file* to read; it does not replace reading it.

**General rule:** Use the graph for structural questions ("who calls X?", "what breaks if I change Y?"). Use raw file reading for content questions ("what does line 47 do?", "does this handle null?"). The graph is a map, not a copy.

---

## 8. Setup: CodeGraph in Claude Code

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

## 9. Setup: KiroGraph in Kiro CLI

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

## 10. Common Workflows

| Workflow | Goal | CodeGraph | KiroGraph |
| :--- | :--- | :--- | :--- |
| **Onboarding to a New Repo** | Understand overall structure, entry points, and high-level routing. | `codegraph_explore(query="survey system architecture and main entry points")` | `kirograph_architecture()`, `kirograph_package()` |
| **Impact Analysis Before Refactoring** | Find all downstream code affected before modifying a function signature. | `codegraph_explore(query="blast radius for PaymentService.process")` or `git diff --name-only HEAD \| codegraph affected --stdin` | `kirograph_impact(symbol="PaymentService.process")` |
| **Finding Dead Code** | Locate unused functions, classes, or unreferenced exports. | `codegraph_status()` or query unreferenced nodes via `codegraph_explore` | `kirograph_dead_code()` |
| **Debugging a Call Chain** | Trace how data flows from an API route down to a database call. | `codegraph_explore(query="how does POST /checkout reach database write")` | `kirograph_path(from="handleCheckout", to="dbInsert")`, `kirograph_callees(symbol="handleCheckout")` |
| **Architecture & Coupling Review** | Identify unstable components, circular dependencies, and module coupling. | Use `codegraph_explore` for structural overview | `kirograph_circular_deps()`, `kirograph_coupling()`, `kirograph_hotspots()` |

> **Graphify alternative:** For multimodal or provenance-aware workflows, use `graphify query "..."`, `graphify path "A" "B"`, or `graphify explain "Node"`.

---

## 11. Evidence and Benchmarks

Evaluation across 31 real-world repositories compares a traditional file-exploration agent against a graph-based MCP agent.

| Metric | File Exploration Agent | Graph-Based MCP Agent |
| :--- | :--- | :--- |
| Token Usage | Baseline | 90% reduction (10x savings) |
| Tool Calls | Baseline | 2.1x fewer |
| Overall Quality | 92% | 83% |
| Query Latency | Seconds to minutes | < 1 ms via SQL CTEs |

**Trade-offs and limitations**

- **Line-level queries:** Graph tools struggle when the task requires exact line-level statements not stored as named nodes.
- **Preprocessor/macro expansion:** Structural extraction fails on C/C++ macro expansions because AST parsing occurs before preprocessing.
- **Dynamic dispatch:** Static AST indexing cannot capture runtime reflection or dynamic resolution. Raw file reading remains necessary for those cases.

**Caveat:** These figures are vendor-reported and should be validated on your own codebase before relying on them.

---

## 12. CLI and MCP Cheat Sheets

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

## 13. Pitfalls and Tips

**Stale index**  
The graph reflects a snapshot. If code changes without re-indexing, queries return stale results.

- **CodeGraph:** background file watcher with XXH3 content hashing. Check the staleness banner in tool responses after manual edits.
- **KiroGraph:** relies on the `agentStop` hook. If you edit outside Kiro, manually refresh or re-index.
- **Graphify:** uses Git hooks or manual `--update`. Rebuild before critical refactors.

**Tool overload**  
Registering too many granular MCP tools can confuse the model and consume context window during initialization.

- **CodeGraph** exposes one primary tool (`codegraph_explore`) to avoid mis-picks.
- **KiroGraph** exposes many tools; use auto-approve lists to limit visibility.
- **Graphify** has 24+ tools; enable only what you need.

**Security and privacy**  
All three store data locally. CodeGraph and KiroGraph use `.codegraph/` or `.kirograph/`; Graphify writes to `graphify-out/`. Add these to `.gitignore`. CodeGraph also runs binary verification and dependency integrity checks in CI.

**Unsupported languages**  
No graph tool covers every language perfectly. CodeGraph has upstream Kotlin grammar issues and some C/C++ macro limitations. Graphify may not parse certain framework routing. Test on a small project before relying on it for a large codebase.

**Dynamic code**  
Static AST indexing cannot see reflection, dynamic imports, or runtime dependency injection. Use the graph for structural relationships and fall back to raw file reading, logging, or runtime tracing for dynamic behavior. Some tools (e.g. CodeGraph) support ingesting runtime traces to augment the graph.

---

## 14. Learning Path

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

## 15. Appendix: Graphify Quick Start

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

*Guide complete. All chapters linked in the Table of Contents. Section 3 provides the minimum-viable conceptual model for how parser, AST, and graph relate. Section 7 includes the deepened "Use Neither" conditions. Content targets experienced developers new to AI coding agents, MCP, and code knowledge graphs.*
