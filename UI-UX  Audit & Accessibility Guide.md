# Complete UI/UX & Accessibility Audit Guide
## VS Code + GitHub Copilot + MCP — 2026 Edition

> **Last updated:** July 2026  
> **Target audience:** Frontend developers, QA engineers, and product teams building web applications (React/Vue/Angular/Static)  
> **Time to full setup:** 45–90 minutes (Layers 0–3), 2–3 hours (all 5 layers)

---

## Table of Contents

1. [How to Use This Guide](#how-to-use-this-guide)
2. [Prerequisites](#prerequisites)
3. [Layer 0 & 1: Manual Baseline Scanning](#layer-0--1-manual-baseline-scanning)
4. [Layer 2: Static Linting & Component Testing](#layer-2-static-linting--component-testing)
5. [Layer 3: Real-Browser Testing with AI Agents](#layer-3-real-browser-testing-with-ai-agents)
6. [Layer 4: Heuristic & Usability Review](#layer-4-heuristic--usability-review)
7. [Layer 5: CI Enforcement](#layer-5-ci-enforcement)
8. [Troubleshooting](#troubleshooting)
9. [Appendix: Tool Comparison Matrix](#appendix-tool-comparison-matrix)

---

## How to Use This Guide

This guide uses a **5-layer progressive model**. Each layer catches issues the previous layer misses, and each layer is more expensive (time/compute) than the last.

| Layer | Name | Speed | Cost | Catches |
|-------|------|-------|------|---------|
| 0 & 1 | Manual Baseline | ⚡ Fast | Free | Critical a11y, performance, visual bugs |
| 2 | Static Linting | ⚡ Fast | Free | Code-level a11y violations |
| 3 | Real-Browser AI Testing | 🐢 Medium | Low | Runtime ARIA, focus, responsive issues |
| 4 | Heuristic Review | 🐢 Medium | Low (AI tokens) | UX patterns, dark patterns, cognitive load |
| 5 | CI Enforcement | 🐌 Slow | CI minutes | Regressions, nightly drift |

> **💡 You do not need all 5 layers for every project.**
>
> | Project Type | Recommended Layers |
> |-------------|-------------------|
> | Personal blog / landing page | 0, 1, 2 (core) |
> | Small business site (5–10 pages) | 0, 1, 2 (core), 3 |
> | SaaS app / dashboard | 0, 1, 2 (enhanced), 3, 4 |
> | E-commerce / healthcare / gov | **All 5** |
> | Component library / design system | 0, 1, 2 (enhanced + Storybook) |
>
> **Core Layer 2** = `eslint-plugin-jsx-a11y` + axe Linter + `jest-axe`  
> **Enhanced Layer 2** = Core + `design-lint` + `defensive-css` + `apca-w3`

---

## Prerequisites

Do this once per machine.

1. **VS Code** updated to the latest stable version (1.110+).
2. **GitHub Copilot** and **GitHub Copilot Chat** extensions installed and signed in.
3. **Node.js v20+** installed. Verify with:
   ```bash
   node --version
   ```
   > ⚠️ **Node.js 18 is no longer sufficient.** The Playwright MCP server and its transitive dependencies (e.g., Neon database driver) require Node 20+. If you are on v18, upgrade via [nodejs.org](https://nodejs.org/) or your package manager.
4. **Agent Mode is enabled by default** in VS Code 1.100+. You will see an "Agent" option in the Copilot Chat mode dropdown. If you do not see it, update VS Code.

---

## Layer 0 & 1: Manual Baseline Scanning

> **Goal:** Establish a baseline of critical issues before writing any code or configuring automation.  
> **No installation required.**

### Layer 0: Lighthouse (Performance + Accessibility + Best Practices)

1. Open your site in Chrome.
2. Press `F12` → **Lighthouse** tab.
3. Check **Accessibility**, **Performance**, and **Best Practices**.
4. Run the audit on your 3–5 most critical pages (homepage, signup, checkout, dashboard).
5. **Export the report** (JSON or HTML) and save it as your baseline.

> ✅ **Verification:** You have a Lighthouse JSON/HTML report saved with scores for each category.

### Layer 1: axe DevTools + Screen Reader Spot Check

1. Install the **axe DevTools** browser extension from the [Chrome Web Store](https://chromewebstore.google.com/detail/axe-devtools-web-accessib/lhdoppojpmngadmnindnejefpokejbdd).
2. Run a **Full Page Scan** on the same pages you tested in Layer 0.
3. Export the results.
4. **Screen reader spot check (critical):** Automated tools catch only ~30–40% of accessibility issues. Spend 10 minutes navigating your top page with:
   - **Windows:** [NVDA](https://www.nvaccess.org/download/) (free)
   - **macOS:** VoiceOver (`Cmd + F5` to toggle)
   - Verify you can reach every interactive element via keyboard (`Tab`, `Shift+Tab`, `Enter`, `Space`, `Arrow keys`).

> ✅ **Verification:** You have axe DevTools export + a note about any focus traps or unreachable elements found during screen reader testing.

> 📺 **Further learning:** [axe DevTools tutorial — YouTube](https://www.youtube.com/watch?v=Zsq6UA3yXAI)

---

## Layer 2: Static Linting & Component Testing

> **Goal:** Catch accessibility violations while you write code, before they reach the browser.  
> **No MCP server needed.** These tools run inside VS Code or your test suite.

### 2A: ESLint with jsx-a11y (Editor-time linting)

This underlines accessibility issues directly in your editor as you type.

**If your project uses ESLint v9+ (flat config):**

```bash
npm install --save-dev eslint-plugin-jsx-a11y
```

Then in `eslint.config.js`:

```javascript
import jsxA11y from 'eslint-plugin-jsx-a11y';

export default [
  // ...your other configs
  jsxA11y.flatConfigs.recommended,
];
```

**If your project still uses legacy `.eslintrc`:**

```bash
npm install --save-dev eslint-plugin-jsx-a11y
```

Then in `.eslintrc.json`:

```json
{
  "extends": ["plugin:jsx-a11y/recommended"]
}
```

> 💡 **Tip:** Restart VS Code after installing. You should now see red squiggles under inaccessible patterns like `<div onClick={...}>` without `role` or `tabIndex`.

> ✅ **Verification:** Open a component with a known a11y issue (e.g., a clickable `<div>` without keyboard handlers). You should see an ESLint warning in the editor.

### 2B: jest-axe (Test-time assertions)

```bash
npm install --save-dev jest-axe
```

In your test setup file (e.g., `jest.setup.js`):

```javascript
import { toHaveNoViolations } from 'jest-axe';
expect.extend(toHaveNoViolations);
```

Example component test:

```javascript
import { render } from '@testing-library/react';
import { axe } from 'jest-axe';
import MyComponent from './MyComponent';

it('should have no accessibility violations', async () => {
  const { container } = render(<MyComponent />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

> 💡 **Tip:** You can ask Copilot Chat (Ask mode, no MCP needed): *"Add a jest-axe accessibility test for this component"* and paste your component code.

> ✅ **Verification:** Run `npm test`. The test should pass for accessible components and fail with a detailed report for inaccessible ones.

### 2C: Storybook a11y Addon (For component libraries)

If you maintain a component library or design system, add the official Storybook accessibility addon:

```bash
npm install --save-dev @storybook/addon-a11y
```

This adds an "Accessibility" panel to every Storybook story, running axe-core per component with visual regression support.

> ✅ **Verification:** Open Storybook. Every story should show an "Accessibility" tab with pass/fail status.

### 2D: axe Accessibility Linter (VS Code Extension)

For the most reliable editor-time accessibility checking, install Deque's official **axe Accessibility Linter** extension from the VS Code marketplace. Unlike `eslint-plugin-jsx-a11y`, it is powered by the same `axe-core` engine used in Lighthouse and axe DevTools, with a **zero false-positive** design philosophy.

**What it checks:**
- React, Vue, Angular, HTML, and Markdown files
- Missing alt text, invalid ARIA, insufficient color contrast, missing form labels, and more
- Runs continuously as you type (no manual invocation needed)

**Install:** Search "axe Accessibility Linter" in the VS Code Extensions panel, or install from the [marketplace](https://marketplace.visualstudio.com/items?itemName=deque-systems.vscode-axe-linter).

**CLI/API version for CI:**
```bash
npm install --save-dev @axe-devtools/linter
```

> ✅ **Verification:** Open a file with a known a11y issue (e.g., `<img src="logo.png">` without `alt`). The axe Linter should underline it immediately with a tooltip explaining the violation and how to fix it.

> 💡 **Tip:** The axe Linter is stricter and more accurate than `eslint-plugin-jsx-a11y`, but the two can coexist. Use axe Linter as your primary source of truth, and keep `jsx-a11y` as a secondary check.

---

### 2E: @lapidist/design-lint (Design Token Enforcement)

If your team uses a design system with tokens (colors, spacing, typography), this tool enforces that your code uses tokens instead of raw values — preventing visual drift and accessibility regressions.

**Install:**
```bash
npm install --save-dev @lapidist/design-lint
```

**Configure** in `design-lint.config.js`:
```javascript
module.exports = {
  tokens: {
    colors: ['primary', 'secondary', 'danger', 'success', 'warning'],
    spacing: ['xs', 'sm', 'md', 'lg', 'xl'],
    typography: ['body', 'heading', 'caption']
  },
  sources: ['src/**/*.{js,jsx,ts,tsx,css,scss}']
};
```

**What it catches:**
- Raw hex codes like `#ff0000` instead of `var(--color-danger)`
- Magic numbers like `margin: 12px` instead of `spacing.md`
- Inline styles that bypass your design system
- Use of raw `<button>` when your system provides `<Button>`

> ✅ **Verification:** Run `npx design-lint`. It should flag any raw values in your codebase and suggest token replacements.

> 💡 **Tip:** Add `design-lint` to your pre-commit hook or CI pipeline to block design-token violations before they reach production.

---

### 2F: stylelint-plugin-defensive-css (Defensive CSS Practices)

A mature Stylelint plugin that catches CSS anti-patterns that hurt accessibility and responsive design — particularly fixed `px` values that break zoom, text scaling, and adaptive layouts.

**Install:**
```bash
npm install --save-dev stylelint stylelint-plugin-defensive-css
```

**Configure** in `stylelint.config.js`:
```javascript
export default {
  plugins: ['stylelint-plugin-defensive-css'],
  rules: {
    'plugin/defensive-css': true
  }
};
```

**What it catches:**
- Fixed `px` font sizes (breaks user zoom preferences)
- Missing `flex-wrap` on flex containers (causes overflow on small screens)
- Missing fallback colors (fails in high-contrast mode)
- `outline: none` without a replacement focus style

> ✅ **Verification:** Run `npx stylelint "src/**/*.{css,scss}"`. You should see warnings for fixed `px` values and missing defensive patterns.

---

### 2G: apca-w3 (APCA Contrast Validation)

WCAG 2.1 contrast math is known to be inaccurate on modern displays (OLED, high-DPI). **APCA** (Accessible Perceptual Contrast Algorithm) is the W3C's successor standard. Use this package to validate contrast programmatically.

**Install:**
```bash
npm install --save-dev apca-w3
```

**Example test:**
```javascript
import { calcAPCA } from 'apca-w3';

it('meets APCA contrast for body text', () => {
  const contrast = calcAPCA('#ffffff', '#0066cc'); // text, background
  // APCA returns a perceptual lightness contrast value
  // 75+ is recommended for body text, 60+ for large text
  expect(Math.abs(contrast)).toBeGreaterThanOrEqual(75);
});
```

> ✅ **Verification:** Run your test suite. Any color pair failing APCA thresholds will fail the test with the calculated contrast value.

> 💡 **Tip:** If your design system defines color tokens, add APCA tests to your CI to catch contrast regressions when designers update the palette.

---

### 2H: UI UX Suite — All-in-One Design Linter (Experimental)

If you prefer a single tool over the stack above, **UI UX Suite** (`ui-ux-suite`) claims to combine design linting, APCA contrast, and 24 Laws of UX into one package. It operates as both a CLI and MCP server.

**Install:**
```bash
npx ui-ux-suite audit --src ./src --format html
```

> ⚠️ **Experimental warning:** This tool has minimal community adoption (3 GitHub stars) and unverified accuracy. Test it on a non-critical branch first. If it produces too many false positives, rely on the established stack (2D–2G) instead.


---


## Layer 3: Real-Browser Testing with AI Agents

> **Goal:** Use a real browser (not just static analysis) to test keyboard navigation, responsive layouts, ARIA trees, and dynamic interactions.  
> **This is the first layer that requires an MCP server.**

### What is MCP?

**MCP (Model Context Protocol)** is an open standard that lets AI agents like Copilot call external tools — in this case, a real browser. Think of it as a "USB-C port for AI applications." The Playwright MCP server gives Copilot the ability to navigate pages, click elements, take screenshots, and run accessibility audits inside an actual Chromium instance.

### Option A: VS Code Built-in Browser Tools (Zero Setup — Recommended for Simple Tasks)

As of VS Code 1.110 (February 2026), VS Code includes **native browser agent tools** that work without installing anything:

- `openBrowserPage` — navigate to a URL
- `screenshotPage` — capture the viewport
- `readPage` — extract text/structure
- `closeBrowserPage` — clean up

**When to use this:** Quick screenshot comparisons, content verification, or if you are behind a corporate proxy that blocks `npx`.

**Limitation:** These tools are read-only and less powerful than full Playwright MCP. They cannot simulate complex keyboard interactions or run axe-core scans.

### Option B: Playwright MCP Server (Full Power — Recommended for Thorough Audits)

#### Step 1: Configure the MCP server

Create or open `.vscode/mcp.json` in your project root:

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

> ⚠️ **Note:** In older guides you may see MCP config inside `settings.json`. As of 2026, the canonical location is `.vscode/mcp.json` (workspace-level) or your user profile's `mcp.json` (global). Using `settings.json` is deprecated.

Save the file. VS Code will show a **"Start"** link above the JSON block — click it. Alternatively, open the Command Palette (`Ctrl+Shift+P`) → **MCP: Start Server** → select `playwright`.

#### Step 2: Activate in Copilot Chat

1. Open Copilot Chat (`Ctrl+Alt+I`).
2. Switch the mode dropdown from **"Ask"** to **"Agent"**.
3. Click the **wrench/tools icon**.
4. Confirm **"playwright"** is checked as an active MCP server.

> 💡 **Tip:** Name your server keys clearly (`playwright`, `playwright-axe`, `ux-mcp`). Copilot picks tools by key name when reasoning about which server to call. Ambiguous names cause it to select the wrong server mid-task.

#### Step 3: Drive it with natural language

With Agent Mode active, try these prompts:

- *"Navigate to `localhost:3000`, tab through the entire page with keyboard only, and report any place focus is lost or trapped."*
- *"Take screenshots of the signup form at 375px, 768px, and 1440px widths and flag any overflow or truncation."*
- *"Run an accessibility snapshot of the checkout flow and list ARIA violations."*
- *"Fill out the contact form with invalid email format, submit it, and verify the error message is announced to screen readers."*

> ✅ **Verification:** After running a prompt, check the **Tool Calls** panel in Copilot Chat. You should see a transcript of `browser_navigate`, `browser_screenshot`, or `browser_click` calls. If you see no tool calls, the MCP server is not connected.

### Option C: Playwright CLI (Token-Efficient Alternative)

If you plan to run hundreds of browser tasks (e.g., nightly CI or large-scale audits), use the official Playwright CLI instead of MCP. Released in early 2026, it consumes up to **4× fewer tokens** per session than the MCP server:

```bash
npx @playwright/cli@latest
```

This is a headless automation tool designed specifically for AI agents. Use it when MCP token costs become a concern.

### Option D: Axe MCP Server (Enterprise-Grade, Paid)

If you have budget and need the most accurate, enterprise-grade browser accessibility testing with AI-powered remediation, use **Deque's official Axe MCP Server**. It is built by the creators of `axe-core` (the engine behind Lighthouse, axe DevTools, and most accessibility tools).

**What it provides:**
- **`analyze`** — Runs axe DevTools in a real browser against any URL, with responsive viewport testing, authenticated pages, and partial-page scans.
- **`remediate`** — Generates AI-powered, code-level fix guidance that Copilot/Cursor can apply automatically.
- **`igt`** — Runs automated Intelligent Guided Tests (e.g., keyboard navigation, form filling) without manual input.

**Setup:**
```json
{
  "servers": {
    "axe-mcp": {
      "command": "npx",
      "args": ["@axe-devtools/mcp-server"]
    }
  }
}
```

**Requirements:**
- Paid **Axe DevTools for Web** subscription + API key
- Node.js 20+

**When to choose this over Playwright MCP:**
- You need **verified, zero false-positive** accessibility scans (axe-core is the industry standard)
- You want **AI-generated remediation code** (not just violation lists)
- You need **automated keyboard/form testing** (IGT) without writing custom scripts
- Your organization requires **enterprise security/compliance** (Deque is SOC 2 compliant)

> 💡 **Tip:** If budget is a constraint, use the free stack in Layer 2 (axe Linter + jest-axe) + Playwright MCP in Layer 3. The Axe MCP Server is a force-multiplier, not a requirement.

### Optional: Dedicated axe-core Integration

For WCAG-specific reports (rather than general browser control), add the `playwright-axe-mcp` server alongside Playwright:

```json
{
  "servers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    },
    "playwright-axe": {
      "command": "npx",
      "args": ["playwright-axe-mcp"]
    }
  }
}
```

> ⚠️ **Note:** `playwright-axe-mcp` is a community server. For production use, consider the official `@axe-core/playwright` package in your own test scripts instead.

---

## Layer 4: Heuristic & Usability Review

> **Goal:** Evaluate your UI against established usability principles (Nielsen's heuristics, dark-pattern checklists) and cognitive accessibility.  
> **No new MCP server required** — this is a prompting exercise using Copilot's multimodal reasoning.

### Method: Screenshot + Prompt Review

1. **Capture screenshots** of your 4–5 core screens using the Playwright MCP from Layer 3 (or VS Code's built-in screenshot tool for simpler pages).
2. **Drag the screenshots** directly into the Copilot Chat input, or save them to a folder Copilot can reference.
3. **Prompt Copilot:**

   > *"Evaluate these screens against Nielsen's 10 usability heuristics and common dark-pattern checklists. List violations by screen, severity (critical/warning/nice-to-have), and suggest specific fixes."*

4. Copilot will analyze the images and provide a structured heuristic audit.

> 💡 **Tip:** For a more focused review, add constraints to your prompt:
> - *"Focus on cognitive accessibility for users with ADHD."*
> - *"Flag any deceptive patterns that might violate GDPR or CCPA consent requirements."*
> - *"Check color contrast ratios against WCAG 2.1 AA standards."*

### Optional: Structured UX Audit via MCP

If you prefer a tool-call-based audit over free-form prompting, install the **ux-mcp-server**:

```json
{
  "servers": {
    "ux-mcp": {
      "command": "npx",
      "args": ["ux-mcp-server"]
    }
  }
}
```

This exposes a `complete_ux_audit` tool with 28 knowledge bases and 23 analysis tools. It returns structured JSON rather than prose, which is easier to feed into reports or tickets.

> 📖 **Reference:** [ux-mcp-server documentation — GitHub](https://github.com/elsahafy/ux-mcp-server)

> ✅ **Verification:** You should have a written heuristic review document (from Copilot) or a structured JSON audit (from ux-mcp-server) covering all major screens.

> ⚠️ **Important limitation:** AI heuristic review and automated tools are **not a substitute for user testing**. Budget for 3–5 moderated usability tests with real users before major releases.

---

## Layer 5: CI Enforcement

> **Goal:** Prevent accessibility regressions by making checks mandatory in your pull request pipeline.  
> **This layer does not use Copilot directly** — it uses the tools from Layers 2 and 3 inside GitHub Actions.

### Step 1: PR Checks (Block Merges on Violations)

Add this job to `.github/workflows/ci.yml`:

```yaml
name: CI
on: [pull_request]

jobs:
  accessibility:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci

      # Layer 2: Static linting
      - run: npx eslint . --max-warnings=0

      # Layer 2: Unit tests with jest-axe
      - run: npm test -- --coverage

      # Layer 3: Install Playwright browsers
      - run: npx playwright install --with-deps chromium

      # Layer 3: Run Playwright + axe-core against staging
      - run: npx playwright test --project=chromium
        env:
          BASE_URL: ${{ secrets.STAGING_URL }}

      # Layer 0: Lighthouse CI (optional but recommended)
      - name: Lighthouse CI
        run: |
          npm install -g @lhci/cli@0.14.x
          lhci autorun
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
```

> 💡 **Tip:** You can ask Copilot Agent Mode (with Playwright MCP active) to draft this YAML for you directly in VS Code. Since it understands your repo structure and the Playwright MCP tool calls from Layer 3, it can generate a workflow tailored to your project.

### Step 2: Nightly Full Audit

Schedule a nightly job that runs a comprehensive axe-core scan against your staging environment and posts results as a PR comment or workflow artifact:

```yaml
  nightly-audit:
    runs-on: ubuntu-latest
    schedule:
      - cron: '0 2 * * *'  # 2 AM UTC daily
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npx playwright test tests/accessibility-audit.spec.ts
        env:
          BASE_URL: https://staging.yoursite.com
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: accessibility-report
          path: playwright-report/
```

> ✅ **Verification:** Create a PR that introduces an accessibility violation (e.g., remove an `alt` attribute). The PR check should fail and block merging.

### Performance Budgets

Add a `budget.json` to your Lighthouse CI config to fail builds on performance regression:

```json
[
  {
    "path": "/*",
    "resourceSizes": [
      { "resourceType": "document", "budget": 18 },
      { "resourceType": "total", "budget": 500 }
    ],
    "timings": [
      { "metric": "interactive", "budget": 3500 },
      { "metric": "first-meaningful-paint", "budget": 2000 }
    ]
  }
]
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `npx @playwright/mcp@latest` fails with EACCES | Node version < 20 | Upgrade to Node.js 20+: `nvm install 20 && nvm use 20` |
| MCP server shows "Start" but never connects | Corporate proxy blocking CDN | Use VS Code built-in browser tools (Option A) instead, or configure proxy in `.npmrc` |
| `browser_navigate` times out | Staging server not running | Ensure your dev server (`localhost:3000`) is active before running Agent prompts |
| ESLint not showing a11y warnings | Flat config not applied correctly | Verify `eslint.config.js` exports the `jsxA11y.flatConfigs.recommended` object |
| jest-axe fails with cryptic error | `@types/jest-axe` conflict with ESLint 9+ | Remove `@types/jest-axe` if installed; use `jest-axe` directly (it includes its own types as of 2026) |
| Copilot picks wrong MCP server mid-task | Ambiguous server names | Rename keys in `mcp.json` to be explicit: `playwright`, `playwright-axe`, `ux-mcp` |
| Screenshots show blank pages | Browser binaries not installed | Run `npx playwright install chromium` |
| Lighthouse CI fails on first run | No baseline established | Run `lhci wizard` locally once to establish baseline |

---

## Appendix: Tool Comparison Matrix

| Tool | Layer | Cost | Setup | Best For |
|------|-------|------|-------|----------|
| Chrome Lighthouse | 0 | Free | Built-in | Performance + a11y baseline |
| axe DevTools | 1 | Free (basic) | Browser extension | Detailed WCAG violation reports |
| NVDA / VoiceOver | 1 | Free | Install | Real screen reader behavior |
| `eslint-plugin-jsx-a11y` | 2 | Free | npm + config | Catch a11y issues while coding |
| axe Accessibility Linter | 2 | Free | VS Code extension | Zero false-positive editor a11y |
| `@lapidist/design-lint` | 2 | Free | npm + config | Design token enforcement |
| `stylelint-plugin-defensive-css` | 2 | Free | npm + Stylelint config | Defensive CSS / responsive safety |
| `apca-w3` | 2 | Free | npm + test script | APCA contrast validation |
| `jest-axe` | 2 | Free | npm + test setup | Automated a11y in unit tests |
| `@storybook/addon-a11y` | 2 | Free | npm + Storybook | Per-component a11y in isolation |
| VS Code built-in browser tools | 3 | Free | None | Zero-setup screenshot/navigate |
| Playwright MCP | 3 | Token costs | `mcp.json` | Full browser automation via AI |
| Playwright CLI | 3 | Lower tokens | npm install | High-volume automation |
| Axe MCP Server | 3 | Paid | `mcp.json` + subscription | Enterprise-grade a11y + remediation |
| `playwright-axe-mcp` | 3 | Token costs | `mcp.json` | Structured axe reports via MCP |
| `@axe-core/playwright` | 3 | Free | npm + test script | Production-grade axe integration |
| Copilot multimodal (screenshots) | 4 | Token costs | None | Heuristic / UX review |
| `ux-mcp-server` | 4 | Token costs | `mcp.json` | Structured UX audit JSON |
| UI UX Suite | 2–3 | Free | npm / MCP | Experimental all-in-one design linter |
| Lighthouse CI | 5 | CI minutes | GitHub Actions | Performance regression blocking |
| GitHub Actions + Playwright | 5 | CI minutes | YAML config | Nightly full accessibility scans |

---

## Quick-Start Checklist

### Prerequisites
- [ ] Node.js 20+ installed
- [ ] VS Code 1.110+ with Copilot extensions

### Layer 0 & 1
- [ ] Lighthouse baseline run on 3–5 key pages
- [ ] axe DevTools scan completed
- [ ] 10-minute screen reader keyboard test done

### Layer 2 — Core Static Stack (Minimum)
- [ ] `eslint-plugin-jsx-a11y` installed and verified in editor
- [ ] axe Accessibility Linter (VS Code extension) installed and verified
- [ ] `jest-axe` test passing for at least one component

### Layer 2 — Enhanced Static Stack (Recommended)
- [ ] `@lapidist/design-lint` configured and running
- [ ] `stylelint-plugin-defensive-css` configured and running
- [ ] `apca-w3` contrast test passing for primary color pairs
- [ ] `@storybook/addon-a11y` active (if using Storybook)

### Layer 3 — Browser Automation
- [ ] `.vscode/mcp.json` created with Playwright MCP server
- [ ] Playwright MCP started and verified in Copilot Chat Agent Mode
- [ ] At least one real-browser AI test completed successfully
- [ ] *(Optional)* Axe MCP Server configured (if using paid tier)

### Layer 4 — Heuristic Review
- [ ] Screenshots captured for heuristic review
- [ ] Heuristic review prompt submitted to Copilot
- [ ] *(Optional)* `ux-mcp-server` configured for structured audits

### Layer 5 — CI Enforcement
- [ ] GitHub Actions workflow added for PR checks
- [ ] Nightly audit scheduled (for production-grade projects)

---

> **License:** This guide is provided as-is for educational purposes. Tool versions and availability change frequently; always verify against official documentation.
