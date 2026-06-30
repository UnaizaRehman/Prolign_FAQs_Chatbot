# ProLign FAQ Chatbot

A Retrieval-Augmented Generation (RAG) chatbot for **ProLign**, an AI-powered university mentorship platform. The bot answers student/mentor questions about onboarding, mentor matching, booking, payments, sessions, and career prep — and automatically escalates unresolved questions or complaints to a human support team.

Built with **n8n**, **Supabase (pgvector)**, **Jina AI embeddings**, and **Groq-hosted LLMs**.

## How It Works

The system is split into two n8n flows packaged in [`N8n_chatbot.json`](./N8n_chatbot.json):

### 1. FAQ Ingestion (knowledge base setup)
A form trigger accepts a JSON file of FAQ records (see [`faqs_seed.json`](./faqs_seed.json)), then for each record:
1. **Extract from File** — parses the uploaded JSON
2. **Parse JSON to FAQ Records** — normalizes each entry into a `question` / `answer` / `category` / `tags` / `content` shape, where `content` combines the Q&A pair into a single string for embedding
3. **Loop Over Items** — processes records one at a time
4. **HTTP Request (Jina Embedding)** — generates a vector embedding for each FAQ's content using `jina-embeddings-v2-base-en`
5. **Create a row (Supabase)** — stores the question, answer, category, tags, embedding, and metadata in a Supabase `faqs` table (pgvector-backed)

### 2. Live Chat Handling (runtime query flow)
A webhook receives incoming chat messages and routes them through:
1. **Extract Message / Load History** — pulls the user's message and recent conversation context
2. **Embed User Query** — embeds the incoming question via the same Jina model
3. **Is it a Complaint?** — an AI Agent (Groq-hosted LLM) classifies intent as FAQ vs. complaint
   - **Complaint path** → escalates directly to Slack and responds with an acknowledgment
   - **FAQ path** → continues to retrieval
4. **Build RAG Context** — performs a vector similarity search against the Supabase `faqs` table to pull the most relevant FAQ entries
5. **AI Agent (Groq)** — generates a grounded answer using the retrieved FAQ context
6. **Should Escalate?** — if the agent has low confidence or can't answer from the retrieved context, the conversation is escalated to Slack for human follow-up; otherwise the generated answer is returned to the user

```
User Question
     │
     ▼
 Embed Query (Jina) ──► Intent Check (Groq)
                              │
                ┌─────────────┴─────────────┐
                ▼                           ▼
           Complaint                      FAQ
                │                           │
        Slack Escalation          Vector Search (Supabase)
                │                           │
         Ack Response                Generate Answer (Groq)
                                            │
                              ┌─────────────┴─────────────┐
                              ▼                           ▼
                       Confident Answer            Escalate to Slack
                              │                           │
                       Respond to User             Respond + Notify
```

## Knowledge Base

[`faqs_seed.json`](./faqs_seed.json) contains 30 seed FAQ entries covering:

| Category    | Count | Examples |
|-------------|-------|----------|
| Platform    | 8     | Registration, password reset, profile updates, pricing |
| Booking     | 5     | Scheduling sessions, cancellations, reminders |
| Career      | 5     | Resume building, interview prep, DSA, placement tips |
| Payments    | 4     | Stripe checkout, refunds, mentor payouts |
| Sessions    | 3     | Joining video calls, troubleshooting, no-shows |
| Onboarding  | 2     | Post-signup flow, AI assessment interview |
| Feedback    | 2     | Rating mentors, viewing reviews |
| Matching    | 1     | How the mentor-matching algorithm works |

Each entry includes a `question`, `answer`, `category`, `tags` array (for keyword support alongside vector search), and a `helpful_count` for future feedback-driven ranking.

## Tech Stack

- **Workflow orchestration:** [n8n](https://n8n.io)
- **Vector store:** [Supabase](https://supabase.com) with `pgvector`
- **Embeddings:** [Jina AI](https://jina.ai) (`jina-embeddings-v2-base-en`)
- **LLM inference:** [Groq](https://groq.com) (Llama 3.3 70B / Qwen3 32B)
- **Escalation:** Slack

## Setup

1. **Import the workflow** into n8n: `N8n_chatbot.json`
2. **Provision Supabase**: create a `faqs` table with columns for `question`, `answer`, `category`, `tags`, `helpful_count`, `source`, `chunk_number`, and a `vector` column (pgvector extension enabled) for embeddings
3. **Configure credentials** in n8n for:
   - Jina AI API (embeddings)
   - Groq API (chat models)
   - Supabase (database connection)
   - Slack (escalation channel)
4. **Seed the knowledge base**: trigger the "FAQ Ingestion" form in n8n and upload `faqs_seed.json`
5. **Activate the live chat webhook** and connect it to your front-end chat widget (e.g. ProLign's web app)

**Security note:** Before publishing this workflow publicly, double-check that no API keys are hardcoded in the HTTP Request node parameters. Use n8n's credential manager instead of inline `Authorization` headers, and rotate any keys that may have been exposed.

## Roadmap

- [ ] Feedback loop using `helpful_count` to re-rank FAQ retrieval results
- [ ] Multi-turn conversation memory improvements
- [ ] Analytics dashboard for unanswered/escalated queries
- [ ] Support for Urdu-language queries

## License

Internal project — Nespon Solutions / ProLign (FAST National University Karachi).
