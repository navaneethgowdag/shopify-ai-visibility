# Technical Report: AI Store Readability Auditor



## 1. What This System Does

This is a Telegram-based AI agent that connects to a Shopify store and audits it the way an AI shopping agent would. A merchant sends messages through Telegram and gets back a readable report showing what is working, what is missing, and what to fix first. No dashboard. No code. Just chat.



## 2. System Architecture

The system has six main parts connected together:

- **Telegram Bot** — The merchant interface. Every message comes here first.
- **n8n (cloud)** — The brain. Receives the message, decides what to do, calls Shopify, sends data to the LLM, and replies back.
- **Shopify Admin API** — Read-only access. Fetches product titles, descriptions, vendor, variants, images, and status. Cannot modify anything.
- **Ollama (local LLM)** — Runs Llama3.1:8b on the developer machine via ngrok tunnel. Merchant data stays private.
- **Gemini API** — Fallback LLM. If Ollama fails or returns empty, the same prompt goes to Gemini 2.0 Flash automatically.
- **Google Sheets** — Stores merchant credentials. One row per merchant identified by Telegram chatId.

### Data Flow

```
Merchant sends message on Telegram
        ↓
Telegram Trigger fires in n8n
        ↓
Edit Fields node extracts chatId, userText, firstName
        ↓
Switch node routes to correct flow using string matching
        ↓
Google Sheets fetches saved credentials (storeURL, token)
        ↓
Shopify Admin API fetches store products (read-only)
        ↓
Edit Fields builds structured LLM prompt
        ↓
Code node strips newlines and serializes JSON request body
        ↓
HTTP Request sends POST to Ollama via ngrok tunnel
        ↓
IF node checks if $json.response is not empty
   ↓ TRUE (Ollama responded)     ↓ FALSE (Ollama failed)
Merge node                  Gemini API fallback (POST)
                                   ↓
                             Merge node
        ↓
Edit Fields parses response field
        ↓
Telegram sends report back to merchant
```



## 3. Switch Routing Logic

The Switch node uses exact string matching on `$json.userText` — the lowercased, trimmed version of the merchant's message. All routes are deterministic — no AI involved in routing.

| Route | Condition | Action |
|---|---|---|
| `help` | contains `/help` | Show available commands |
| `start` | contains `/start` | Welcome message |
| `submit_url` | contains `.myshopify.com` | Save store URL to Sheets |
| `submit_token` | starts with `shpat_` | Validate token, save to Sheets |
| `audit` | contains `/audit` | Full store readability audit |
| `fix` | contains `/fix` | Product description rewrites |
| `perception` | contains `/perception` | AI agent perception report |
| `status` | contains `/status` | Show connected store info |
| `chat` | contains `/chat` | Conversational store assistant |
| `default` | is not empty | AI assistant with command menu |



## 4. Shopify API Integration

All Shopify calls use the Admin REST API with read-only access. The token is passed via the `X-Shopify-Access-Token` header.

### Products Endpoint

```
GET https://{storeURL}/admin/api/2024-01/products.json
    ?limit=50
    &fields=title,body_html,status,vendor,variants,images
```

This fetches title, description (HTML), status, vendor, variants (price, SKU, inventory, weight), and images for up to 50 products.

### Token Validation Endpoint

```
GET https://{storeURL}/admin/api/2024-01/shop.json
Header: X-Shopify-Access-Token: {token}
```

This lightweight call validates the token before saving. If it returns a valid shop object, the token is saved. If it returns an error, the merchant is told immediately.

### HTML Stripping

Shopify product descriptions are stored as HTML. Before sending to the LLM, tags are stripped:

```javascript
p.body_html ? p.body_html.replace(/<[^>]*>/g, '').trim() : 'MISSING'
```

This prevents the LLM from seeing raw HTML and producing noisy output.



## 5. LLM Integration

### Ollama — Primary LLM

Ollama runs locally on the developer machine and is exposed to n8n cloud via ngrok tunnel.

**HTTP Request Configuration:**
```
Method:  POST
URL:     https://fringe-dedicate-rethink.ngrok-free.dev/api/generate
Header:  ngrok-skip-browser-warning: true
Body:    {{ $json.requestBody }}  (pre-serialized JSON string)
```

### Code Node — Request Body Builder

Each flow has a Code node that serializes the prompt before sending to Ollama. This prevents newline characters from breaking the JSON body:

```javascript
const prompt = $input.item.json.promptText.replace(/\n/g, ' ');

return {
  json: {
    requestBody: JSON.stringify({
      model: "llama3.1:8b",
      prompt: prompt,
      stream: false
    })
  }
};
```

`stream: false` ensures Ollama returns a single complete response. Streaming would break n8n response parsing.

### Gemini — Fallback LLM

**HTTP Request Configuration:**
```
Method:  POST
URL:     https://generativelanguage.googleapis.com/v1beta/models/
         gemini-2.0-flash:generateContent?key={API_KEY}
Body (JSON):
{
  "contents": [{"parts": [{"text": "{{ promptText }}"}]}]
}
```

Each flow has its own Gemini node pointing to its specific prompt field in Edit Fields.



## 6. Fallback Architecture

Every AI flow has an identical fallback pattern applied independently:

```
Ollama HTTP Request
(Continue on Fail: enabled)
        ↓
IF node: $json.response is not empty?
   ↓ TRUE                    ↓ FALSE
Merge Input 1           Gemini HTTP Request
                              ↓
                        Merge Input 2
        ↓
Edit Fields — parse response
        ↓
Telegram Send Message
```

### IF Node Condition (same across all flows)

```json
{
  "leftValue": "={{ $json.response }}",
  "operator": {
    "type": "string",
    "operation": "notEmpty"
  }
}
```

