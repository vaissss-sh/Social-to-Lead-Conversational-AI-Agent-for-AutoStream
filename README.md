# Social-to-Lead-Conversational-AI-Agent-for-AutoStream

# AutoStream AI Agent

A **LangGraph-powered conversational sales agent** for AutoStream — a fictional SaaS platform for automated video editing. Built for the ServiceHive / Inflx ML Intern assignment.

---

## Features

- 🧠 **Intent classification** — greeting, product inquiry, or high-intent lead
- 📚 **RAG pipeline** — ChromaDB vector store over a local Markdown knowledge base
- 🔧 **Tool calling** — `mock_lead_capture()` fires only after all three lead fields are collected
- 💬 **State persistence** — full conversation history retained across turns via LangGraph state

---

## Project Structure

```
autostream_agent/
├── knowledge_base.md   # Pricing, plans, and company policies
├── rag.py              # ChromaDB + sentence-transformers RAG pipeline
├── tools.py            # mock_lead_capture() tool
├── agent.py            # LangGraph graph definition (nodes + routing)
├── main.py             # CLI runner
├── requirements.txt
└── README.md
```

---

## How to Run Locally

### 1. Clone / download the project

```bash
git clone <your-repo-url>
cd autostream_agent
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

> **Note:** The first run downloads the `all-MiniLM-L6-v2` sentence-transformer model (~80 MB) used for embeddings.

### 3. Set your API key

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

### 4. Run the agent

```bash
python main.py
```

### Example conversation

```
You: Hi, what plans do you offer?
Agent: AutoStream offers two plans: Basic ($29/month) with 10 videos/month at 720p,
       and Pro ($79/month) with unlimited videos, 4K resolution, and AI captions.

You: That sounds great, I want to try the Pro plan for my YouTube channel.
Agent: Awesome! I'd love to get you set up. Could I get your full name?

You: My name is Alex Rivera
Agent: Great, Alex! What's your email address?

You: alex@example.com
Agent: Perfect! And I can see you're on YouTube — let me get you signed up...

✅  Lead captured: Alex Rivera | alex@example.com | Youtube
```

---

## Architecture (~200 words)

### Why LangGraph?

LangGraph was chosen because it models the agent as an **explicit state machine** rather than a free-running loop. Each conversation turn passes through a directed graph of nodes — `classify_intent → rag_respond | collect_lead → capture_lead` — making the control flow transparent, testable, and easy to extend. Unlike vanilla LangChain chains, LangGraph makes it trivial to add conditional routing (e.g., "only fire the lead tool when all three fields are populated") without convoluted if/else in application code.

### State Management

State is a typed dictionary (`AgentState`) that persists the full message history plus individual lead fields (`lead_name`, `lead_email`, `lead_platform`, `lead_captured`) across every turn. LangGraph's `add_messages` reducer appends new messages without overwriting history, giving the LLM full conversational context at each step. Routing functions inspect state to decide which node runs next — for instance, the agent resumes lead collection mid-conversation if a user drifts to a question after giving only their name.

### RAG Pipeline

The knowledge base (`knowledge_base.md`) is split into semantic chunks, embedded with `all-MiniLM-L6-v2` via ChromaDB, and retrieved via cosine similarity at each `rag_respond` call. This grounds every product answer in verified KB content.

---

## WhatsApp Deployment via Webhooks

To deploy this agent on WhatsApp, you would use the **WhatsApp Business Cloud API** (Meta) with a webhook integration:

### Architecture

```
WhatsApp User
     │  (sends message)
     ▼
Meta WhatsApp Cloud API
     │  (HTTP POST to your webhook URL)
     ▼
Your Web Server  (FastAPI / Flask)
     │
     ├── Verify webhook token (GET /webhook)
     │
     └── On POST /webhook:
            1. Parse incoming JSON → extract sender ID + message text
            2. Load or create per-user AgentState from a database (e.g. Redis / PostgreSQL)
            3. Call graph.invoke(state) with the new HumanMessage
            4. Persist updated state back to DB
            5. Send AI reply via Meta Send Message API (POST to graph.facebook.com)
```

### Key implementation points

| Concern | Solution |
|---|---|
| **Webhook verification** | Respond to GET with `hub.challenge` token (Meta requirement) |
| **Per-user state** | Store `AgentState` keyed by WhatsApp sender ID in Redis or a DB |
| **Async handling** | Use FastAPI + background tasks so webhook responds in < 5 s |
| **Sending replies** | `POST https://graph.facebook.com/v19.0/{phone_id}/messages` with Bearer token |
| **Lead capture confirmation** | After `mock_lead_capture`, send a WhatsApp template message |

A minimal FastAPI webhook handler would look like:

```python
@app.post("/webhook")
async def receive_message(payload: dict, background_tasks: BackgroundTasks):
    message = payload["entry"][0]["changes"][0]["value"]["messages"][0]
    sender  = message["from"]
    text    = message["text"]["body"]

    background_tasks.add_task(process_message, sender, text)
    return {"status": "ok"}
```

---

## Evaluation Checklist

| Criterion | Implementation |
|---|---|
| Intent detection | Zero-shot LLM classification in `classify_intent` node |
| RAG pipeline | ChromaDB + sentence-transformers in `rag.py` |
| State management | LangGraph `AgentState` TypedDict with `add_messages` |
| Tool calling | `mock_lead_capture` called only from `capture_lead` node, only after all fields collected |
| Code clarity | Typed state, docstrings, single-responsibility nodes |
| Deployability | Webhook section above; stateless graph + external state store |
