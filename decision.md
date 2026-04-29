# Perceptra — Architecture and Design Decisions

This document explains every major decision we made while building the AI Store Readability Auditor, including what we considered, what we chose, and why.



## 1. Initial Plan vs Final Approach

### What We Planned First

We initially planned to build everything from scratch using code.

The plan was:
- FastAPI or Node.js for the backend
- Custom Shopify API integration
- Manual audit logic with hardcoded rules
- LLM API calls handled in code
- Custom frontend or web dashboard

### Why We Changed

After planning, we realized this would take too much time for a hackathon timeline. Handling APIs, error flows, fallback logic, and a frontend simultaneously was too much scope.

We also ran into API token limits with cloud LLMs early on.

### Final Decision

We switched to n8n as our automation layer and Ollama as our local LLM. This let us focus on product logic instead of infrastructure.



## 2. Why n8n Instead of Custom Backend

### Options Considered
- FastAPI (Python)
- Node.js + Express
- n8n (no-code workflow automation)

### Decision
We chose n8n.

### Reasons
- Connects to Shopify, Telegram, Google Sheets, and LLMs without writing boilerplate code
- Visual workflow makes it easy to debug and trace errors
- Built-in error handling and fallback routing
- Faster to build and iterate
- Non-technical team members can understand the flow

### Tradeoff
- Less control compared to writing custom code
- Not suitable for large-scale production without significant changes
- Some advanced logic requires workarounds in n8n


## 3. Why Telegram Instead of a Web Dashboard

### Options Considered
- Custom React web dashboard
- Telegram Bot
- WhatsApp Business API

### Decision
We chose Telegram.

### Reasons
- Merchants already use Telegram daily — zero onboarding friction
- No new app to install, no login, no new tab to open
- Telegram Bot API is free and well documented
- Reply keyboard buttons make it one-tap for merchants
- Works on any phone and any OS

### Tradeoff
- Cannot add charts, graphs, or rich visual UI
- Less control over design and layout
- Telegram message length limit of 4096 characters requires output trimming

### How We Argued It
We chose Telegram because the best interface is the one merchants already have open. A non-technical merchant like Priya does not want another dashboard. She wants answers in the app she already uses.



## 4. Why Ollama as Primary LLM

### Options Considered
- OpenAI GPT-4
- Gemini API
- Ollama (local LLM)
- Anthropic Claude

### Decision
We chose Ollama with Llama3 as primary and Gemini as fallback.

### Reasons
- Merchant store data stays completely private — nothing sent to third party AI
- No API costs per request — important for a tool that runs multiple times per merchant
- We can switch models easily without changing code
- Gemini fallback ensures the bot always responds even if Ollama is down

### Tradeoff
- Local LLM quality is lower than GPT-4 or Claude for complex reasoning
- Response time is slower especially for larger product catalogs
- ngrok tunnel required to connect local Ollama to n8n cloud
- ngrok URL changes on every restart — must update manually in n8n

### Model Choice
We tested Gemma3:1b, Llama3.2:3b, and Llama3.1:8b. We settled on Llama3.1:8b for best output quality and format adherence.



## 5. Why Google Sheets Instead of a Database

### Options Considered
- PostgreSQL
- Firebase
- Supabase
- Google Sheets

### Decision
We chose Google Sheets.

### Reasons
- Zero infrastructure setup — no server, no hosting, no configuration
- n8n has native Google Sheets integration
- Fast to implement for MVP
- Easy to inspect and debug data during development

### Tradeoff
- Not secure for production — credentials stored in plain text
- Cannot handle concurrent merchants at scale
- No proper indexing or query performance
- Would need to be replaced with a real database before production



## 6. Why Read-Only Shopify Access

### Options Considered
- Full Shopify API access (read and write)
- Read-only access (read_products, read_content)

### Decision
We chose read-only access.

### Reasons
- Builds merchant trust — bot cannot change prices, delete products, or touch orders
- Reduces security risk significantly
- Simpler permission scope is easier for merchants to approve
- Aligns with our product philosophy — show fixes, do not auto-fix

### Tradeoff
- Cannot auto-apply description rewrites directly to Shopify
- Merchant must copy suggestions manually into their store
- Limits future automation features



## 7. Why We Show Fixes Instead of Auto-Fixing

### Options Considered
- Auto-apply AI rewrites directly to Shopify products
- Show suggestions and let merchant decide

### Decision
We chose to show suggestions only.

### Reasons
- Trust is more important than automation at this stage
- Merchants should review and approve changes to their own store
- Auto-fixing could overwrite content merchants intentionally wrote
- Simpler to implement and explain

### Tradeoff
- More manual work for the merchant
- Reduces the wow factor of full automation



## 8. Why Prompt-Based Audit Instead of Rule-Based Scoring

### Options Considered
- Hardcoded rules (if description length < 50 words then flag)
- LLM-based analysis with structured prompts

### Decision
We chose LLM-based analysis.

### Reasons
- Handles any store structure without writing custom rules per product type
- Can evaluate clarity, tone, and customer friendliness — not just character count
- More flexible and extensible
- Closer to how an actual AI shopping agent reads a store

### Tradeoff
- LLM output can be inconsistent
- Format instructions are sometimes ignored by smaller models
- Harder to guarantee exact output structure every time
- Requires careful prompt engineering to minimize hallucination



## 9. Why ngrok Instead of Deploying Ollama to a Server

### Options Considered
- Deploy Ollama to a cloud VM (AWS, GCP, Azure)
- Use ngrok tunnel to expose local Ollama

### Decision
We chose ngrok for the hackathon.

### Reasons
- Zero cost — no server bills during development and demo
- Fast setup — running in minutes
- Sufficient for demo and testing purposes

### Tradeoff
- ngrok free tier URL changes every session
- Must manually update the URL in all Ollama HTTP Request nodes in n8n after each restart
- Not reliable for production use

### Future Fix
Replace ngrok with Cloudflare Tunnel which is free and gives a permanent URL.



## 10. Fallback Strategy — Ollama to Gemini

### Decision
Added Gemini 2.0 Flash as automatic fallback when Ollama returns empty response or fails.

### How It Works
- Ollama HTTP Request node has Continue on Fail enabled
- IF node checks if response field is empty
- If empty — same prompt is sent to Gemini API automatically
- Both outputs merge before sending to Telegram

### Reasons
- Ensures bot always responds even during Ollama downtime
- Gemini is fast and reliable as a backup
- Merchant never sees a failure — they just get a response

### Tradeoff
- Gemini API has usage costs if triggered frequently
- Prompt may need slight adjustment for Gemini vs Ollama response format



## Summary Table

| Decision | Chose | Rejected | Main Reason |
|---|---|---|---|
| Backend | n8n | FastAPI, Node.js | Speed and simplicity |
| Interface | Telegram | Web dashboard | Merchant already there |
| Primary LLM | Ollama | OpenAI, Gemini | Privacy and cost |
| Fallback LLM | Gemini | None | Reliability |
| Storage | Google Sheets | PostgreSQL, Firebase | Zero setup for MVP |
| Shopify access | Read-only | Full access | Trust and safety |
| Audit method | LLM prompts | Hardcoded rules | Flexibility |
| Tunnel | ngrok | Cloud VM | Cost and speed |
| Fix approach | Show suggestions | Auto-fix | Merchant trust |