- **TRUE** → Ollama responded → Merge Input 1
- **FALSE** → Ollama empty or failed → Gemini called → Merge Input 2

### Merge Node

Mode: `Append` — combines whichever branch produced output without requiring matching fields.

### Fallback Nodes Per Flow

| Flow | Ollama Node | IF Node | Gemini Node | Merge Node |
|---|---|---|---|---|
| Audit | HTTP Request2 | If | HTTP Request1 | Merge |
| Fix | HTTP Request4 | If1 | HTTP Request9 | Merge1 |
| Perception | HTTP Request8 | If2 | HTTP Request10 | Merge2 |
| Chat | HTTP Request (chat) | If3 | HTTP Request11 | Merge3 |



## 7. Key Implementation Decisions

- **Telegram as UI** — Merchants already use it. Zero onboarding friction. No new app, no login, no dashboard.
- **n8n for automation** — Connects all services without writing a full backend. Visual workflow makes debugging fast.
- **Local LLM first** — Merchant data stays private. Gemini only used as fallback when Ollama fails.
- **Read-only API** — Token has only `read_products` and `read_content` permissions. Bot cannot modify store data.
- **One-time token setup** — Stored in Google Sheets after first entry. Never asked again.
- **Token validation before saving** — Every token validated against `shop.json` before saving. Invalid tokens rejected immediately.
- **Reply keyboard buttons** — Merchants never need to type commands. Buttons always visible after each response.
- **stream: false on Ollama** — Ensures single complete response instead of streaming tokens.
- **Newline stripping in Code node** — Newlines inside JSON strings cause bad request errors. Stripped before serialization.
- **Continue on Fail on Ollama nodes** — Allows workflow to continue to fallback even when Ollama returns HTTP error.



## 8. What AI Does vs What Deterministic Code Does

### Deterministic Code Handles

- **Routing** — Switch node does exact string matching. No AI involved.
- **Credential storage and lookup** — Google Sheets read and write via n8n nodes.
- **Shopify API calls** — HTTP Request nodes with exact endpoints.
- **Token validation** — `shop.json` call checks token before saving.
- **HTML stripping** — JavaScript regex removes HTML from descriptions.
- **Fallback routing** — IF node checks Ollama response and routes to Gemini if empty.
- **Request serialization** — Code node builds and serializes JSON request body.
- **Response trimming** — Response trimmed to 4000 characters before Telegram send.

### AI Handles

- Scoring product descriptions for clarity and completeness
- Generating ranked priority action plans (Critical, High, Medium)
- Simulating how an AI shopping agent would describe the store
- Identifying trust signal gaps (reviews, policies, brand story)
- Rewriting weak or missing product descriptions
- Answering merchant questions about their store
- Handling greetings, off-topic messages, and product queries

### Why This Line

Routing and data fetching are deterministic because they must be reliable and predictable. A routing failure would break the entire workflow. AI is only used where language understanding is genuinely needed.


## 9. Prompt Design and Hallucination Prevention

### Common Techniques Across All Prompts

- Product count stated explicitly: `This store has exactly X products`
- Product titles listed before data so LLM cannot confuse field values with names
- Prompts instruct: never invent products, never use values like NOT SET as product names
- Plain text only — no markdown, no symbols, no asterisks, no backslashes
- Every section has strict format instructions

### Audit Prompt

Checks all required Shopify fields. Outputs score, health summary, issues, missing fields, trust signals, AI perception, and ranked action plan.

### Fix Prompt

Only fixes products with description under 40 words, or missing price, SKU, vendor, images, inventory, or weight. Never flags present fields. Generates 40 to 60 word benefit-focused rewrites.

### Perception Prompt

Simulates an AI shopping agent speaking to the merchant. Shows what the agent can and cannot answer, why the store is not recommended, competitor advantages, and specific fixes.

### Chat Prompt

Uses only real store data. Never invents details. Each case is strictly separated — greeting, list products, specific product, off-topic — never combined.



## 10. Failure Handling

| Failure | Detection | Response |
|---|---|---|
| Invalid or expired Shopify token | IF node checks shop.json response | Merchant told to re-enter token |
| Ollama unreachable | Continue on Fail on HTTP Request | Routes to Gemini fallback automatically |
| Ollama returns empty response | IF node: `$json.response` not empty | Routes to Gemini fallback automatically |
| LLM returns messy output | Prompt instructs plain text only | Mitigated by strict prompt formatting |
| Telegram message too long | Response trimmed to 4000 chars | Stays within Telegram 4096 char limit |
| Unexpected user input | Switch default route | AI assistant handles it |
| ngrok URL changes | Manual update required | Known limitation |



## 11. Known Limitations

- **ngrok URL changes every session** — Must manually update in all Ollama HTTP Request nodes after each restart.
- **Llama3.1:8b output consistency** — Occasionally ignores formatting rules for complex prompts.
- **Google Sheets** — Not secure for production. Credentials in plain text. Cannot scale.
- **No audit history** — Each audit is independent. No score tracking over time.
- **Local machine dependency** — Ollama requires developer machine to be on. Gemini handles requests if machine is off.



## 12. What We Would Improve With More Time

- Replace ngrok with Cloudflare Tunnel for a permanent free URL
- Add audit history to track score changes over time
- Replace Google Sheets with Supabase or PostgreSQL
- Deploy Ollama to a cloud VM to remove local machine dependency
- Add scheduled weekly audit reports sent automatically
- Use Llama3.1:70b or GPT-4 for higher quality output
- Support multiple merchants with proper credential isolation
- Optional Shopify write access so approved rewrites apply in one tap
