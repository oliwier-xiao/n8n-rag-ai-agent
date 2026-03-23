# 🤖 n8n Docs RAG AI Agent

> AI-powered chatbot that answers questions about n8n using official documentation via RAG (Retrieval-Augmented Generation).

![n8n](https://img.shields.io/badge/n8n-ffffff?style=for-the-badge&logo=n8n&logoColor=black)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-3ECF8E?style=for-the-badge&logo=supabase&logoColor=white)
![Google Gemini](https://img.shields.io/badge/Gemini-4285F4?style=for-the-badge&logo=google&logoColor=white)

---

## 🎯 What This Does

This n8n workflow creates an **AI assistant** that:

1. **Scrapes official n8n documentation** from GitHub every month
2. **Embeds documents** using Google Gemini embeddings
3. **Stores in Supabase** vector database
4. **Answers questions** via chat interface using RAG

The bot cites sources from official docs and maintains conversation memory.

---

## 📋 Requirements

### Infrastructure

| Service | Version | Purpose |
|---------|---------|---------|
| **n8n** | 1.0+ | Workflow engine |
| **Supabase** | - | Vector database (pgvector) |
| **PostgreSQL** | 14+ | Chat memory storage |

### API Keys

- 🔑 **GitHub** - Personal Access Token
- 🔑 **Google Gemini API** - For embeddings + chat model
- 🔑 **Supabase** - Project URL + API Key

---

## ⚡ Quick Start (Import from URL)

**One-click import to n8n:**

```
https://raw.githubusercontent.com/oliwier-xiao/n8n-rag-ai-agent/main/workflow.json
```

**How to import:**
1. In n8n → Click **Menu** (☰)
2. Select **Import from URL**
3. Paste the link above
4. Configure your credentials (see below)

---

## 🚀 Setup Guide

### Step 1: Create Supabase Project

1. Go to [supabase.com](https://supabase.com) and create a project
2. Enable **pgvector** extension in SQL Editor:
   ```sql
   CREATE EXTENSION IF NOT EXISTS vector;
   ```
3. Enable vector extension and create tables:
   ```sql
   -- Enable pgvector
   CREATE EXTENSION IF NOT EXISTS vector;

   -- Main docs vector store (768 dims for Gemini embeddings)
   CREATE TABLE n8n_docs (
     id bigserial PRIMARY KEY,
     content text,
     metadata jsonb,
     embedding vector(768)
   );

   -- Chat memory
   CREATE TABLE n8n_chat_history (
     id bigserial PRIMARY KEY,
     session_id text NOT NULL,
     message jsonb NOT NULL
   );

   -- Create indexes
   CREATE INDEX ON n8n_docs USING ivfflat (embedding vector_cosine_ops);
   CREATE INDEX ON n8n_chat_history (session_id);
   ```

4. Create similarity search function:
   ```sql
   CREATE OR REPLACE FUNCTION match_documents (
     query_embedding vector(768),
     match_count int DEFAULT null,
     filter jsonb DEFAULT '{}'
   ) RETURNS TABLE (
     id bigint,
     content text,
     metadata jsonb,
     similarity float
   )
   LANGUAGE plpgsql
   AS $$
   BEGIN
     RETURN QUERY
     SELECT
       id,
       content,
       metadata,
       1 - (n8n_docs.embedding <=> query_embedding) as similarity
     FROM n8n_docs
     WHERE metadata @> filter
     ORDER BY n8n_docs.embedding <=> query_embedding
     LIMIT match_count;
   END;
   $$;
   ```
4. Copy your **Project URL** and **anon/public key** from Settings → API

### Step 2: Configure Credentials in n8n

In n8n → **Settings → Credentials**, create these:

#### GitHub API
```json
{
  "accessToken": "ghp_xxxxxxxxxxxx"
}
```

#### Google Gemini (PalM)
```json
{
  "apiKey": "AIzaSyxxxxxxxxxxxx"
}
```

#### Supabase
```json
{
  "supabaseUrl": "https://your-project.supabase.co",
  "supabaseKey": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

#### PostgreSQL
```json
{
  "host": "db.your-project.supabase.co",
  "database": "postgres",
  "user": "postgres",
  "password": "your-password",
  "port": 5432
}
```

### Step 3: Import Workflow

1. Open n8n
2. Click **Import from File**
3. Select `workflow.json` from this repo
4. Update credentials in each node to match yours

### Step 4: Activate Workflows

This repo contains **two workflows**:

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| **Docs Scraper** | Schedule (monthly) | Syncs n8n docs to vector store |
| **AI Chat Agent** | Webhook | Chat interface for users |

**Important:** Activate the **Docs Scraper** first to populate the vector store!

---

## 🔧 Workflow Overview

### Docs Scraper Workflow

```
Schedule Trigger (monthly)
    ↓
Execute SQL (TRUNCATE n8n_docs)
    ↓
HTTP Request (GitHub tree API)
    ↓
Code (filter .md files)
    ↓
Loop Over Items
    ↓
GitHub (get file content)
    ↓
Filter (check content not empty)
    ↓
Default Data Loader
    ↓
Embeddings Google Gemini
    ↓
Supabase Vector Store
```

### AI Chat Agent Workflow

```
Chat Trigger (webhook)
    ↓
AI Agent (Gemini)
    ↓
├── Google Gemini Chat Model
├── Postgres Chat Memory
└── Supabase Vector Store (RAG tool)
```

---

## 💬 Usage

### Chat Interface

1. Copy the **Chat Webhook URL** from the `When chat message received` node
2. Embed in your website or use via API:
   ```bash
   curl -X POST "YOUR_WEBHOOK_URL" \
     -H "Content-Type: application/json" \
     -d '{"message": "How do I set up a webhook trigger?"}'
   ```

### Example Questions

- "How do I configure a PostgreSQL node?"
- "What are the best practices for error handling?"
- "How to use the Code node?"
- "Explain AI Agent configuration"

---

## ⚙️ Configuration Options

### Adjusting RAG Results

In `Supabase Vector Store` node (for chat):

| Parameter | Default | Description |
|-----------|--------|-------------|
| `topK` | 5 | Number of documents to retrieve |

### Update Frequency

In `Schedule Trigger` node:

| Value | Frequency |
|-------|-----------|
| `1` | Every month |
| `7` | Weekly |
| `30` | Daily |

### Embedding Model

Current: `models/gemini-embedding-001`

To change, edit both `Embeddings Google Gemini` nodes.

---

## 🐛 Troubleshooting

### Empty responses from AI
- Run the **Docs Scraper** workflow manually first
- Check Supabase table `n8n_docs` has records

### GitHub rate limits
- Add more GitHub tokens
- Or reduce update frequency

### Embedding errors
- Verify Google Gemini API key has embedding permissions
- Check API quota in Google AI Studio

### Database connection issues
- Verify PostgreSQL credentials
- Check Supabase project is not paused

---

## 📁 Project Structure

```
n8n-rag-ai-agent/
├── README.md              # This file
├── workflow.json          # n8n workflow (import this)
├── LICENSE                # MIT License
└── docs/
    └── SETUP.md           # Detailed setup guide
```

---

## 🤝 Contributing

Contributions welcome! Please:

1. Fork the repo
2. Create a feature branch
3. Submit a PR

---

## 📄 License

MIT License - feel free to use in personal and commercial projects.

---

## 🙏 Credits

- [n8n](https://n8n.io/) - Workflow automation platform
- [Supabase](https://supabase.com/) - Vector database
- [Google Gemini](https://ai.google.dev/) - AI/Embedding models

---

<p align="center">
  <strong>Star this repo if you find it useful!</strong>
</p>
