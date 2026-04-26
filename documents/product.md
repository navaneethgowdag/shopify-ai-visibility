# Why AI Shopping Agents Skip Your Store — and What to Do About It

---

## Who It's For

A non-technical Shopify merchant with a well-designed store but zero visibility in AI-driven discovery. They need simple, actionable fixes they can apply from their phone — without touching code or hiring a developer.

---

## Problem

A new layer of commerce is emerging. AI agents inside platforms like ChatGPT, Perplexity, Google Shopping, and Amazon Rufus are making shopping decisions on behalf of users. These agents browse, compare, and recommend products automatically.

Most independent Shopify merchants are not being surfaced by these agents. They are effectively invisible in this new AI-driven shopping layer — and they don't know it.

---

## Reason

The issue is not product quality or pricing. It is machine readability.

AI agents rely on structured, unambiguous data to understand and recommend products. Most Shopify stores lack:

- Clear, descriptive product titles
- Sufficient product descriptions (what it is, who it's for, what it does)
- Trust indicators (reviews, return policy, shipping policy, brand story)
- Reliable inventory and availability signals
- Structured metadata and standardized attributes

To a human, the store may look polished and appealing. To an AI system, it looks like unstructured noise instead of usable signal. So the agent skips it — and moves to a store it can understand.

---

## Why It Matters Now

Agent-influenced purchases are growing faster than most merchants realize. Almost no merchants understand why they are being skipped.

Search engine optimization took a decade to become table stakes. Agent optimization will move faster. The merchants who become legible to AI agents in the next 12 to 18 months will have a structural advantage that compounds. Those who wait will face a channel they cannot buy their way into.

Unlike SEO, no one has explained this to non-technical merchants yet. The gap between what developers know and what store owners understand is total.

---

## What We Built

A conversational AI store audit tool that connects directly to a Shopify store through Telegram — the app merchants already use — and reads the store the way an AI agent would.

The merchant connects their store in under 2 minutes by sharing their store URL and a read-only API token. No code. No dashboard. No new app to install.

The bot also validates the API token instantly when submitted. If the token is invalid or expired, the merchant is told immediately and asked to try again.

Once connected, the merchant can run commands from a tap:

**Audit** — Fetches all Shopify products and pages in real time, analyzes AI readability, and outputs a score, strengths, issues, trust signals, and a prioritized action plan (Critical, High, Medium).

**Perception** — Simulates how AI shopping agents like Google Shopping or Amazon Rufus interpret the store using current data, highlights unanswered questions, and pinpoints changes needed to improve AI recommendations — bridging the gap between merchant perception and AI understanding.

**Fix** — Scans all products, detects weak or missing descriptions, and generates clear, benefit-focused rewrites ready to paste into Shopify. No copywriter needed.

**Assistant** — A conversational assistant that answers store-specific queries using only real product data. No hallucinations or invented details.

**Status** — Merchant can check which store is currently connected and confirm their setup is active.

**Chat** — Ask any store specific question and get answers using only real store data. No hallucinations.

---

## How It Works

```
Merchant sends message on Telegram
        ↓
n8n receives and routes the message
        ↓
Shopify Admin API fetches store data (read-only)
        ↓
Ollama (Llama3.1:8b) analyzes the data locally
        ↓
Gemini API as fallback if local model is unavailable
        ↓
Report sent back to merchant on Telegram
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Telegram Bot API | Merchant interface |
| n8n | Workflow automation connecting all services |
| Shopify Admin API | Read-only store data access |
| Ollama + Llama3.1:8b | Local LLM, merchant data stays private |
| Gemini 2.0 Flash | Fallback LLM for reliability |
| ngrok | Tunnel exposing local Ollama to n8n cloud |
| Google Sheets | Lightweight merchant credential storage |

---

## Tradeoffs and How We Resolved Them

We chose simple, understandable output over deep technical audits involving schema graphs and edge cases. We chose showing fixes over auto-fixing, sacrificing automation to gain simplicity and user trust. We chose Telegram over a web dashboard because merchants are already there and need zero onboarding.

---

## What We Chose Not to Build and Why

**SEO and marketing tools** — SEO optimizes for search engines and humans, not AI agents. Different problem.

**Custom store design and UI improvements** — AI agents do not evaluate visual design. They parse structured data. Irrelevant to the problem.

**Auto-fix and direct Shopify editing** — We chose not to write back to the store automatically. Trust is more important than automation at this stage.
