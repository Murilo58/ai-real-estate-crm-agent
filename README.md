# AI Real Estate CRM Agent

## Overview

This n8n template automates the first line of email support for a real estate agency. It reads incoming emails, uses an LLM-powered agent to understand what the client wants, optionally looks up available properties and registers/updates a lead in a Google Sheets CRM, then replies to the client by email — all without human intervention.

The template ships with example business content (a fictitious agency and sample property listings) so it can be imported and tested immediately. The AI persona's instructions are written in English; the CRM spreadsheet field names remain in Portuguese by design (see [Configuration Guide](#configuration-guide)). Everything sample-specific is meant to be replaced with your own data before production use.

## Use Case

Real estate agencies (or any lead-driven business) that want to:

- Give every inbound email a fast, consistent first response
- Automatically qualify and log leads without manual data entry
- Let an AI agent answer routine questions (availability, pricing ranges, documentation, process steps) while keeping a record of the conversation

## How It Works

1. **Gmail Trigger** polls the connected inbox for new emails.
2. The **Real Estate AI Agent** (a LangChain agent node) receives the email body and, guided by its system prompt, decides what to do.
3. If the client asks about properties, the agent calls the **Search Available Properties** tool, which reads a Google Sheets listing.
4. If the client shows real buying/renting interest, the agent calls the **Register Lead in CRM** tool, which appends or updates a row in a Google Sheets CRM (matched by email address).
5. The agent drafts an HTML reply and **Send Email Reply (Gmail)** sends it back to the original sender, reusing the original subject line.
6. **Conversation Memory (Redis)** keeps short-term context per email thread, so the agent remembers earlier messages in the same conversation.
7. The agent uses **Google Gemini** as its primary language model, automatically falling back to **OpenAI** if the primary model fails.

## Features

- Automated, context-aware email replies via an LLM agent
- Automatic lead capture and CRM update in Google Sheets (create or update, matched on email)
- Property lookup against a Google Sheets listing before answering availability questions
- Per-thread conversation memory backed by Redis
- Primary/fallback LLM setup (Google Gemini + OpenAI)
- HTML-formatted email responses

## Workflow Architecture

```
Gmail Trigger
     │
     ▼
Real Estate AI Agent ──▶ Send Email Reply (Gmail)
     │        │  │  │
     │        │  │  └── Primary AI Model (Google Gemini)
     │        │  └───── Fallback AI Model (OpenAI)
     │        └──────── Conversation Memory (Redis)
     └───────────────── Search Available Properties (tool)
                         Register Lead in CRM (tool)
```

The workflow contains 8 functional nodes plus sticky notes documenting configuration. Nodes were renamed from n8n's generic defaults (e.g. "AI Agent", "Send a message") to functional names describing what each one does.

## Requirements

