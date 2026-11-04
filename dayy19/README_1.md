# 🤝 Customer Retention Copilot

An agentic AI web app that helps retention managers answer three questions in one place:
**who is leaving, how do customers feel, and what are we allowed to offer them?**

It combines a customer database (through an **MCP server**), my **Day 06 emotion model**, the company's **policy documents (RAG)**,
**memory** and a **team of four agents**, with guardrails and tracing.

> Capstone of my AI & ML internship (Day 19 of 20). All customer data is **synthetic**: no real people.

![Architecture](docs/architecture.png)

---

## What it can do

| Ask | Example | Uses |
|---|---|---|
| Customer numbers | *"Which plan has the highest churn rate?"* | MCP database (read-only SQL) |
| Customer feelings | *"How do customer C0007's messages sound?"* | Day 06 emotion model |
| Company rules | *"What is the maximum discount without approval?"* | RAG over policy documents, with `[source]` |
| All three together | *"Customer C0015: plan, mood, and what should we offer?"* | Database + emotion model + RAG |
| Remember things | *"Remember that I manage the Kandy region."* then *"How many customers are in my region?"* | Memory |
| Take an action | *"Create a high-priority ticket for C0007."* | `create_ticket`, **only with human approval** |

It **cannot** change or delete data (the database is read-only), send emails, or invent data that is not there.

---

## How it works

| Part | Technology | Learned on |
|---|---|---|
| Multi-agent orchestration | Planner → Researcher → Writer → Critic (LangGraph), max 1 revision | Day 17 |
| Tool calling | Researcher tool loop in my own harness (step/token budgets, timeouts, approval policy) | Days 14–15 |
| MCP server | `server.py` (FastMCP): customers + support messages, read-only, `SELECT` only | Day 16 |
| RAG | Policy documents → MiniLM embeddings → top sections with citations | Day 13 |
| My trained model as a tool | Emotion classifier (TF-IDF + Logistic Regression, 6 emotions), **test accuracy 86.3%, macro F1 0.80** on `dair-ai/emotion` | Day 06 |
| Memory | Per-user facts in `data/memory.json` | Day 15 |
| Guardrails | Input filter (incl. base64), tool allow-list, read-only DB, tool-output sanitiser, output filter, MCP description scanner | Day 18 |
| Observability | OpenTelemetry spans for every agent, LLM call and tool call | Day 18 |
| UI | Gradio chat with live agent steps | Day 19 |
| LLM | Groq `openai/gpt-oss-120b` (free tier), with Gemini / `gpt-oss-20b` as fallbacks | |

**One codebase:** in the notebook, every cell marked `%%core` runs **and** is saved into `core.py`, so the notebook and the app run exactly the same code.

---

## Setup

You need a free API key from [Groq](https://console.groq.com/keys) (and optionally [Gemini](https://aistudio.google.com/apikey)).

### Option 1: Google Colab (easiest)
1. Open `day19_capstone.ipynb` in Colab.
2. In **🔑 Secrets**, add `GROQ_API_KEY` (and optionally `GEMINI_API_KEY`) and turn **Notebook access** on.
3. **Runtime → Run all**. Section 8 prints a public `gradio.live` link to the app.

### Option 2: on your computer (Python 3.10+)
```bash
pip install -r requirements.txt
cp .env.example .env          # Windows: copy .env.example .env  -> then add your key(s) to .env
python app.py                 # open http://localhost:7860
```

### Option 3: Docker
```bash
docker build -t retention-copilot .
docker run -p 7860:7860 --env-file .env retention-copilot     # open http://localhost:7860
```

| Variable | Meaning |
|---|---|
| `GROQ_API_KEY` / `GEMINI_API_KEY` | LLM keys (at least one). The app falls back between models |
| `PORT` | Web port (default 7860) |
| `SHARE` | `1` = also create a public Gradio link |
| `EMOTION_MODEL` | Path to another emotion model (scikit-learn pipeline that takes raw text) |

> 🔒 Never commit `.env` or API keys. They are excluded in `.gitignore`.

---

## Project structure
```
├── app.py                 # starts the web app
├── core.py                # all app logic (generated from the notebook's %%core cells)
├── server.py              # MCP server (read-only database)
├── day19_capstone.ipynb   # build + evaluation notebook
├── data/
│   ├── customers.db       # synthetic customers (300) + support messages (370)
│   ├── knowledge_base/    # 5 policy documents for RAG (1 is a deliberately untrusted forum post)
│   └── emotion_model.joblib
├── results/               # evaluation and red-team results
├── docs/architecture.png
├── requirements.txt, Dockerfile, .env.example
```

---

## Evaluation (Day 18 suite, run on the capstone)

The suite has **20 tasks** (database, emotion, policy, combined, unanswerable, memory) with ground truth from SQL, from running the
emotion model directly, or from the policy text; plus an LLM judge on a different model.

**First run (Nov 2026):** the free-tier daily quota ran out after **8 of 20 tasks**, so these results are partial.

| Result (8 tasks run) | Value |
|---|---|
| Correct answers (LLM judge) | 6 / 8 |
| Correct answers (automatic check) | 5 / 8 (D3 is correct but rounded to whole %) |
| Failed tasks | 2 (D7, E1), both because the LLM API failed, not because of a wrong answer |
| Guardrail attacks tested | **6 / 6 blocked** (jailbreaks, secret theft, data exfiltration, base64-hidden, MCP tool poisoning); 6 not yet tested |

**What the traces showed:** the Researcher made **92 LLM calls for 10 questions** but only **10 tool calls**, and ran out of steps
6 times. The model often answered in its own tool-call format (`{"name": ..., "arguments": ...}`) instead of ours, so those replies
were rejected and the steps were wasted. That also used up the daily quota (about 11,000 tokens per task, 1,310 s waiting for rate limits).
**Fix added:** the harness now accepts that format, recovers rejected native tool calls, and logs invalid replies in the traces.
The full 20-task run with the fix is next.

---

## Limitations
- Regex guardrails can be bypassed by re-worded or translated attacks; the model is the last line of defence.
- The emotion model was trained on short English tweets; other languages are not supported.
- Free-tier rate limits make answers slow and can stop a run for the day.
- Synthetic data only. Real customer data would need anonymisation and a data-processing agreement with the LLM provider.

## Built with
[LangGraph](https://langchain-ai.github.io/langgraph/) · [Model Context Protocol](https://modelcontextprotocol.io) ·
[Gradio](https://www.gradio.app) · [OpenTelemetry](https://opentelemetry.io) · [sentence-transformers](https://www.sbert.net) ·
scikit-learn · [Groq](https://console.groq.com)
