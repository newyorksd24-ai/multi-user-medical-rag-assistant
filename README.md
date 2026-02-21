# 🏥 Health Assistant — n8n Workflow

> An AI-powered Telegram bot that ingests Italian medical reports (PDF), stores them as vector embeddings in Supabase, and answers health-related questions using RAG (Retrieval-Augmented Generation).

**⚠️ Language note:** The medical reports used in this project are in **Italian** (`referti medici ITA`). All user-facing Telegram messages (welcome, confirmations, prompts) are also written in Italian. The AI Agent is multilingual and will respond in the same language the user writes in.

---

## 📌 Introduction

This project is a demo workflow built with **n8n** as part of a personal automation portfolio. It simulates a personal health document assistant that a user can interact with entirely via Telegram.

The user can:
- Send `/start` to get a welcome message
- Upload a **PDF medical report** (referto medico) — the bot extracts, embeds and stores it
- Ask **natural language questions** about the uploaded documents

The assistant never diagnoses. It only reads and summarizes what is in the documents, with a built-in **Opinion Mode** for informational evaluations and a mandatory medical disclaimer.

---

## 🗂️ Architecture Overview

The workflow is split into **3 synchronized execution paths**, all triggered from a single Telegram Trigger node.

### Part 1 — Entry Point & Message Routing

![Architecture Part 1 — Entry & Routing](Project_Health-Assistant-Part-1-welcome.jpg)

| Step | Node | Description |
|------|------|-------------|
| 1 | `Telegram Trigger` | Listens for any incoming message (text or document) |
| 2 | `Message Type` (Switch) | Routes by type: document → PDF pipeline, `/start` → Welcome, text → AI Agent |
| 3 | `Welcome` | Sends an Italian welcome message with usage examples |

The Switch node uses **3 output routes** based on strict conditions:
- **Output 0**: `message.document.file_id` is not empty
- **Output 1**: `message.text` equals `/start`
- **Output 2**: `message.text` is not empty (any other text)

---

### Part 2 — PDF Ingestion & Vector Storage

![Architecture Part 2 — PDF Ingestion Pipeline](Project_Health-Assistant_Part-2.jpg)

| Step | Node | Description |
|------|------|-------------|
| 1 | `HTTP Request - Get File Path` | Calls Telegram API `getFile` to resolve the file path |
| 2 | `DownloadFile` | Downloads the PDF binary from Telegram CDN |
| 3 | `Extract PDF Text` | Extracts raw text from the PDF |
| 4 | `Check Text Length` | Code node: counts characters, sets `needsOCR = true` if `charCount < 100` |
| 5 | `Needs OCR?` | IF node: branches on `needsOCR` boolean |
| 6a | `OCR via Mistral` | POST to `mistral.ai/v1/ocr` using `mistral-ocr-latest` (scanned/image PDFs) |
| 6b | `Extract OCR Text` | Code node: extracts markdown pages from Mistral response |
| 7 | `Merge` (append) | Reunites OCR path and direct-text path into a single stream |
| 8 | `Supabase Vector Store1` | Inserts embeddings into `documents` table with `user_id` + `filename` metadata |
| 9 | `Aggregate` | Collects all inserted chunk metadata |
| 10 | `Confirm Save` | Telegram: "✅ Documento salvato!" |
| 11 | `Ask Opinion` | Telegram: prompts user to request an informational evaluation |

**Sub-nodes attached to Vector Store (ingestion):**
- `Embeddings OpenAI1` — generates text embeddings
- `Default Data Loader` — loads text with metadata
- `Recursive Character Text Splitter` — chunks with 200-token overlap

---

### Part 3 — AI Agent & RAG Query

![Architecture Part 3 — AI Agent & RAG](Project_Health-Assistant_Part-3.jpg)

| Step | Node | Description |
|------|------|-------------|
| 1 | `AI Agent` | LangChain Agent node (n8n), receives user text from Telegram |
| 2 | `OpenAI Chat Model` | Model: `gpt-4.1-mini` |
| 3 | `Simple Memory` | Window Buffer Memory, 10-message context window, keyed by `chat.id` |
| 4 | `Supabase Vector Store` | Retrieve-as-tool mode, filters by `user_id` metadata |
| 5 | `Embeddings OpenAI` | Generates query embeddings for similarity search |
| 6 | `Send Reply` | Sends the agent's response back to the user via Telegram |

The agent uses **metadata filtering** (`user_id`) so each user only accesses their own documents.

---

## ⚙️ Setup & Configuration

### Prerequisites

| Service | Purpose |
|---------|---------|
| n8n (self-hosted or cloud) | Workflow engine |
| Telegram Bot (via @BotFather) | User interface |
| OpenAI API key | GPT-4.1-mini + Embeddings |
| Mistral AI API key | OCR for scanned PDFs |
| Supabase project | Vector database (pgvector) |

### Supabase Setup

Run this SQL in your Supabase project to create the vector store table:

