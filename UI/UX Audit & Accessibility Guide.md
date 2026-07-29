# UI/UX Audit & Accessibility Guide — 2026 Edition

*A practical, step‑by‑step guide for VS Code + GitHub Copilot, covering manual checks, static analysis, automated browser testing, heuristic review, and CI enforcement.*

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Layer 0 & 1 – Manual & One‑Click Audits](#layer-0--1--manual--oneclick-audits)
- [Layer 2 – Static Linting in VS Code](#layer-2--static-linting-in-vs-code)
- [Layer 2.5 – Component‑Level A11y (optional)](#layer-25--componentlevel-a11y-optional)
- [Layer 3 – Real‑Browser Testing with Playwright MCP](#layer-3--realbrowser-testing-with-playwright-mcp)
- [Layer 4 – Heuristic / Usability Review via Copilot Chat](#layer-4--heuristic--usability-review-via-copilot-chat)
- [Layer 5 – CI Enforcement (optional)](#layer-5--ci-enforcement-optional)
- [Which Layers Do I Need?](#which-layers-do-i-need)
- [Troubleshooting MCP Servers](#troubleshooting-mcp-servers)
- [Appendix: Full Example Repository](#appendix-full-example-repository)

---

## Prerequisites (do this once)

- **VS Code** 1.110+ (latest stable).
- **GitHub Copilot** and **GitHub Copilot Chat** extensions installed and signed in.
- **Node.js 20+** – Node 18 is no longer reliably supported by the MCP toolchain; upgrade if needed.
- **Enable Copilot Agent Mode** – Agent mode is now the default in the chat dropdown; no special setting is required. In the Copilot Chat panel, ensure the dropdown is set to **“Agent”** (not “Ask” or “Edit”).
- (Optional) Install the **Playwright** browser binaries globally if you plan to use the Playwright MCP server:  
  `npx playwright install`

> 💡 **What is MCP?**  
> The Model Context Protocol (MCP) is a standard that lets AI agents (like Copilot) call external tools directly. Instead of copying code, you tell Copilot to *do* something, and it uses the MCP server to run actions (e.g., open a browser, take a screenshot). This guide focuses on MCP servers for testing and automation.

---

## Layer 0 & 1 – Manual & One‑Click Audits

*No MCP required – baseline manual checks.*

### Accessibility & Performance Quick‑Checks

1. **Lighthouse** (built into Chrome DevTools)  
   - Open your page, press F12, go to the **Lighthouse** tab.  
   - Run an audit with **Accessibility**, **Performance**, and **Best Practices** enabled.  
   - Save the report for later reference.

2. **axe DevTools** (browser extension)  
   - Install from the [Chrome Web Store](https://chrome.google.com/webstore/detail/axe-devtools-web-accessib/lhdoppojpmngadmnindnejefpokejbdd).  
   - Click the extension icon on any page to run an accessibility scan.  
   - **Manual validation** is still vital – automated tools catch ~30‑40% of issues.


---

## Layer 2 – Static Linting in VS Code

*Configuration files only – no MCP.*

### ESLint with `jsx-a11y` (flat config)

1. Install the plugin:  
   ```bash
   npm install --save-dev eslint-plugin-jsx-a11y
   ```

2. Create or update your `eslint.config.js` (ESLint 9+ flat config):  
   ```javascript
   import jsxA11y from 'eslint-plugin-jsx-a11y';

   export default [
     // ... your other configs
     jsxA11y.configs.recommended,
   ];
   ```
   > If you’re still on ESLint 8, you can keep `.eslintrc` with `"plugin:jsx-a11y/recommended"` – but we recommend migrating to flat config.

3. VS Code’s built-in ESLint extension will now highlight accessibility violations in real time.

### jest‑axe (for unit tests)

1. Install:  
   ```bash
   npm install --save-dev jest-axe
   ```
   > ⚠️ **Note:** `@types/jest-axe` is no longer required – jest-axe ships its own types. Also be aware of dependency conflicts with ESLint 10; if you encounter errors, try pinning jest‑axe to `^4.0.0`.

2. In your test setup file (e.g., `jest.setup.js`), extend Jest:  
   ```javascript
   import { toHaveNoViolations } from 'jest-axe';
   expect.extend(toHaveNoViolations);
   ```

3. **Verification:** Write a simple test:  
   ```javascript
   import { axe } from 'jest-axe';

   test('my component has no a11y violations', async () => {
     const { container } = render(<MyComponent />);
     const results = await axe(container);
     expect(results).toHaveNoViolations();
   });
   ```
   Run `npm test` – if the test passes, you’re all set.

---

## Layer 2.5 – Component‑Level A11y (optional)

If you use **Storybook**, add the official accessibility addon to get per‑component axe scans directly in the UI:

```bash
npm install --save-dev @storybook/addon-a11y
```

Add it to your `.storybook/main.js` and see a new **Accessibility** panel when viewing stories – great for catching issues during component development.

---

## Layer 3 – Real‑Browser Testing with Playwright MCP

*Core MCP setup – connects Copilot to a live browser.*

### Zero‑Setup Alternative (VS Code Built‑in Tools)

Since early 2026, VS Code provides native browser tools (no MCP required) for simple tasks:
- Open the command palette and run **“Copilot: Open Browser”** to launch a controlled browser.
- Use prompts like:  
  *“Take a screenshot of localhost:3000 and check for overflow”* – Copilot can call the built‑in `screenshotPage` tool.

For **advanced** testing (tab‑traversal, multi‑page flows, WCAG snapshots), we recommend the official Playwright MCP server.

### Installing Playwright MCP

**Option A – One‑click (fastest):**  
Visit the [Playwright MCP page](https://playwright.dev/mcp/clients/vscode) and click **“Install in VS Code”** – the extension will auto‑configure everything.

**Option B – Manual (more control):**  
Create a `.vscode/mcp.json` file in your project root (this is the 2026 standard – **not** `settings.json`):

```json
{
  "servers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

> ⚠️ **Important:** VS Code looks for MCP configs in `.vscode/mcp.json` (workspace) or your user‑level `mcp.json`. The old `settings.json` approach is deprecated.

After saving, VS Code will show a **“Start”** link above the JSON; click it to launch the MCP server. The server status appears in the status bar.

### Using Playwright MCP in Copilot Chat

1. In the Copilot Chat panel, ensure the dropdown is set to **“Agent”**.
2. Click the wrench/tools icon and confirm **“playwright”** is checked.
3. Now you can issue commands in plain English:

   - *“Navigate to localhost:3000, tab through the page with only the keyboard, and report any focus traps.”*
   - *“Take screenshots of the signup form at 375px, 768px, and 1440px; check for horizontal overflow.”*
   - *“Run an accessibility snapshot of the checkout flow and list all ARIA violations.”*

4. **Verification:** After a command, Copilot will show a transcript of which tools it called. If you don’t see any tool calls, the server may not be running – check the status bar or open the MCP panel.

### Official Axe‑Core Integration (instead of unverified community servers)

For dedicated WCAG reporting, use the official `@axe-core/playwright` package **with** Playwright MCP’s generic browser control. You can run axe checks via MCP’s `browser_evaluate`:

- Prompt: *“Use Playwright to run axe-core on the current page and return a summary of violations.”*

Copilot’s agent mode can also generate a script that runs `@axe-core/playwright` in a standalone test – see Layer 5 for CI integration.

> 💡 **Token‑Efficient Alternative:** For high‑volume usage (e.g., scanning hundreds of pages), consider the **Playwright CLI** (`@playwright/cli`) – it’s 4× more token‑efficient than MCP and can be triggered via Copilot’s terminal execution.

---

## Layer 4 – Heuristic & Usability Review via Copilot Chat

*No extra MCP server required – pure prompting.*

1. Use the Playwright MCP (or VS Code built‑in browser) to capture screenshots of your 4‑5 core screens.
2. Drag those images into the Copilot Chat input, or ask the agent to save them to a folder you can reference.
3. Prompt:  
   *“Evaluate these screens against Nielsen’s 10 usability heuristics and common dark‑pattern checklists. List violations by screen.”*
   Copilot’s multimodal reasoning will analyse the visuals and provide a structured report.

4. **Optional – Structured Audits:** If you prefer a formal tool, you can install the `ux-mcp-server` (from @elsahafy). Add it to your `.vscode/mcp.json` as another server:

   ```json
   "ux-mcp": {
     "command": "npx",
     "args": ["ux-mcp-server@latest"]
   }
   ```

   Then in Agent mode, ask: *“Call the complete_ux_audit tool for my homepage screenshot.”*

> ⚠️ **Limitation:** Automated heuristics are expert reviews, not real user testing. Always supplement with actual user feedback when possible.

---

## Layer 5 – CI Enforcement (optional)

*GitHub Actions + pre‑commit hooks – no direct Copilot interaction.*

This layer is **adjacent** to the core VS Code flow, but valuable for teams.

1. **Pre‑commit hook:** Add `eslint` and `jest-axe` to your lint‑staged config so every commit fails if violations are introduced.

2. **GitHub Actions PR check:**  
   In your workflow, run:
   ```yaml
   - run: npm run lint
   - run: npm test
   ```
   This ensures ESLint and jest‑axe run on every pull request.

3. **Nightly Playwright scan:**  
   Set up a scheduled workflow that spins up a headless Playwright test against your staging URL. Use `@axe-core/playwright` to scan the page and post a report as a workflow artifact.

4. **Ask Copilot to generate the YAML:**  
   In Agent mode, prompt: *“Write a GitHub Actions workflow that runs ESLint, jest‑axe, and a nightly Playwright axe scan against staging.”* – Copilot will produce the YAML using your repo’s structure and the MCP tools it just used.

---

## Which Layers Do I Need?

| Project Type                 | Recommended Layers |
|------------------------------|---------------------|
| **Solo dev / landing page**  | 0, 1, 2 (maybe 2.5) |
| **Small team / marketing**   | 0, 1, 2, 3, 4       |
| **Enterprise SaaS / product**| All 5 layers        |
| **Component library**        | 2.5, 2, 3 (optional) |

Use Layer 3 (Playwright MCP) if you need to test complex interactions (multi‑step forms, modals, keyboard traps). Skip it if you only have static pages.

---

## Troubleshooting MCP Servers

| Symptom | Probable Cause | Fix |
|---------|---------------|-----|
| MCP server not starting | Node version < 20 | Run `node -v`; upgrade to 20+ |
| Browser not launching | Missing browser binaries | Run `npx playwright install` |
| Copilot doesn’t see MCP tools | Server not started or not checked in tools panel | Click “Start” in `.vscode/mcp.json` and check the wrench icon |
| `npx` fails with permission errors | Corporate proxy or offline | Set `NPM_CONFIG_REGISTRY` or download binaries manually |
| Tools are called but no output | Server may be stuck; restart VS Code | Reload window and restart the server |


---

*Guide updated for 2026 tooling – if you encounter any outdated steps, please file an issue on the repository.*
