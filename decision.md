**Initial Plan**
At first, I planned to build everything from scratch using code.
I thought of using FastAPI or Node.js to build backend, connect Shopify, write audit logic, and handle LLM responses.

**Problem I Faced**
After thinking more, I understood this will take too much time.
Also, I need to handle many things like APIs, errors, and flow control manually.
I encountered an issue where the API token was exhausted, preventing further requests.

**Final Decision**
So I changed my approach and decided to use n8n.
I switched to Ollama and no longer have to worry about API limits.

**Why I Chose n8n**
It is fast to build
Easy to connect APIs (Shopify, LLM, Telegram)
Workflow is clear and visual
Failure handling is easier (fallback logic)
Tradeoff
Less control compared to full coding
Not best for large-scale production

**Outcome**
Using n8n, I was able to build a working system quickly.
It has real data flow, proper logic, and fallback handling.
