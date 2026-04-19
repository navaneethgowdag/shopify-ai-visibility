**AI Store Readability Auditor**

**What is this?**



This project helps Shopify store owners check

why their store is not visible to AI tools like ChatGPT.



It finds problems and tells what to fix.



**Problem**



AI tools are now helping people to buy products.



But many Shopify stores are not shown.



**Reason:**

Missing product details

Weak descriptions

No clear data for AI



So AI cannot understand the store



**Solution**



We built a simple Telegram bot.



**User just:**



Send store URL

Send API token

Type "audit my store"





**How it works**

Telegram → n8n → Shopify → AI → Telegram

**Steps**

1\. Start bot

/start

2\. Send store URL

yourstore.myshopify.com

3\. Send API token

shpat\_xxxxxx

4\. Run audit

audit my store



**Output**

Score (out of 10)

What is good

What is wrong

What to fix



**Tech Used**

n8n

Telegram Bot

Shopify API

Ollama (gemma model)

Google Sheets



**Limitations**

Only Shopify

Needs API token

Basic audit



**Goal**

Make store easy for AI to understand

