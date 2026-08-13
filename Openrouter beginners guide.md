# 🚀 The Complete Beginner's Guide to OpenRouter AI
## From Perplexity to OpenRouter: Cover All Your LLM Needs on a €10/Month Budget

---

## 📋 Table of Contents
1. [What Is OpenRouter? (And How It Differs from Perplexity)](#what-is-openrouter)
2. [Account Setup & Your €10 Budget Strategy](#account-setup)
3. [🏷️ How to Use Model Suffixes: The Complete Practical Guide](#suffixes)
4. [Need 1: Web Chat](#need-1-web-chat)
5. [Need 2: Deep Research in Web Chat](#need-2-deep-research)
6. [Need 3: Summarize & Explain Articles](#need-3-summarize)
7. [Need 4: Create Markdown & HTML Guides](#need-4-guides)
8. [Need 5: Coding with Claude Code & Kiro CLI](#need-5-coding)
9. [Need 6: YouTube Transcript Translation to Greek](#need-6-youtube)
10. [Token Efficiency: How to Stretch €10 Across a Month](#token-efficiency)
11. [Weekly Budget Tracker & Model Cheat Sheet](#cheat-sheet)

*Read top to bottom on first pass: sections 1–2 get you oriented and set up, section 3 is a suffix reference you can skim now and return to later, and sections 4–11 walk through each of the six needs plus budget management.*

---

<a id="what-is-openrouter"></a>

## 1. What Is OpenRouter? (And How It Differs from Perplexity)

### The Core Difference

| Feature | Perplexity | OpenRouter |
|---------|-----------|------------|
| **What it is** | AI search engine with chat | API gateway to 400+ LLMs |
| **Interface** | Web app + mobile app | Web chatroom + API + integrations |
| **Models** | Proprietary + limited selection | 400+ models from 70+ providers |
| **Pricing** | Subscription ($20/month) | Pay-as-you-go (credits) |
| **Web search** | Built-in, automatic | Optional plugin (extra cost) |
| **Best for** | Quick research answers | Power users who want model choice |

**OpenRouter is NOT a chat app like Perplexity.** It's a **unified API** that lets you access hundreds of AI models (Claude, GPT, Gemini, Llama, DeepSeek, etc.) through a single account and API key. You use it via:
- The **web chatroom** at `openrouter.ai/chat` (for casual chatting)
- **Third-party apps** (Claude Code, Kiro, TypingMind, etc.)
- **Direct API calls** (for custom scripts)

> 💡 **Key Insight:** Perplexity gives you one experience. OpenRouter gives you **choice** — you pick the best model for each task, which is how you save money.

---

<a id="account-setup"></a>

## 2. Account Setup & Your €10 Budget Strategy

### Step 1: Sign Up (Free)
1. Go to [openrouter.ai](https://openrouter.ai)
2. Sign up with Google, GitHub, or email
3. **No credit card required** to start

### Step 2: Understand the Free Tier
Before spending anything, you get:
- **25+ free models** at $0/token cost
- **20 requests per minute** limit
- **50 requests per day** limit (with $0 balance)
- **1,000 requests per day** (once you buy €10+ in credits — even if balance drops to zero later)

### Step 3: Add Your €10 Budget
1. Go to **Credits** in your dashboard
2. Add ~$11 USD (≈€10) via credit card or crypto
3. ⚠️ **Important:** OpenRouter charges a **5.5% fee** on credit purchases. So €10 buys you ~€9.45 in usable credits.

### Step 4: Create an API Key
1. Go to **Keys** → **Create Key**
2. Name it (e.g., "Personal-2026")
3. **Set a credit limit** on the key (e.g., €3) to prevent accidental overspending
4. Copy the key immediately — it starts with `sk-or-v1-...`

### Step 5: Set Up Spending Alerts
- Go to **Activity** tab to track usage in real-time
- Set up alerts if spending spikes unexpectedly
- Create **separate API keys** for different tools (Claude Code, Kiro, scripts) to track costs per project

---

<a id="suffixes"></a>

## 3. 🏷️ How to Use Model Suffixes: The Complete Practical Guide

Model suffixes are the **#1 money-saving feature** on OpenRouter, but beginners often miss them because they're not obvious in the interface. This chapter shows you **exactly** where to type them and what happens.

---

### What Are the Suffixes?

| Suffix | What It Does | When to Use |
|--------|-------------|-------------|
| `:free` | Routes to a zero-cost provider (only works on the ~30 models that have a free variant) | Always try this first |
| `:floor` | Routes to the cheapest paid provider | When free is rate-limited or unavailable |
| `:nitro` | Routes to the fastest provider | When speed matters more than cost |

> ⚠️ **There is no `:auto` suffix.** You can't append `:auto` to a model slug — OpenRouter will reject it. "Let OpenRouter pick the best model for me" is a *different feature* called `openrouter/auto`, which you use as its own model (not as a suffix on another model). See **Section 3.6** below for how it actually works.

> ⚠️ **Important:** These suffixes go **at the end of the model slug**, not the beginning. `meta-llama/llama-3.3-70b-instruct:free` ✅ — `:free/llama-3.3-70b` ❌

---

### 3.1 Using Suffixes in the OpenRouter Web Chatroom

The web chatroom is at `openrouter.ai/chat`. Here's exactly how to use suffixes there:

#### Step-by-Step (with screenshots description)

**Step 1: Open the model selector**
- Look at the top of the chat window
- You'll see the current model name (e.g., "Meta: Llama 3.3 70B Instruct")
- **Click on it** — this opens the model dropdown

**Step 2: Type the suffix directly in the model search box**
- The model selector has a search box at the top
- You can type the **full model slug with suffix** here
- Example: Type `meta-llama/llama-3.3-70b-instruct:free`
- Press Enter or click the model when it appears

**Step 3: Verify it's working**
- The model name at the top should now show your selection
- If you selected `:free`, you'll see "(Free)" or similar indicator
- Send a test message — if it responds, the suffix worked

**Alternative: Use the model slug directly in the URL**
- You can navigate directly to: `openrouter.ai/chat?model=meta-llama/llama-3.3-70b-instruct:free`
- This pre-selects the model with the suffix
- Bookmark these URLs for quick access to your favorite free models

#### What You'll See

**Without suffix (default behavior):**
```
Model: Meta: Llama 3.3 70B Instruct
→ OpenRouter picks a provider based on availability
→ Could be any provider, any price tier
```

**With `:free` suffix:**
```
Model: Meta: Llama 3.3 70B Instruct (Free)
→ OpenRouter forces a free provider
→ Cost: €0.00 per request
→ Limit: 20 requests/minute, 50-1000/day depending on balance
```

**With `:floor` suffix:**
```
Model: Meta: Llama 3.3 70B Instruct (Floor)
→ OpenRouter picks the cheapest paid provider
→ Cost: Minimum possible for this model
→ No free-tier rate limits
```

---

### 3.2 Using Suffixes in API Calls (Python, JavaScript, curl)

When calling the OpenRouter API directly, the suffix is part of the `model` string in your JSON payload.

#### Python Example

```python
import requests

API_KEY = "sk-or-v1-YOUR-KEY-HERE"

# WITHOUT suffix — OpenRouter picks any provider
response = requests.post(
    "https://openrouter.ai/api/v1/chat/completions",
    headers={"Authorization": f"Bearer {API_KEY}"},
    json={
        "model": "meta-llama/llama-3.3-70b-instruct",  # No suffix
        "messages": [{"role": "user", "content": "Hello!"}]
    }
)

# WITH :free suffix — forces zero-cost provider
response = requests.post(
    "https://openrouter.ai/api/v1/chat/completions",
    headers={"Authorization": f"Bearer {API_KEY}"},
    json={
        "model": "meta-llama/llama-3.3-70b-instruct:free",  # ← Suffix here
        "messages": [{"role": "user", "content": "Hello!"}]
    }
)

# WITH :floor suffix — cheapest paid provider
response = requests.post(
    "https://openrouter.ai/api/v1/chat/completions",
    headers={"Authorization": f"Bearer {API_KEY}"},
    json={
        "model": "meta-llama/llama-3.3-70b-instruct:floor",  # ← Suffix here
        "messages": [{"role": "user", "content": "Hello!"}]
    }
)
```

#### JavaScript/Node.js Example

```javascript
const response = await fetch("https://openrouter.ai/api/v1/chat/completions", {
  method: "POST",
  headers: {
    "Authorization": "Bearer sk-or-v1-YOUR-KEY-HERE",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    model: "anthropic/claude-sonnet-4.5:floor",  // ← Suffix in the model string
    messages: [{ role: "user", content: "Hello!" }]
  })
});
```

#### curl Example

```bash
curl https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer sk-or-v1-YOUR-KEY-HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek/deepseek-v3.2:floor",
    "messages": [{"role": "user", "content": "Explain quantum computing"}]
  }'
```

---

### 3.3 Using Suffixes in Claude Code

Claude Code is a CLI tool. You switch models using the `/model` command.

#### Interactive Mode

```bash
$ claude
# Claude Code starts...

# Switch to a free model
> /model poolside/laguna-m.1:free
✓ Model switched to poolside/laguna-m.1:free

# Switch to cheapest provider for a premium model
> /model anthropic/claude-sonnet-4.5:floor
✓ Model switched to anthropic/claude-sonnet-4.5:floor

# Switch to fastest provider
> /model meta-llama/llama-3.3-70b-instruct:nitro
✓ Model switched to meta-llama/llama-3.3-70b-instruct:nitro
```

#### Non-Interactive Mode (Scripts)

```bash
# One-off command with a specific model
claude -p "Explain Python decorators" --model poolside/laguna-m.1:free

# Or set environment variable for the session
export CLAUDE_CODE_MODEL="meta-llama/llama-3.3-70b-instruct:free"
claude -p "Write a function to sort a list"
```

#### In `.claude/settings.local.json`

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://openrouter.ai/api",
    "ANTHROPIC_AUTH_TOKEN": "sk-or-v1-YOUR-KEY-HERE",
    "ANTHROPIC_API_KEY": ""
  },
  "model": "poolside/laguna-m.1:free"
}
```

> ⚠️ **Get this the right way round.** Your OpenRouter key goes in `ANTHROPIC_AUTH_TOKEN`, and `ANTHROPIC_API_KEY` must be left as an empty string. Swapping these is the most common reason Claude Code silently falls back to an Anthropic login prompt instead of routing through OpenRouter.

> 💡 **Pro Tip:** Create a shell alias for quick model switching:
> ```bash
> alias claude-free='claude --model poolside/laguna-m.1:free'
> alias claude-cheap='claude --model deepseek/deepseek-v3.2:floor'
> alias claude-fast='claude --model meta-llama/llama-3.3-70b-instruct:nitro'
> ```

---

### 3.4 Using Suffixes in Kiro CLI

> 📝 **Important correction:** Kiro (AWS's agentic IDE/CLI) is a real, actively developed product — but it isn't primarily an "OpenRouter front-end." Kiro ships with its own built-in model access (Anthropic Claude, OpenAI GPT-5.6, DeepSeek, Qwen, GLM, MiniMax, and more) billed through your AWS/Kiro credits, configured via `kiro-cli settings` commands rather than a `models.yaml` file or a "Settings → AI Providers" screen. There is **no confirmed native OpenRouter provider slot** in Kiro at the time of writing — treat any specific menu path as unverified until you check Kiro's own docs at [kiro.dev/docs](https://kiro.dev/docs/).

What *is* confirmed: OpenRouter publishes an MCP server that gives MCP-compatible tools — including Kiro — access to its 400+ models. Kiro supports MCP servers via a JSON config file (`.kiro/settings/mcp.json` at the workspace level, or a global equivalent), in this general shape:

```json
{
  "mcpServers": {
    "openrouter": {
      "url": "https://openrouter.ai/api/v1/mcp",
      "headers": {
        "Authorization": "Bearer sk-or-v1-YOUR-KEY-HERE"
      }
    }
  }
}
```

This routes *tool calls* through OpenRouter rather than swapping Kiro's own chat model, so it's most useful for reaching a specific OpenRouter-only model or capability Kiro doesn't already offer natively — not a like-for-like replacement for typing a `:free`/`:floor`/`:nitro` suffix into Kiro's own model picker. If your goal is just "use free or cheap models for coding," Kiro's own model picker (with its "Auto" routing option) may already cover that without OpenRouter in the loop at all. Confirm exact syntax against [Kiro's MCP configuration docs](https://kiro.dev/docs/mcp/configuration/) before relying on this.

---

### 3.5 Using Suffixes in Other Third-Party Tools

Most tools that support OpenRouter accept the suffix in the model field:

#### TypingMind
- **Settings → AI Provider → OpenRouter**
- **Model** field: Enter `meta-llama/llama-3.3-70b-instruct:free`
- **Save** — all new chats use this model

#### Open WebUI
- **Admin Panel → Settings → Connections**
- Add OpenRouter connection
- In **Model** dropdown, type the slug with suffix manually if not in list
- Or use the **API Model ID** field: `anthropic/claude-haiku-4.5:floor`

#### Continue.dev (VS Code Extension)
- Open `~/.continue/config.json`
- Find the OpenRouter model config:
```json
{
  "models": [
    {
      "title": "OpenRouter Free",
      "provider": "openrouter",
      "model": "meta-llama/llama-3.3-70b-instruct:free",
      "apiKey": "sk-or-v1-YOUR-KEY-HERE"
    }
  ]
}
```

#### Any Tool Using OpenAI-Compatible API
Any tool that lets you set a "Model" or "Model ID" string works the same way — just append the suffix to the model name.

---

### 3.6 The Special `openrouter/auto` Model + Cost Tiers

This is a **different concept** from suffixes but achieves the same goal: saving money.

#### What Is `openrouter/auto`?
Instead of picking a specific model, you tell OpenRouter: *"Pick the best model for this task, but stay within my budget."*

#### How to Use It

**In API calls:**
```json
{
  "model": "openrouter/auto",
  "plugins": [{
    "id": "auto-router",
    "cost_tier": "low"
  }],
  "messages": [{"role": "user", "content": "Hello!"}]
}
```

**Cost tier options:**
| Tier | Approximate Cost | Use Case |
|------|-----------------|----------|
| `low` | Free to $0.50/M tokens | Daily tasks, simple Q&A |
| `medium` | $0.50-2.00/M tokens | Standard coding, writing |
| `high` | $2.00-5.00/M tokens | Complex reasoning |
| `xhigh` | $5.00-10.00/M tokens | Near-frontier quality |
| `max` | $10.00+/M tokens | Best available, no budget limit |

**In the web chatroom:**
- Select "OpenRouter: Auto" from the model dropdown
- It will automatically pick a model based on your request complexity
- You can't set `cost_tier` in the chatroom — it defaults to balanced

> 💡 **Pro Tip:** Use `openrouter/auto` with `cost_tier: low` for unpredictable workloads. It's like having a smart assistant that never overspends.

---

### 3.7 Common Mistakes & How to Fix Them

#### Mistake 1: Typing the Suffix in the Wrong Place
```
❌ Wrong:  :free/meta-llama/llama-3.3-70b-instruct
❌ Wrong:  meta-llama/llama-3.3-70b-instruct/free
✅ Right:  meta-llama/llama-3.3-70b-instruct:free
```

#### Mistake 2: Using `:free` on a Model That Has No Free Provider
```
❌ Wrong:  anthropic/claude-opus-4.5:free
→ Error: "No free provider available for this model"
✅ Right:  anthropic/claude-opus-4.5:floor
→ Routes to cheapest paid provider
```

**How to check if a model has a free version:**
1. Go to [openrouter.ai/models](https://openrouter.ai/models)
2. Filter by "Free" — only these models work with `:free`
3. Or try it — if you get an error, switch to `:floor`

#### Mistake 3: Forgetting the Colon
```
❌ Wrong:  llama-3.3-70b-instruct-free
❌ Wrong:  llama-3.3-70b-instruct_free
✅ Right:  llama-3.3-70b-instruct:free
```

#### Mistake 4: Using Suffixes in the API Key
```
❌ Wrong:  sk-or-v1-...:free
✅ Right:  sk-or-v1-... (key stays clean, suffix goes on model)
```

#### Mistake 5: Assuming `:floor` Is Always Cheaper Than `:free`
```
:free  = €0.00 (but rate-limited)
:floor = cheapest PAID option (costs money, but no rate limits)

If :free works, it's always cheaper than :floor.
Only switch to :floor when :free hits rate limits.
```

---

### 3.8 Quick Reference: Where to Type the Suffix

| Tool/Platform | Where to Type | Example |
|---------------|--------------|---------|
| **Web Chatroom** | Model search box at top | `meta-llama/llama-3.3-70b-instruct:free` |
| **Python script** | `"model"` field in JSON | `"model": "deepseek/deepseek-v3.2:floor"` |
| **JavaScript** | `model` property in body | `model: "anthropic/claude-sonnet-4.5:floor"` |
| **curl** | `"model"` in `-d` payload | `"model": "openai/gpt-oss-120b:free"` |
| **Claude Code** | `/model` command or `--model` flag | `/model poolside/laguna-m.1:free` |
| **Kiro** | Not applicable — reach OpenRouter models via its MCP server instead (see §3.4) | n/a |
| **TypingMind** | Settings → Model field | `google/gemma-4-31b-it:free` |
| **Open WebUI** | API Model ID field | `nvidia/nemotron-3-ultra-550b-a55b:free` |
| **Continue.dev** | `model` in config.json | `"model": "cohere/north-mini-code:free"` |
| **Any OpenAI-compatible tool** | Model name field | Always append `:free` or `:floor` |

---

### 3.9 Testing Your Suffix Setup

Run this quick test to verify everything works:

**Test 1: Free Model in Chatroom**
1. Go to `openrouter.ai/chat`
2. Click model selector
3. Type: `meta-llama/llama-3.3-70b-instruct:free`
4. Send: "Say 'free model works'"
5. ✅ If you get a response, free tier is active

**Test 2: Floor Model via API**
```bash
curl https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer sk-or-v1-YOUR-KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/llama-3.3-70b-instruct:floor",
    "messages": [{"role": "user", "content": "Test"}]
  }'
```
✅ If you get JSON back with a response, `:floor` works

**Test 3: Check Cost in Activity Tab**
1. Send 5 messages with `:free`
2. Go to **Activity** tab
3. Cost should show **$0.00**
4. Send 1 message with `:floor`
5. Cost should show a small amount (e.g., $0.0001)

---

### 3.10 Advanced: Combining Suffixes with Other Features

#### Suffix + Web Search Plugin
```json
{
  "model": "meta-llama/llama-3.3-70b-instruct:free",
  "plugins": [{"id": "web"}],
  "messages": [{"role": "user", "content": "Latest news about AI"}]
}
```
→ Free model + free web search = **€0 total**

#### Suffix + Max Price Cap
```json
{
  "model": "anthropic/claude-sonnet-4.5:floor",
  "provider": {
    "max_price": {
      "prompt": 1.0,
      "completion": 2.0
    }
  },
  "messages": [{"role": "user", "content": "Complex task"}]
}
```
→ Cheapest provider, but **never** more than $1/$2 per million tokens

#### Suffix + Context Limits
```json
{
  "model": "nvidia/nemotron-3-ultra-550b-a55b:free",
  "max_tokens": 500,
  "messages": [{"role": "user", "content": "Summarize this 50,000-word article..."}]
}
```
→ Free 1M-context model, but **capped at 500 output tokens** to prevent runaway costs

---

> 🎯 **Remember:** The suffix is just text appended to the model name. There are no special buttons, no hidden menus, no configuration files to edit (unless you want to). Just type `:free`, `:floor`, or `:nitro` at the end of any model slug, and OpenRouter handles the rest.



<a id="need-1-web-chat"></a>

## 4. Need 1: Web Chat

### Option A: OpenRouter Chatroom (Browser)
- Go to `openrouter.ai/chat`
- Select any model from the dropdown
- **Free models** work here at zero cost
- **Web search is available for free** in the chatroom (as of January 2025)

### Option B: Better Chat Interfaces (Recommended)
Since OpenRouter's native chatroom is basic, use these free front-ends:

| Tool | Cost | Best For |
|------|------|----------|
| **OpenRouter Chatroom** | Free | Quick tests |
| **TypingMind** (one-time purchase) | ~$15 | Best UI, local storage |
| **Open WebUI** (self-hosted) | Free | Privacy, full control |
| **Lobechat** | Free | Modern UI, mobile-friendly |

### Recommended Free Models for General Chat
```
meta-llama/llama-3.3-70b-instruct:free    # Multilingual, stable
openai/gpt-oss-20b:free                    # Fast, lightweight
nvidia/nemotron-3-super-120b-a12b:free     # Strong reasoning
```

### Pro Tip: Use `:floor` for Cheapest Paid Models
If free models are rate-limited, append `:floor` to any model to get the cheapest provider:
```
meta-llama/llama-3.3-70b-instruct:floor
```

---

<a id="need-2-deep-research"></a>

## 5. Need 2: Deep Research in Web Chat

### How OpenRouter Handles Research

OpenRouter offers **two** research capabilities:

#### A) Web Search Plugin (API/Chat)
Add `"plugins": [{"id": "web"}]` to your request. Available engines:

| Engine | Cost per Request | Best For |
|--------|-----------------|----------|
| **Exa (Deep)** | $0.012 | Thorough research, 10 results |
| **Exa (Deep Reasoning)** | $0.015 | Complex multi-step research |
| **Perplexity** | $0.005 | Fast, reliable |
| **Parallel (Basic)** | $0.005 | Balanced speed/depth |
| **Native** | Provider pricing | If using OpenAI/Anthropic models |

**In the OpenRouter chatroom, web search is FREE.** Just toggle it on.

#### B) Deep Research Tool (Since March 2025)
OpenRouter has a built-in deep research feature that:
- Performs multi-step web searches
- Synthesizes information with **full citations**
- Returns structured research reports

**To use it:**
1. In API requests, use the deep research plugin configuration
2. In chatroom, enable research mode (if available)

### Cost-Effective Research Strategy
```
1. Start with FREE web search in OpenRouter chatroom
2. For API/automated research: Use Exa Deep ($0.012/request)
3. For complex analysis: Route to a cheap but capable model
   Example: qwen/qwen3-30b-a3b-instruct-2507 ($0.05/M input)
```

### Budget Math for Research
- 50 deep research queries/month = 50 × $0.012 = **$0.60**
- Plus model tokens (~$0.10-0.50 per research session)
- **Total: ~€1-3/month** for heavy research use

---

<a id="need-3-summarize"></a>

## 6. Need 3: Summarize & Explain Articles

### The Workflow
1. **Paste the article text** directly into chat (most efficient)
2. **Or use web search** to fetch the article, then summarize

### Best Free Models for Summarization
```
nvidia/nemotron-3-ultra-550b-a55b:free     # 1M context — can handle entire books
google/gemma-4-31b-it:free                 # Multilingual, 140+ languages
meta-llama/llama-3.3-70b-instruct:free     # Reliable, stable
```

### Token-Efficient Prompting for Summaries
Instead of:
```
"Please read this article and summarize it for me..."
```

Use:
```
"Summarize in 3 bullet points. Key insights only. Article:

[PASTE TEXT]"
```

**Why this saves money:**
- Shorter system prompt = fewer input tokens
- Explicit output format prevents rambling
- Every token costs money — brevity is your friend

### For Long Articles (10,000+ words)
Use models with **1M context windows**:
```
nvidia/nemotron-3-ultra-550b-a55b:free     # 1M context, FREE
```
This lets you paste entire PDFs or long articles without chunking.

---

<a id="need-4-guides"></a>

## 7. Need 4: Create Markdown & HTML Guides

### Best Models for Structured Content

| Task | Free Model | Cheap Paid Alternative |
|------|-----------|----------------------|
| Markdown guides | `poolside/laguna-m.1:free` | `deepseek/deepseek-v3.2` ($0.27/M) |
| HTML/CSS | `cohere/north-mini-code:free` | `mistralai/mistral-nemo` ($0.02/M) |
| Technical docs | `openai/gpt-oss-120b:free` | `openai/gpt-oss-120b` ($0.04/M) |

### Prompt Engineering for Guides
**Bad (wastes tokens):**
```
"Write me a guide about Python programming. Make it comprehensive..."
```

**Good (saves tokens & money):**
```
"Create a markdown guide: Python Decorators. Include: 1) Syntax, 2) 3 examples, 3) Common pitfalls. Max 500 words."
```

### Pro Tips for Guide Creation
1. **Use `:floor` suffix** — always get the cheapest provider
2. **Set `max_tokens`** — prevents overly long outputs
3. **Use structured output** — request JSON/markdown directly
4. **Iterate in free models first** — only switch to paid for final polish

---

<a id="need-5-coding"></a>

## 8. Need 5: Coding with Claude Code & Kiro CLI

### Part A: Claude Code + OpenRouter Setup

Claude Code is Anthropic's CLI coding assistant. Normally it requires an Anthropic subscription, but you can route it through OpenRouter.

#### Method 1: Shell Environment Variables (Recommended)

Add to your shell profile (`~/.zshrc`, `~/.bashrc`, or `~/.config/fish/config.fish`):

```bash
export OPENROUTER_API_KEY="sk-or-v1-YOUR-KEY-HERE"
export ANTHROPIC_BASE_URL="https://openrouter.ai/api"
export ANTHROPIC_AUTH_TOKEN="$OPENROUTER_API_KEY"
export ANTHROPIC_API_KEY=""  # MUST be explicitly empty
export CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1
```

**Important:** Run `source ~/.zshrc` (or restart terminal) after saving. Your OpenRouter key goes in `ANTHROPIC_AUTH_TOKEN` — `ANTHROPIC_API_KEY` is the one that must be empty. If you've logged into Claude Code with a real Anthropic account before, also run `/logout` once inside Claude Code so the cached login doesn't override these variables.

#### Method 2: Project-Level Config (Safer)

Create `.claude/settings.local.json` in your project root:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://openrouter.ai/api",
    "ANTHROPIC_AUTH_TOKEN": "sk-or-v1-YOUR-KEY-HERE",
    "ANTHROPIC_API_KEY": "",
    "CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY": "1"
  }
}
```

#### Switching Models in Claude Code
Once connected, switch models mid-session:
```bash
claude
# Then type:
/model poolside/laguna-m.1:free
```

#### Best Free Models for Coding in Claude Code
```
poolside/laguna-m.1:free        # Best for complex software engineering
cohere/north-mini-code:free     # Good for general coding tasks
openai/gpt-oss-120b:free        # Strong reasoning, Apache 2.0 license
poolside/laguna-xs-2.1:free     # Fast, compact coding agent
```

#### Cheap Paid Fallbacks (When Free Hits Limits)
```
deepseek/deepseek-v3.2          # $0.27/M input — near-frontier quality
openai/gpt-oss-120b             # $0.04/M input — open-weight, strong
inclusionai/ling-2.6-flash      # $0.01/M input — ultra-cheap
```

---

### Part B: Kiro CLI + OpenRouter Setup

**What is Kiro?** Kiro is AWS's agentic IDE, available as an editor, a CLI, and a web surface, built around "spec-driven" development (it turns a prompt into requirements, a plan, and tasks before writing code).

**A note before you set anything up:** Kiro is not primarily an OpenRouter front-end. It comes with its own built-in model lineup (Anthropic Claude, OpenAI GPT-5.6, DeepSeek, Qwen, GLM, MiniMax, and an "Auto" mode that picks a model for you), billed through your Kiro/AWS credits — not through OpenRouter. There's no confirmed native "add OpenRouter as a provider" screen in Kiro at the time of writing. If Kiro's own model lineup already covers what you need, you likely don't need OpenRouter here at all.

#### Reaching OpenRouter Models from Kiro (via MCP)

Where OpenRouter *does* fit in is through its MCP server, which several MCP-compatible tools (including Kiro) can connect to for access to models Kiro doesn't offer natively. Kiro reads MCP servers from a JSON config file (workspace-level `.kiro/settings/mcp.json`, or a global equivalent):

```json
{
  "mcpServers": {
    "openrouter": {
      "url": "https://openrouter.ai/api/v1/mcp",
      "headers": {
        "Authorization": "Bearer sk-or-v1-YOUR-KEY-HERE"
      }
    }
  }
}
```

This gives Kiro's agent access to OpenRouter's model catalog as a *tool* it can call, rather than swapping out Kiro's own chat model wholesale. Treat the exact file path and JSON shape above as a starting point, and confirm the current syntax against [Kiro's MCP configuration docs](https://kiro.dev/docs/mcp/configuration/) before relying on it — MCP config formats are exactly the kind of detail that changes between releases.

#### Token Efficiency for Coding Agents

Coding agents (Claude Code, Kiro) can burn through tokens fast. Here's how to control costs:

| Strategy | How | Savings |
|----------|-----|---------|
| **Use free models for drafting** (Claude Code) | `laguna-m.1:free` for initial code | 100% |
| **Use cheap models for simple edits** (Claude Code) | `ling-2.6-flash` at $0.01/M | ~95% vs Claude |
| **Reserve expensive models for debugging** | Only use Claude/GPT for complex bugs | 80% |
| **Limit context window** | Cap context sent per turn instead of the full window | 50%+ |
| **Enable context pruning** | Auto-trim old tool results | 30%+ |
| **Use `:floor` routing** (Claude Code) | Cheapest provider automatically | 20-50% |

> Kiro's own credit/cost controls are configured through `kiro-cli settings` and its model picker, not through OpenRouter suffixes — see [Kiro's settings reference](https://kiro.dev/docs/reference/settings/) for the native equivalents.

---

<a id="need-6-youtube"></a>

## 9. Need 6: YouTube Transcript Translation to Greek

### The Workflow

OpenRouter doesn't directly access YouTube, so you need a 2-step process:

#### Step 1: Get the Transcript (Free Tools)
Use one of these to extract transcripts:
- **YouTube Transcript** (browser extension)
- **downsub.com** (paste YouTube URL)
- **yt-dlp** (CLI tool): `yt-dlp --write-auto-sub --skip-download URL`
- **Python script** using `youtube-transcript-api`

#### Step 2: Translate with OpenRouter

**Best free models for Greek translation:**
```
google/gemma-4-31b-it:free          # 140+ languages, 262K context
meta-llama/llama-3.3-70b-instruct:free  # Supports Greek
nvidia/nemotron-3-ultra-550b-a55b:free   # 1M context for long transcripts
```

**Prompt template:**
```
Translate the following YouTube transcript from [SOURCE_LANGUAGE] to Greek.
Preserve timestamps. Maintain natural, conversational Greek tone.
Format: [TIMESTAMP] GREEK_TEXT

Transcript:
[PASTE HERE]
```

#### For Analysis (Not Just Translation)
```
Translate to Greek, then analyze: main arguments, tone, target audience, and 3 key takeaways.
```

### Cost Estimate
- A 30-minute video transcript ≈ 10,000 tokens
- With free models: **€0**
- With cheap paid models (`ling-2.6-flash` at $0.01/M): **~€0.0001**
- Even with premium models: **~€0.02-0.05 per video**

**You could translate 200+ videos for under €1.**

---

<a id="token-efficiency"></a>

## 10. Token Efficiency: How to Stretch €10 Across a Month

### The Golden Rules

#### Rule 1: Start Free, Escalate Only When Needed
```
Free models first → Cheap paid → Premium only for final quality check
```

#### Rule 2: Match Model to Task Complexity

| Task Type | Use This | Avoid This |
|-----------|----------|------------|
| Simple Q&A | Free Llama/Gemma | Claude Opus |
| Code generation | Free Laguna/Cohere | GPT-5 |
| Summarization | Free Nemotron | Claude Sonnet |
| Complex debugging | Cheap DeepSeek | Only if free fails |
| Critical final review | Claude/GPT | Don't use free for this |

#### Rule 3: Use `:floor` for Automatic Savings
Append `:floor` to any model slug to route to the cheapest provider automatically — full syntax and examples are in **Section 3**.

#### Rule 4: Use `:nitro` When Speed Matters
Append `:nitro` when latency matters more than cost — see **Section 3** for details.

#### Rule 5: Use `openrouter/auto` with Cost Tiers
Let OpenRouter pick a model for you within a budget band (`low`/`medium`/`high`/`xhigh`/`max`) — this is a separate model, not a suffix; see **Section 3.6** for the full setup.

#### Rule 6: Compress Your Prompts

**Before (wasteful):**
```
"Hello, I hope you're doing well. I have a question about Python. I've been learning it for a while and I'm struggling with decorators. Could you please explain..."
```

**After (efficient):**
```
"Explain Python decorators with 2 examples."
```

**Savings: ~70% fewer input tokens**

#### Rule 7: Set Hard Budget Caps
```json
{
  "provider": {
    "max_price": {
      "prompt": 1.0,      // $1 per million input tokens max
      "completion": 2.0   // $2 per million output tokens max
    }
  }
}
```
If no provider qualifies, the request fails instead of overspending.

#### Rule 8: Use Context Pruning (For Long Sessions)
Long agent sessions accumulate old tool output and chat history in the context window, and you pay to re-send all of it on every turn. The general fix is the same across tools: periodically drop or summarize older turns instead of keeping the full history. The JSON shape below is illustrative, not a verified universal parameter — check your specific tool's docs (Claude Code, Kiro, etc.) for its actual context-management setting:
```json
{
  "contextPruning": {
    "mode": "cache-ttl",
    "ttl": "5m",
    "keepLastAssistants": 3
  }
}
```

#### Rule 9: Separate API Keys Per Project
Create different keys for:
- Claude Code
- Kiro
- Personal scripts
- Research

Each with its own spending limit.

#### Rule 10: Monitor Weekly
Check your OpenRouter **Activity** tab every Sunday. If you're at 50% budget by week 2, switch more tasks to free models.

---

<a id="cheat-sheet"></a>

## 11. Weekly Budget Tracker & Model Cheat Sheet

### Your €10 Monthly Budget Breakdown

| Week | Budget | Primary Use | Safety Net |
|------|--------|-------------|------------|
| Week 1 | €2.50 | Experiment with paid models | Free models if overspending |
| Week 2 | €2.50 | Establish workflows | Track what's expensive |
| Week 3 | €2.50 | Optimize, use free more | Cut paid model usage |
| Week 4 | €2.50 | Reserve for critical tasks | Pure free model week if needed |

### Model Cheat Sheet (Bookmark This)

#### 🆓 FREE TIER (€0)
```
General Chat:
  meta-llama/llama-3.3-70b-instruct:free
  openai/gpt-oss-20b:free

Coding:
  poolside/laguna-m.1:free
  cohere/north-mini-code:free
  poolside/laguna-xs-2.1:free

Long Context (1M tokens):
  nvidia/nemotron-3-ultra-550b-a55b:free

Multilingual/Greek:
  google/gemma-4-31b-it:free

Reasoning/Research:
  nvidia/nemotron-3-super-120b-a12b:free
  qwen/qwen3-next-80b-a3b-instruct:free

Auto-select:
  openrouter/free
```

#### 💰 CHEAP PAID (€0.01-0.50 per heavy session)
```
Ultra-cheap general:
  inclusionai/ling-2.6-flash          $0.01/M input
  mistralai/mistral-nemo              $0.02/M input

Strong open-weight coding:
  openai/gpt-oss-120b                 $0.04/M input
  qwen/qwen3-30b-a3b-instruct-2507  $0.05/M input

Near-frontier reasoning:
  deepseek/deepseek-v3.2              $0.27/M input
```

#### 💎 PREMIUM (Reserve for critical tasks)
```
  anthropic/claude-sonnet-4.5         ~$3/M input
  anthropic/claude-haiku-4.5          ~$1/M input
  openai/gpt-5.1                      ~$1.25/M input
```

### Cost-Saving Quick Reference

| Technique | Code/Config | Expected Savings |
|-----------|-------------|------------------|
| Free models | `:free` suffix | 100% |
| Cheapest provider | `:floor` suffix | 20-50% |
| Fastest provider | `:nitro` suffix | Use sparingly |
| Auto budget routing | `openrouter/auto` + `cost_tier: low` | 30-60% |
| Hard price cap | `max_price` in provider config | Prevents overspend |
| Short prompts | Remove fluff | 50-70% input reduction |
| Context limits | `maxTokens: 1000` | Prevents long outputs |
| Context pruning | Tool-specific — check your agent's docs | 30%+ on long sessions |

---

## 🎯 Your First Week Action Plan

### Day 1: Setup
- [ ] Sign up at openrouter.ai
- [ ] Add €10 credits
- [ ] Create 3 API keys (Claude Code, Kiro, General)
- [ ] Set €3 limit on each key

### Day 2: Web Chat & Research
- [ ] Test `openrouter.ai/chat` with free models
- [ ] Try web search in chatroom (free)
- [ ] Compare 3 free models on same question

### Day 3: Article Summarization
- [ ] Paste a long article into chat
- [ ] Test `nemotron-3-ultra:free` (1M context)
- [ ] Practice concise prompting

### Day 4: Guide Creation
- [ ] Create a markdown guide using `laguna-m.1:free`
- [ ] Test HTML generation
- [ ] Measure token usage in Activity tab

### Day 5: Claude Code Setup
- [ ] Install Claude Code
- [ ] Configure with OpenRouter (Method 1 or 2)
- [ ] Run first coding task with free model
- [ ] Test `/model` command to switch models

### Day 6: Kiro Setup
- [ ] Configure Kiro with OpenRouter API
- [ ] Test coding with free model
- [ ] Compare Kiro vs Claude Code experience

### Day 7: YouTube Translation
- [ ] Extract a transcript
- [ ] Translate to Greek with `gemma-4-31b:free`
- [ ] Analyze cost in Activity tab
- [ ] Review weekly spending

---

## ⚠️ Common Beginner Mistakes to Avoid

1. **Forgetting the 5.5% purchase fee** — €10 only gives ~€9.45 in credits
2. **Using premium models for simple tasks** — Don't use Claude for "hello"
3. **Not setting API key limits** — One runaway script can drain your budget
4. **Ignoring free models** — Many are genuinely capable (Laguna, Nemotron)
5. **Long, fluffy prompts** — Every word costs money. Be concise.
6. **Not using `:floor`** — Same model, different provider = 2-10x price difference
7. **Forgetting context accumulates** — Long chats get expensive. Start fresh often.
8. **Not monitoring Activity tab** — Check weekly, not monthly
9. **Using web search plugins unnecessarily** — Chatroom search is free; API search costs extra
10. **Not having fallback models** — If a free model hits rate limits, have a cheap paid backup ready

---

## 📚 Additional Resources

- **OpenRouter Models:** [openrouter.ai/models](https://openrouter.ai/models) (filter by "Free")
- **Pricing Calculator:** Check per-model costs before using
- **Activity Dashboard:** Track spending in real-time
- **Discord Community:** Get help at OpenRouter's Discord
- **Claude Code Docs:** [openrouter.ai/docs/cookbook/coding-agents/claude-code-integration](https://openrouter.ai/docs/cookbook/coding-agents/claude-code-integration)
- **Web Search Docs:** [openrouter.ai/docs/guides/features/plugins/web-search](https://openrouter.ai/docs/guides/features/plugins/web-search)
- **Cost Optimization Guide:** [openrouter.ai/blog/tutorials/how-to-get-the-lowest-cost-llm-inference-on-openrouter](https://openrouter.ai/blog/tutorials/how-to-get-the-lowest-cost-llm-inference-on-openrouter)

---

## ✅ Final Checklist: Are You Ready?

- [ ] OpenRouter account created
- [ ] €10 credits added (≈€9.45 after fee)
- [ ] API keys created with spending limits
- [ ] Free models tested and bookmarked
- [ ] Claude Code configured for OpenRouter
- [ ] Kiro configured for OpenRouter
- [ ] YouTube transcript workflow tested
- [ ] Weekly budget check scheduled (Sundays)
- [ ] `:floor` and `:free` suffixes memorized
- [ ] First week's action plan ready

---

*Good luck with your OpenRouter journey! With smart model selection and token discipline, €10/month is more than enough to cover all six of your needs — and then some.*

*Last updated: August 2026*
