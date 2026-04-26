# Technical Report: AI Store Readability Auditor

---

## 1. What This System Does

This is a Telegram-based AI agent that connects to a Shopify store and audits it the way an AI shopping agent would. A merchant sends messages through Telegram and gets back a readable report showing what is working, what is missing, and what to fix first. No dashboard. No code. Just chat.

---

## 2. System Architecture

The system has six main parts connected together:

- **Telegram Bot** — The merchant interface. Every message comes here first.
- **n8n (cloud)** — The brain. Receives the message, decides what to do, calls Shopify, sends data to the LLM, and replies back.
- **Shopify Admin API** — Read-only access. Fetches product titles, descriptions, and page content. Cannot modify anything.
- **Ollama (local LLM)** — Runs Llama3.1:8b on the developer machine. Merchant data stays private.
- **Gemini API** — Fallback LLM. If Ollama fails, the same prompt goes to Gemini automatically.
- **Google Sheets** — Stores merchant credentials. One row per merchant.

### Data Flow

```
Merchant sends message on Telegram
        ↓
Telegram Trigger fires in n8n
        ↓
Edit Fields node extracts chatId, userText, firstName
        ↓
Switch node routes to correct flow
        ↓
Google Sheets fetches saved credentials (storeURL, token)
        ↓
Shopify Admin API fetches store data (read-only)
        ↓
Code node builds LLM request body
        ↓
Ollama (Llama3.1:8b) analyzes data locally
        ↓
IF node checks if Ollama returned valid response
        ↓
Gemini API as fallback if Ollama is empty or failed
        ↓
Merge node combines whichever LLM responded
        ↓
Edit Fields parses response
        ↓
Telegram sends report back to merchant
```

### Switch Routes

| Route | Trigger | Action |
|---|---|---|
| `/start` | User sends /start | Welcome message |
| `submit_url` | Message contains .myshopify.com | Save store URL to Sheets |
| `submit_token` | Message starts with shpat_ | Validate token, save to Sheets |
| `/audit` | Message contains /audit | Full store readability audit |
| `/fix` | Message contains /fix | Product description rewrites |
| `/perception` | Message contains /perception | AI agent perception report |
| `/chat` | Message contains /chat | Conversational store assistant |
| `/status` | Message contains /status | Show connected store info |
| `default` | Anything else | AI assistant with menu |

---

## 3. Key Implementation Decisions

- **Telegram as UI** — Merchants already use it. Zero onboarding friction. No new app, no login, no dashboard.
- **n8n for automation** — Connects all services without writing a full backend. Visual workflow makes debugging fast.
- **Local LLM first** — Merchant data stays private. Gemini only used as fallback when Ollama fails.
- **Read-only API** — Token has only read_products and read_content permissions. Bot cannot modify any store data.
- **One-time token setup** — Stored in Google Sheets after first entry. Merchant never asked again.
- **Token validation** — Every new token is validated against shop.json before saving. Invalid tokens are rejected immediately with a clear message.
- **Reply keyboard buttons** — Merchants never need to type commands manually. Four buttons always visible after each response.
- **HTML stripping** — Shopify product descriptions contain HTML tags. These are stripped before sending to the LLM using `.replace(/<[^>]*>/g, '')` to avoid noisy prompts.

---

## 4. What AI Does vs What Deterministic Code Does

### Deterministic Code Handles

- **Routing** — Switch node does exact string matching for /audit, /fix, /perception, /chat, /status
- **Credential storage and lookup** — Google Sheets read and write via n8n nodes
- **Shopify API calls** — HTTP Request nodes with exact endpoints
- **Token validation** — shop.json call checks if token is valid before saving
- **HTML stripping** — JavaScript expression removes HTML tags from product descriptions before LLM sees them
- **Fallback routing** — IF node checks Ollama response and routes to Gemini if empty
- **Message trimming** — Response trimmed to 4000 characters to stay within Telegram limits

### AI Handles

- Scoring product descriptions for clarity and completeness
- Generating ranked priority action plans (Critical, High, Medium)
- Simulating how an AI shopping agent would describe the store to a customer
- Identifying trust signal gaps (reviews, policies, brand story)
- Rewriting weak or missing product descriptions in customer-friendly language
- Answering merchant questions about their specific store products
- Handling greetings, off-topic messages, and product queries conversationally

