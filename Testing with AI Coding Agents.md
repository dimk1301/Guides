# AI Agents and Testing: A Practical Guide

*Agent-agnostic — applies to any coding agent (Claude Code, Cursor, Copilot, Codex, etc.)*

## Contents

- [TL;DR](#tldr)
- [1. The workflow](#1-the-workflow)
- [2. Spotting shallow tests](#2-spotting-shallow-tests)
- [3. Example: debounced SearchBox (React + Jest + Stryker)](#3-example-debounced-searchbox-react--jest--stryker)
  - [Tests — generated from the spec in a clean context](#tests--generated-from-the-spec-in-a-clean-context)
  - [Implementation — tests locked](#implementation--tests-locked)
  - [Guardrail — Stryker](#guardrail--stryker)
- [4. Agent setup (any agent, same five mechanisms)](#4-agent-setup-any-agent-same-five-mechanisms)
- [5. Extensions — add only what fills a gap](#5-extensions--add-only-what-fills-a-gap)
- [6. Brownfield: no tests, no spec](#6-brownfield-no-tests-no-spec)
- [Checklist](#checklist)

## 1. The workflow

| Step | What | Rule |
| --- | --- | --- |
| 1. Spec | Bulleted acceptance criteria + edge cases | Written before anything else |
| 2. Tests | Generate from the spec in a **clean context** (new session, or a subagent/task given only the spec + public interface) | No implementation exists yet |
| 3. Review | Human reads the *assertions*, not just pass/fail | Skipping this defeats the purpose |
| 4. Implement | Against the approved tests | "Do not modify the test file." If a test must change, the human re-approves the diff first |
| 5. Guardrail | Mutation test or red-team pass | Green ≠ checked. Each survivor → new test (clean context) → rerun |

Layer test types: unit (logic, edges) → property-based/fuzz (invariants) → integration (seams: auth, external calls, concurrency). Use snapshots sparingly — they freeze current output, bugs included.

Why the clean context matters: an agent writing tests for its own code anchors on what the code *does*, not what it was *supposed* to do — a suite that stays green while shipping bugs.

**Existing code with no tests?** The workflow still holds, but steps 1–2 invert: first reconstruct the spec from what the code does today and triage contract vs. bug, then write *characterization* tests that pin current behavior (green is the baseline, not the goal). Details and legacy-UI specifics: §6.

## 2. Spotting shallow tests

Watch for:

- No real assertion, or an assertion that restates the implementation
- Mocks that stub out the exact thing being tested
- `try/catch` swallowing real failures
- **Missing negative and boundary cases** — the most common gap
- Flakiness — run new suites several times before merging
- High-risk code (auth, payments, migrations): read the actual assertions, don't trust green CI

**Mutation testing > coverage.** Coverage shows code *ran*; mutation score shows a break would be *noticed*. A mutation tool flips operators and boundary constants ("mutants") and reruns the suite against each one — a mutant that survives is a real behavior change no test would catch.

| Language | Mutation tool |
| --- | --- |
| JS/TS | StrykerJS |
| Python | mutmut, cosmic-ray |
| Java | PIT |

Cheap alternative — red-team prompt (clean context): *"Here's the implementation and its tests. Find an input where the implementation is wrong but every test still passes."*

## 3. Example: debounced SearchBox (React + Jest + Stryker)

Stack: React, Jest + React Testing Library (`jest-environment-jsdom`, `@testing-library/jest-dom`), Stryker for the mutation guardrail.

Spec (written first): API fires 300ms after typing stops; empty/whitespace → no call, results cleared; loading state while pending; results list / "No results found" / "Something went wrong"; rapid retyping → only the latest query.

### Tests — generated from the spec in a clean context

Prompt: *"Write Jest + React Testing Library tests for this spec. Use fake timers for the debounce and mock `./api`. Do not write the implementation."*

```tsx
// src/SearchBox.test.tsx
import { render, screen, fireEvent } from "@testing-library/react";
import SearchBox from "./SearchBox";
import { searchApi } from "./api";

jest.mock("./api");
const mockedSearchApi = jest.mocked(searchApi);

beforeEach(() => { jest.useFakeTimers(); mockedSearchApi.mockReset(); });
afterEach(() => jest.useRealTimers());

test("whitespace-only input never calls the API", () => {
  render(<SearchBox />);
  fireEvent.change(screen.getByRole("textbox"), { target: { value: "  " } });
  jest.advanceTimersByTime(300);
  expect(mockedSearchApi).not.toHaveBeenCalled();
});

test("fires at 300ms, not 299ms (boundary)", () => {
  render(<SearchBox />);
  fireEvent.change(screen.getByRole("textbox"), { target: { value: "shoes" } });
  jest.advanceTimersByTime(299);
  expect(mockedSearchApi).not.toHaveBeenCalled();
  jest.advanceTimersByTime(1);
  expect(mockedSearchApi).toHaveBeenCalledWith("shoes");
});

test("rapid retyping searches only the latest query", () => {
  render(<SearchBox />);
  const input = screen.getByRole("textbox");
  fireEvent.change(input, { target: { value: "sho" } });
  jest.advanceTimersByTime(200);
  fireEvent.change(input, { target: { value: "shoes" } });
  jest.advanceTimersByTime(300);
  expect(mockedSearchApi).toHaveBeenCalledTimes(1);
  expect(mockedSearchApi).toHaveBeenCalledWith("shoes");
});

test("empty results: 'No results found' is shown", async () => {
  mockedSearchApi.mockResolvedValue([]);
  render(<SearchBox />);
  fireEvent.change(screen.getByRole("textbox"), { target: { value: "shoes" } });
  jest.advanceTimersByTime(300);
  expect(await screen.findByText("No results found")).toBeInTheDocument();
});
```

Also cover: loading state, API failure — one test each.

### Implementation — tests locked

Prompt: *"Implement `SearchBox` to satisfy the test file exactly as written. Do not edit the test file."*

```tsx
// src/SearchBox.tsx (effect logic)
useEffect(() => {
  if (!query.trim()) { setStatus("idle"); setResults([]); return; }
  setStatus("loading");
  const t = setTimeout(() => {
    searchApi(query)
      .then(d => { setResults(d); setStatus("success"); })
      .catch(() => setStatus("error"));
  }, 300);
  return () => clearTimeout(t);
}, [query]);
```

### Guardrail — Stryker

```bash
npm i -D @stryker-mutator/core @stryker-mutator/jest-runner && npx stryker init
```

```json
// stryker.conf.json — "perTest" runs only the tests that cover each mutant (~10× faster)
{ "testRunner": "jest", "mutate": ["src/SearchBox.tsx"], "coverageAnalysis": "perTest" }
```

Run `npx stryker run`. The report lists each surviving mutant with its file, line, and the exact change applied — treat every survivor as a missing test, not as noise to configure away.

Here, the mutants map directly onto the spec's rules:

| Mutant | Spec rule it probes | Result with the tests above |
| --- | --- | --- |
| `300` → `301` (debounce constant) | "fires at 300ms, not 299ms" | **Killed** — boundary test fails |
| `if (!query.trim())` → `if (false)` | "whitespace never calls the API" | **Killed** — whitespace test fails |
| remove `clearTimeout(t)` | "rapid retyping → only the latest query" | **Killed** — both `sho` and `shoes` get searched |
| `results.length > 0` → `>= 0` | "'No results found' shown *instead of* a list" | **Survived** — nothing asserted the list is absent |

The survivor is the interesting one: the empty-results test asserted "No results found" was *present*, never that the list was *absent*. Fix — add one line to that test:

```tsx
expect(screen.queryByRole("list")).not.toBeInTheDocument();
```

Rerun → 100%. That score — not green CI — is the proof.

## 4. Agent setup (any agent, same five mechanisms)

Whatever agent you use, encode the workflow in the tool rather than in your memory:

1. **A persistent rules file** the agent loads every session (`CLAUDE.md`, `.cursor/rules`, `AGENTS.md`, Copilot instructions). Write test conventions once: framework, run command, and "tests come from the spec, not from the implementation's current behavior."
2. **A read-only / plan mode** for the spec phase — have the agent draft its interpretation of acceptance criteria before writing anything; catch misunderstandings while they're still cheap.
3. **Fresh contexts by structure, not discipline** — a separate session, subagent, or task whose only inputs are the spec and the public interface. If your agent supports dedicated agent definitions (`.claude/agents/`, Cursor rulesets), give test-writing its own.
4. **Mechanical gates** — a hook, pre-commit, or CI step that runs the suite (ideally a mutation pass) automatically after edits, so "tests pass" is verified, not claimed.
5. **A packaged command or skill for the whole flow** — spec → tests → pause for review → implement → guardrail — instead of re-prompting each time. Most agents support this natively (slash commands, workflows, custom modes).

## 5. Extensions — add only what fills a gap

Every extension costs context and startup time. None are agent-specific: MCP servers work in any MCP-compatible agent, the rest are plain libraries any agent can drive.

| When you need | Use | Why |
| --- | --- | --- |
| Integration tests that don't mock the thing they test | **MSW** | Intercepts HTTP at the network layer |
| Invariant coverage beyond hand-picked cases | **fast-check** (JS) / **Hypothesis** (Python) | Property-based: hundreds of generated inputs |
| Contract checks at API seams | **Pact** | Verifies service contracts without a live backend |
| The agent to run and iterate on E2E tests itself | **Playwright MCP** (`@playwright/mcp`) | Browser automation over MCP |
| Debugging failing E2E | **chrome-devtools-mcp** | Perf traces, network, console |

## 6. Brownfield: no tests, no spec

Everything above assumes greenfield. Most agent work isn't. Two steps invert:

1. **Reconstruct the spec** — in a fresh context, have the agent read the legacy module and draft acceptance criteria for *what it does today*. You review and triage: which behaviors are contract, which are accidents/bugs. This review replaces the test-review gate, and the anchoring risk is structural — the agent reads the implementation by definition — so inject intent: say what the feature *should* do, not just what it does.
2. **Characterization tests** — the agent writes tests asserting current behavior; the suite starts green against unmodified code. Green is the baseline, not the goal. Validate with the red-team prompt (§2): *"Find an input where every test passes but the behavior is wrong."*
3. **Then run the §1 workflow on the change itself** — spec → tests (clean context) → modify → guardrail.

Amendments to the rules:

- Characterization tests *may* be edited — when you deliberately change behavior, update the pinned expectation and call it out in the PR.
- Cover the blast radius of the change, not the app. Don't retrofit tests for code you're not touching.

**Legacy UI specifically** — often the only viable layers are component tests (after extraction) or Playwright E2E:

- Prefer role/text queries (`getByRole`, `getByLabelText`) over CSS/XPath; adding missing labels/roles is part of the change.
- Record current flows with Playwright codegen; have the agent clean them up in a fresh context. Don't let the agent invent flows from reading code.
- Visual regression (screenshot diffs) is the flakiest option — if used, mask dynamic regions.
- Rewriting the UI (e.g., jQuery → React)? Pin legacy behavior with E2E first, then require the new implementation to pass the same suite.

## Checklist

- [ ] Spec written before tests
- [ ] Tests written before implementation, in a clean context
- [ ] Human reviewed tests (negative/boundary cases present)
- [ ] Implementation against approved tests; tests unmodified (any test change re-approved)
- [ ] Guardrail run; every survivor converted to a test and rerun
- [ ] Conventions in the agent's rules file; a mechanical gate enforces them
- [ ] *(Brownfield only)* Reconstructed spec triaged — contract vs. bug decided per behavior
