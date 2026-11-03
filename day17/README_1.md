# Day 17 – Multi-Agent Research Team (LangGraph + MCP + A2A)

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/YOUR-USERNAME/YOUR-REPO/blob/main/day17_multi_agent.ipynb)

A team of four AI agents that research a customer database together, plus a live **AI Agent Office** web app where you can watch the robots work.

> Part of my AI & ML internship task series (Day 17 of 20). Builds on Day 15 (agent harness) and Day 16 (MCP server).

![AI Agent Office](screenshots/agent_office.png)

---

## The team

| Agent | Job | Tools |
|---|---|---|
| 🧭 **Planner** (orchestrator) | Splits the goal into 2–3 research questions and hands them out | none |
| 🔍 **Researcher** | Answers each question with SQL on the customer database | Day 16 **MCP** tools (read-only) |
| ✍️ **Writer** | Writes a short report from the findings | none |
| 🧐 **Critic** | Checks the report; sends it back to the Writer once if something is wrong | none |

```
User ─► Planner ─(A2A)─► Researcher ─(MCP)─► customers.db
                              │
                              ▼
                           Writer ◄──► Critic ─► Final report
```

- **LangGraph** wires the agents as nodes in a graph with shared state.
- **MCP** (Day 16) connects the Researcher to the database: read-only, `SELECT` only.
- **A2A** (Agent2Agent, `a2a-sdk` 1.3): the Researcher also runs as an A2A server with an **Agent Card**; the Planner discovers it and delegates over JSON-RPC.

| | MCP | A2A |
|---|---|---|
| Connects | an agent to **tools / data** | an agent to **another agent** |
| Discovery | `tools/list` | Agent Card (`/.well-known/agent-card.json`) |

---

## Run it (Google Colab, CPU only)

1. Click **Open in Colab** above (or upload `day17_multi_agent.ipynb` to Colab).
2. Get a free API key at [console.groq.com/keys](https://console.groq.com/keys) (no card needed).
3. In Colab, open 🔑 **Secrets** (left bar) → **Add new secret**:
   - Name: `GROQ_API_KEY`
   - Value: your key (paste it **once**; it starts with `gsk_`)
   - Turn **Notebook access** ON
4. **Runtime → Run all**. The model cell should print:
   ```
   Backend: groq · openai/gpt-oss-120b · RPM limit 25
   ```
5. The last cell starts the **AI Agent Office** and prints two links:
   - 🔒 **Colab link**: opens fast, only works for you
   - 🌍 **Public link** (`trycloudflare.com`): share it with others; may take 1–2 minutes to start working

   Both links work **only while the Colab session is running**.

In the office: type a question (or click an example), tick **Reach the Researcher over A2A** if you want, and press **Start the team**.

> Never put the API key in the code, in a screenshot or on GitHub. Keep it only in Colab Secrets.

---

## Results

Model: `openai/gpt-oss-120b` on Groq (free tier). Data: 300 **synthetic** customers.

**Single agent vs multi-agent** (same 5 tasks, LLM judge with SQL ground truth, scores 1–5):

| System | Quality | Correct | Complete | Clear | Tokens | Seconds | LLM calls |
|---|---|---|---|---|---|---|---|
| Single agent | 3.13 | 2.6 | 3.4 | 3.4 | 4,940 | 28.3 | 5.0 |
| **Multi-agent** | **4.20** | 3.2 | 4.6 | 4.8 | 10,562 | 50.0 | 12.6 |

![Single vs multi](screenshots/single_vs_multi.png)

The team writes more complete and clearer reports, but costs about **2.1× the tokens** and **1.8× the time**.

**Fact check:** I recomputed the numbers with SQL. When the Researcher called the SQL tool the numbers were correct (e.g. Galle churn 40.5%).
When it answered **without** a tool call, it invented numbers (e.g. "Basic churn 12%", real 31.6%), and the Critic approved them because it only checks the draft against the findings.

**Lesson:** more agents do not make facts more correct. Every agent must be grounded in tool results, and the reviewer must check against the source data.

**Next fixes:** reject Researcher answers with no tool call; give the Critic the SQL rows; raise the Researcher step budget from 6 to 8.

Full details: [Day17_Report_Multi_Agent.pdf](Day17_Report_Multi_Agent.pdf)

---

## Repository contents

| File | What it is |
|---|---|
| `day17_multi_agent.ipynb` | Colab notebook: team, A2A, comparison, AI Agent Office |
| `Day17_Report_Multi_Agent.pdf` | Report with theory, results and fact check |
| `screenshots/` | Office and comparison chart |
| `.gitignore` | Keeps keys, logs, traces and the database out of Git |

## Built with
- [LangGraph](https://langchain-ai.github.io/langgraph/) – agent graph
- [Model Context Protocol](https://modelcontextprotocol.io) – Python SDK (`mcp` 1.x)
- [A2A Protocol](https://a2a-protocol.org) – `a2a-sdk` 1.3
- [Groq](https://console.groq.com) – free LLM API
- Starlette + Uvicorn – AI Agent Office web app; Cloudflare quick tunnel for the public link