### Why This Line

Routing and data fetching need to be reliable, fast, and deterministic. If the Switch node used AI to decide where to route a message, it could fail unpredictably. AI is only used where language understanding and generation are genuinely needed — analysis, scoring, rewriting, and conversation.

---

## 5. Prompt Design

Each flow has a carefully structured prompt to minimize hallucination and enforce clean output.

### Audit Prompt Approach
- Tells LLM exact number of products in the store
- Lists product titles explicitly before data so LLM cannot confuse field values with product names
- Instructs LLM to only flag fields that are actually missing
- Enforces plain text output with no markdown, no symbols, no asterisks
- Uses ranked priority sections (Critical, High, Medium) for actionable output

### Fix Prompt Approach
- Passes product count and titles explicitly
- Conditions are strict — only fix if description under 40 words, or price/SKU/vendor/images/inventory/weight missing
- Missing fields section only lists fields that are actually absent
- Descriptions must be 40 to 60 words, benefit focused, customer friendly

### Perception Prompt Approach
- Simulates an AI shopping agent speaking directly to the merchant
- Shows what the agent can and cannot answer
- Explains why the store is not being recommended right now
- Lists competitor advantages based on missing data
- Ends with a single honest bottom line sentence

### Hallucination Prevention
- Product titles are listed at the top of every prompt
- LLM is told the exact number of products
- Prompts say: never invent products, never use field values as product names
- If data is missing, say exactly: Not available

---

## 6. Failure Handling

- **Shopify API down or invalid token** — HTTP Request node has Continue on Fail enabled. IF node checks the response. If token is invalid, merchant gets a plain message asking them to re-enter. Token is validated using the lightweight shop.json endpoint before saving.
- **Ollama returns empty response** — IF node detects empty response field and automatically sends the same prompt to Gemini API as fallback.
- **Ollama is unreachable** — Continue on Fail on Ollama HTTP Request node routes to Gemini without stopping the workflow.
- **LLM returns messy output** — Prompts explicitly instruct the model to use plain text only, no markdown, no asterisks, no backslashes. HTML is stripped from inputs before the LLM sees them.
- **Unexpected user input** — Default route in Switch node catches all unmatched messages and sends them to the AI assistant which handles greetings, product questions, and off-topic messages.
- **Telegram message too long** — Response is trimmed to 4000 characters before sending to stay within Telegram's 4096 character limit.
- **ngrok URL changes** — Currently a manual process. URL must be updated in all Ollama HTTP Request nodes after each ngrok restart.

---

## 7. Known Limitations

- **ngrok URL changes every session** — Requires manual update in n8n each restart. Not suitable for production.
- **Llama3.1:8b output quality** — Better than smaller models but still inconsistent for complex format instructions. Occasionally ignores formatting rules.
- **Google Sheets is not a proper database** — Not secure for production. Credentials stored in plain text. Cannot handle concurrent merchants at scale.
- **No audit history** — Each audit is independent. Merchants cannot track score improvement over time.
- **Single merchant testing** — System is built and tested for one merchant at a time. Multi-merchant handling needs more robust credential management.
- **Local machine dependency** — Ollama runs on the developer's machine. If the machine is off, Gemini fallback handles requests but long-term reliability requires a deployed solution.

---

## 8. What We Would Improve With More Time

- Replace ngrok with Cloudflare Tunnel for a permanent free URL with no manual updates
- Add audit history so merchants can track score changes week over week
- Replace Google Sheets with a proper database (Supabase or PostgreSQL) for security and scale
- Add scheduled weekly audit reports sent automatically to merchants
- Deploy Ollama to a cloud VM to remove the local machine dependency
- Use a larger LLM (Llama3.1:70b or GPT-4) for higher quality and more consistent output
- Support multiple merchants properly with better credential isolation
- Add a simple web dashboard for merchants who prefer a visual interface
- Implement direct Shopify write access (optional) so approved rewrites can be applied in one tap