- A self-hosted or cloud n8n instance with the following node packages available: `n8n-nodes-base` (Gmail, Google Sheets) and `@n8n/n8n-nodes-langchain` (Agent, Chat Models, Redis Memory)
- A Gmail account that can be connected via OAuth2
- A Google account with two Google Sheets tabs (properties + leads) — see [Configuration Guide](#configuration-guide)
- A Google Gemini API key
- An OpenAI API key
- A reachable Redis instance (self-hosted or managed)

## Required Credentials

None of the credentials are included in the exported JSON. After importing, you must create and assign your own credentials for:

| Credential | Used by |
|---|---|
| Gmail OAuth2 | Gmail Trigger, Send Email Reply (Gmail) |
| Google Sheets OAuth2 | Search Available Properties, Register Lead in CRM |
| Google Gemini (PaLM) API | Primary AI Model (Google Gemini) |
| OpenAI API | Fallback AI Model (OpenAI) |
| Redis | Conversation Memory (Redis) |

## Installation

1. In n8n, go to **Workflows → Import from File** and select `AI Real Estate CRM Agent.json`.
2. Open every credential-enabled node and assign your own credential (see table above).
3. Open **Search Available Properties** and **Register Lead in CRM** and point each one to your own Google Sheets document and tab (the original document reference has been removed from the template).
4. Review and edit the system prompt on **Real Estate AI Agent** to reflect your own business (see [Configuration Guide](#configuration-guide)).
5. Activate the workflow.

## Configuration Guide

Each integration is also documented directly on the canvas via sticky notes. Summary:

- **Trigger**: Gmail Trigger polls every minute and pulls up to 5 emails per run — adjust in the node's options if needed.
- **AI Model**: Google Gemini (`gemini-2.5-flash`) is the primary model; OpenAI (`gpt-5-mini`) is the fallback, used automatically if the primary model errors out. Both are swappable for other models supported by their respective nodes.
- **Memory**: Redis stores conversation history keyed by the Gmail thread ID (`sessionTTL` = 100s, `contextWindowLength` = 30 messages).
- **Properties sheet**: expects columns such as ID, Tipo, Modalidade, Bairro, Metragem, Quartos, Suítes, Vagas, Preço, Condomínio and Destaques — adapt to your own listings.
- **Leads/CRM sheet**: written via append-or-update, matched on the "E-mail" column. Expected columns: Data, Nome, E-mail, Telefone, Região de Preferência, Orçamento Máximo, Imóvel de Interesse (ID), Resumo da Conversa, Status, Próximo Passo.
- **AI persona**: the system prompt on the agent node is written in English for a fictitious agency ("Nova Lar Imóveis") based in São Paulo, Brazil, with example pricing, service areas, business hours and FAQs. Replace the business details with your own — but keep the tool-usage instructions, since the agent depends on them to decide when to search properties or register a lead.
- **CRM/spreadsheet field names**: the Google Sheets column names referenced by "Register Lead in CRM" (e.g. "Região de Preferência", "Orçamento Máximo (R$)") are intentionally kept in Portuguese, since they must match the header names of the target spreadsheet exactly. If you rename the columns in your own sheet, update the corresponding field mappings on that node to match.

## AI Processing

The **Real Estate AI Agent** node is a LangChain agent that:

- Receives the raw email text as input
- Follows a detailed system prompt that defines its persona, tone, business knowledge, FAQs and mandatory tool-usage rules
- Decides autonomously whether to call the property search tool, the lead registration tool, both, or neither, before composing its reply
- Always responds in valid HTML
- Retries up to 2 times on failure (`retryOnFail`, `maxTries: 2`)

## CRM / Lead Management

Leads are captured only when the agent detects genuine interest (property requests, budget/region/room information, visit requests, etc.), as defined in the system prompt. Missing fields are filled with default placeholder values (e.g. "Not provided", 0) so the row is never left incomplete. Existing leads are matched by email and updated rather than duplicated.

## Testing

To test after installation:

1. Send an email to the connected Gmail account asking about property availability in a region present in your Properties sheet.
2. Confirm the agent replies with an HTML email referencing your sheet data.
3. Check your Leads/CRM sheet for a new or updated row for that sender.
4. Send a follow-up email in the same thread and confirm the agent's reply reflects the earlier conversation (validates Redis memory).
5. Temporarily invalidate the Gemini credential to confirm the OpenAI fallback model takes over.

## Security

- The exported JSON contains no credentials, API keys, tokens, or real spreadsheet/document identifiers — these were replaced with placeholders (e.g. `GOOGLE_SHEETS_URL_REMOVED`) or removed.
- Sample screenshots in this repository have had any real personal contact information redacted.
- All contact details, prices and company information in the system prompt are fictitious example content.
- Review your own Gmail, Google Sheets and Redis access scopes before enabling this workflow against production data.

## Limitations

- The agent relies entirely on the quality/coverage of the system prompt and the connected sheet data — it does not have a general-purpose knowledge base or vector search.
- Property search and lead registration are Google Sheets based; there is no dedicated database or external CRM integration.
- Conversation memory depends on Redis being reachable; if Redis is unavailable, the agent loses cross-message context but does not otherwise fail gracefully (this has not been implemented).
- The workflow polls Gmail on a fixed interval rather than reacting to a push/webhook event.
- The Google Sheets column names used by "Register Lead in CRM" are in Portuguese and must be replicated exactly on your own spreadsheet unless you also update the node's field mappings.

## Troubleshooting

- **No emails are picked up**: verify the Gmail OAuth2 credential has the correct scopes and that the poll interval/`maxResults` matches your expected volume.
- **Agent replies without checking properties**: confirm the Google Sheets credential and document/tab are correctly set on "Search Available Properties" and that the sheet has data.
- **Leads are not appearing in the CRM sheet**: check that "Register Lead in CRM" points to the correct sheet and that the "E-mail" column exists and matches the `matchingColumns` configuration.
- **Agent doesn't remember earlier messages**: verify the Redis credential/connection and that `sessionKey` correctly resolves to the Gmail thread ID.
- **Both AI models fail**: check API key validity and quota for both Google Gemini and OpenAI.

## Customization

- Rewrite the system prompt on "Real Estate AI Agent" for your own agency, language, and rules.
- Swap the Google Gemini/OpenAI models for any other chat model node supported by `@n8n/n8n-nodes-langchain`.
- Extend the CRM schema (add/remove columns) as long as you update the corresponding `$fromAI(...)` field mappings on "Register Lead in CRM".
- Replace Redis memory with another supported memory node if you don't want to run Redis.

## Repository Structure

```
.
├── AI Real Estate CRM Agent.json       # Sanitized n8n workflow — import this file
├── README.md                           # This file
└── imagem/                             # Reference screenshots
    └── 02-google-sheets-properties.png
```

## License

No LICENSE file is currently included in this repository. All rights reserved by the author unless a license is added.

## Disclaimer

This template is provided for educational and demonstration purposes. All company names, contact details, prices and property listings are fictitious sample data. You are responsible for your own credentials, API usage costs, data privacy compliance, and for reviewing the AI-generated content before using it with real clients.