```sql
-- Enable pgvector extension
create extension if not exists vector;

-- Create documents table
create table documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(1536)
);

-- Create match function for similarity search
create or replace function match_documents (
  query_embedding vector(1536),
  match_count int default 5,
  filter jsonb default '{}'
) returns table (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
language plpgsql
as $$
begin
  return query
  select
    id,
    content,
    metadata,
    1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where metadata @> filter
  order by documents.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

### n8n Credentials Required

1. **Telegram API** — Bot token from @BotFather
2. **OpenAI API** — Used for both Chat Model and Embeddings
3. **Mistral Cloud API** — Used for OCR on scanned PDFs
4. **Supabase API** — Project URL + service role key

### Import & Activate

1. Import `Project_Health-Assistant.json` into your n8n instance
2. Set all credentials in the nodes
3. Replace `<TOKEN>` placeholders in the HTTP Request nodes with your actual Telegram bot token
4. Set your Telegram webhook or use polling
5. Activate the workflow

---

## 🧠 AI Agent System Prompt

The agent is configured with a strict system prompt (in English) that enforces:

- **No hallucination**: Only uses content from retrieved documents
- **Opinion Mode**: When user asks for evaluation, provides informational analysis with cautious language ("may be associated with", "could suggest")
- **Mandatory disclaimer**: Every Opinion Mode response ends with a medical disclaimer in the user's language
- **Multilingual**: Responds in whatever language the user writes in (IT, EN, ES, DE supported)
- **No diagnosis**: Cannot state "you have [disease]", prescribe medications, or give treatment recommendations

---

## 💬 Live Demo

Real conversation captured from the Telegram bot during testing. The medical report used is an **Italian cardiology dossier** covering December 2019 – September 2020.

---

**User** → `/start`

**Bot** →
> 👋 Benvenuto nel tuo Health Assistant!
> 📄 Carica documenti medici (PDF) e poi fammi domande su di essi.
> Esempi: "Qual era il mio valore di colesterolo?", "Riassumi le mie ultime analisi"

---

**User** → uploads `Referto 2022.pdf` (536 KB)

**Bot** →
> ✅ Documento salvato! 📄 Referto 2022.pdf — Ora puoi farmi domande su questo documento.
> 📄 Se vuoi una nostra analisi informativa, scrivimi: *"Valuta questo referto e fammi un riepilogo per il medico"*

---

**User** → *"Sì grazie voglio una valutazione"*

**Bot** → *(asks for clarification: blood tests, ECG, echocardiogram, or full overview?)*

**User** → *"L'ultimo documento inviato"*

**Bot** →
> Ecco una valutazione informativa basata sul dossier cardiologico:
>
> 1. **Esami del sangue** — Lieve alterazione lipidica e glicemica a dic 2019. Picco trigliceridi/colesterolo ad apr 2020, con miglioramento a set 2020. Possibile rischio cardiovascolare aumentato.
> 2. **ECG** — Ritmo sinusale normale. Deviazione assiale sinistra e sospetta ipertrofia ventricolare.
> 3. **Ecocardiogramma** — Ispessimento pareti ventricolari, funzione di pompa conservata. Aorta ectasica (~42 mm). Insufficienza mitralica lieve e aortica lieve-moderata.
> 4. **Ecocolordoppler carotidi** — Placca non ostruttiva presente.
>
> *Questa è solo una valutazione informativa. La diagnosi e le scelte terapeutiche spettano al tuo cardiologo.*

---

## 🛡️ Error Handling

| Scenario | Handling Strategy |
|----------|------------------|
| PDF with no extractable text (`charCount < 100`) | Automatic OCR fallback via Mistral `mistral-ocr-latest` |
| Invalid Mistral OCR response | Code node throws descriptive error with full JSON dump |
| User queries documents with no uploads | Vector store returns empty; agent responds "no documents found" |
| OCR and text paths rejoined | `Merge (append)` node normalizes both into a single data stream |
| Supabase insert failure | Execution stops; n8n execution log captures the error |
| Telegram API timeout | n8n default retry on HTTP nodes (configure manually if needed) |

---

## 🔐 Privacy & Multi-user Isolation

Each document is stored with a `user_id` metadata field (Telegram `from.id`). Both the ingestion pipeline and the retrieval tool filter by this field, ensuring **complete data isolation** between users.

> This is a demo project. For production use, additional security layers (authentication, rate limiting, data deletion) would be required.

---

## 🚀 Future Improvements

- Replace Window Buffer Memory with **persistent Postgres/Supabase memory** for cross-session context
- Add **document deletion command** via Telegram (e.g., `/delete [filename]`)
- Support **image uploads** with Mistral Vision for non-PDF reports
- Add **voice message support** via OpenAI Whisper transcription
- Implement **multi-document timeline comparison** (e.g., "How did my cholesterol trend in 2023 vs 2024?")
- Generate a **weekly health summary** delivered automatically via Telegram
- Build a lightweight **web dashboard** for document management (upload, view, delete)
- Add **rate limiting per user** to prevent API cost abuse in production

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Automation engine | n8n (v1.x) |
| Interface | Telegram Bot API |
| LLM | OpenAI GPT-4.1-mini |
| Embeddings | OpenAI text-embedding (1536 dim) |
| OCR | Mistral AI (`mistral-ocr-latest`) |
| Vector DB | Supabase (pgvector) |
| Memory | n8n Window Buffer Memory |
| Architecture pattern | RAG (Retrieval-Augmented Generation) |

---

## 📁 Repository Structure

```
.
├── Project_Health-Assistant.json                  # n8n workflow export
├── Project_Health-Assistant-Part-1-welcome.jpg    # Architecture slide 1
├── Project_Health-Assistant_Part-2.jpg            # Architecture slide 2
├── Project_Health-Assistant_Part-3.jpg            # Architecture slide 3
└── README.md
```

---

## 👤 Author

**Fabio Roggero**

- 🌐 Languages: Italian, Spanish, English, German
- 🎯 Focus: Workflow automation, n8n, AI integration
- 📧 Contact: [fabio.roggero90@gmail.com](mailto:fabio.roggero90@gmail.com)
- 🔗 LinkedIn: [linkedin.com/in/f-roggero](https://www.linkedin.com/in/f-roggero)

---

*This project is for educational and demonstration purposes only. It does not provide medical advice.*
