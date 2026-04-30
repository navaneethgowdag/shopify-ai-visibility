# Contributing — AI Store Readability Auditor

## 1. Overview

This document outlines the roles, responsibilities, and contributions of each team member for the **AI Store Readability Auditor** — a Telegram-based AI agent that audits Shopify stores for machine readability and AI agent discoverability.

The project spans product thinking, system design, backend workflow engineering, LLM integration, and prompt design. Both **Navaneeth Gowda G** and **Manjunath H** contributed collaboratively across all major areas of the project, including problem definition, system architecture, implementation, and testing. Responsibilities were shared to ensure balanced involvement in both product thinking and engineering execution.

---

## 2. Team Members

| Name | Role |
|---|---|
| Navaneeth Gowda G | Project Lead — Product, Architecture & AI Engineering |
| Manjunath H | Supporting Engineer — Backend Workflows & API Integration |

---

## 3. Contribution Breakdown

### Navaneeth Gowda G — Project Lead

Navaneeth drove the project from conception to completion. He owned the problem definition, product design, system architecture, LLM integration, prompt engineering, and end-to-end testing.

**Problem Definition and Product Strategy**
- Identified the core problem: independent Shopify merchants are invisible to AI shopping agents because their stores lack machine-readable structure, not because of poor products or pricing
- Defined the target user — non-technical Shopify merchants who need simple, actionable fixes without touching code or hiring developers
- Framed the product around a narrow, high-value insight: agent-influenced commerce is growing faster than merchants realize, and early movers will have a structural advantage that compounds
- Decided the product should explain the problem clearly first before surfacing fixes — because most merchants do not know this problem exists

**Feature Design and Scope**
- Designed all five core commands — Audit, Fix, Perception, Status, and Chat — defining the scope, trigger conditions, and expected output format for each
- Specified the Audit output structure: overall score, health summary, strengths, issues, missing fields, trust signal gaps, AI perception summary, and a ranked action plan (Critical, High, Medium)
- Designed the Perception command to go beyond field-checking — simulating what an AI shopping agent would say about the store, what questions it cannot answer, and why it would skip the store in favor of a competitor
- Defined the Fix command constraints: only target products with descriptions under 40 words, or missing price, SKU, vendor, images, inventory, or weight; generate rewrites of 40–60 words in benefit-focused plain language
- Made the deliberate call not to auto-apply fixes — showing rewrites ready to paste rather than writing directly to Shopify, prioritizing trust over automation at the prototype stage

**System Architecture**
- Designed the complete end-to-end data flow: Telegram → n8n → Shopify Admin API → Ollama (primary LLM) → Gemini (fallback) → Telegram response
- Chose n8n as the automation layer to avoid building a full backend, enabling fast iteration and visual debugging of each workflow
- Chose Telegram as the merchant interface because merchants are already there — zero onboarding friction, no new app, no login, no dashboard
- Selected Ollama with Llama3.1:8b as the primary LLM to keep merchant data private on local infrastructure, with Gemini 2.0 Flash as a reliability fallback
- Chose Google Sheets as lightweight credential storage for the prototype phase, with clear awareness it would need to be replaced with a database at scale

**LLM Prompt Engineering**
- Designed all LLM prompts across Audit, Fix, Perception, and Chat flows
- Applied systematic hallucination prevention across all prompts: explicit product counts stated upfront, product titles listed before field data so the model cannot confuse field values with product names, and strict instructions to never invent products or use placeholder values like NOT SET as product names
- Wrote the Chat prompt with four strictly separated cases — greeting, list all products, specific product query, off-topic — and explicit instructions never to blend cases
- Enforced plain text output in all prompts: no markdown, no asterisks, no symbols, no backslashes — ensuring clean Telegram delivery
- Iterated on prompt output quality across multiple test runs against real Shopify store data

**LLM Infrastructure and Fallback Architecture**
- Set up Ollama running Llama3.1:8b locally and exposed it to n8n cloud via ngrok tunnel
- Implemented the complete fallback pattern applied independently across all four AI flows: Ollama HTTP Request (Continue on Fail enabled) → IF node checks `$json.response` not empty → TRUE path to Merge, FALSE path to Gemini → Merge → response parsing → Telegram send
- Configured `stream: false` on all Ollama requests to ensure single complete responses that n8n can parse reliably
- Wrote Code nodes for each flow to strip newlines and serialize the JSON request body before sending to Ollama, preventing malformed request errors
- Configured each Gemini fallback node independently per flow, pointing to its own prompt field in Edit Fields

**Token Validation and Merchant Onboarding**
- Designed the two-step merchant setup: store URL submission followed by read-only API token submission
- Built the token validation flow using the Shopify `shop.json` lightweight endpoint — token is validated immediately before saving, and merchants are told instantly if the token is invalid or expired
- Ensured the token is stored once in Google Sheets and never asked for again in subsequent sessions

**Testing and Reliability**
- Tested all flows end-to-end against real Shopify store data
- Identified and resolved the newline-in-JSON-body failure mode that caused bad request errors to Ollama
- Applied response trimming to 4000 characters before Telegram send to stay within the 4096 character limit
- Set up reply keyboard buttons after every response so merchants never need to type commands

---

### Manjunath H — Supporting Engineer

Manjunath contributed to backend workflow implementation and Shopify API integration, working within the architecture and specifications defined by Navaneeth.

**n8n Workflow Implementation**
- Assisted in building n8n workflow nodes for the Audit and Fix commands
- Implemented the Switch node routing logic using exact string matching on lowercased, trimmed user input across all defined routes
- Set up the Google Sheets read and write nodes for credential storage and lookup

**Shopify API Integration**
- Integrated the Shopify Admin REST API products endpoint with the `X-Shopify-Access-Token` header
- Configured the products fetch to retrieve title, description, status, vendor, variants, and images for up to 50 products
- Implemented HTML stripping using JavaScript regex (`replace(/<[^>]*>/g, '')`) to clean `body_html` before passing to the LLM

**Documentation Support**
- Assisted in writing the technical documentation covering the system architecture diagram, data flow steps, and known limitations
- Documented the fallback node mapping across flows and the IF node condition used in each

---

## 4. Collaboration Areas

**Technology Stack Evaluation**
Both members discussed the full stack together — Telegram Bot API, n8n, Shopify Admin API, Ollama, Gemini, ngrok, and Google Sheets — with Navaneeth making the final calls on each tool based on the product and privacy requirements.

**Scope Decisions**
Both members discussed what not to build. The decision to exclude SEO tools, UI improvement features, and auto-fix functionality was led by Navaneeth and aligned with Manjunath.

**End-to-End Testing**
Both members tested flows using real Shopify store data, with Navaneeth leading prompt iteration and Manjunath verifying node outputs and API responses.

---

*This project was built as a functional prototype demonstrating AI-driven store auditing over a conversational interface. All components are integrated and operational end-to-end.*
