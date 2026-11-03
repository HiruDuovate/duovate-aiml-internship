# Day 16 – Model Context Protocol (MCP): Customer Data Server

An **MCP server** built with the official Python SDK (**FastMCP**) that gives AI apps safe, read-only access to a customer database.
The same server was tested with a Python client, over **stdio** and **Streamable HTTP**, connected to **Claude Desktop**, and used from my own **agent harness** (Day 15).

> Part of my AI & ML internship task series (Day 16 of 20).

---

## What is MCP?

MCP (Model Context Protocol) is an open standard for connecting AI applications to tools and data, like **USB-C for AI**:
build a tool server **once**, and **any** MCP-compatible app (Claude Desktop, Claude Code, VS Code, Cursor, or your own agent) can discover and use it.

```
Host (Claude Desktop / my harness)
  ├── MCP client ──► MCP server: customer-data   ← this repo
  └── MCP client ──► other MCP servers (files, GitHub, ...)
```

| Primitive | Controlled by | In this server |
|---|---|---|
| **Tools** | the model (it decides to call them) | `describe_data`, `query_customers`, `count_by` |
| **Resources** | the app / user (attached as context) | `schema://customers` |
| **Prompts** | the user (ready-made templates) | `analyse_customers(question)` |

| Transport | How it works | Use it for |
|---|---|---|
| **stdio** | The host starts the server as a local process and talks through stdin / stdout | Local tools in Claude Desktop / VS Code |
| **Streamable HTTP** | The server runs as a web service at a URL (`/mcp`) | Shared or remote servers (needs authentication in production) |

All messages use **JSON-RPC 2.0**.

---

## Repository contents

| File | What it is |
|---|---|
| `server.py` | The MCP server (FastMCP): 3 tools, 1 resource, 1 prompt |
| `customers.db` | SQLite database with **300 synthetic customers** (no real people) |
| `poisoned_server.py` | **Test only.** A deliberately malicious server used to test tool-poisoning defences. Do not connect it to a real app. |
| `day16_mcp.ipynb` | Colab notebook: fundamentals, server, tests, harness integration, security |
| `Day16_Report_MCP.pdf` | Full report with results and screenshots |
| `screenshots/` | Claude Desktop screenshots |

### The data
`customers` table: `customer_id`, `city`, `age`, `plan` (Basic / Standard / Premium), `months_active`, `monthly_spend_lkr`, `churned`, `notes`.
All rows are **synthetic**. One row (`C0042`) contains a planted prompt-injection note on purpose, for the security test.

---

## Quick start

**Requirements:** Python 3.10+, and the MCP SDK **v1** (FastMCP was renamed to `MCPServer` in v2).

```bash
pip install "mcp<2"
```

### Run the server
```bash
python server.py              # stdio (for Claude Desktop / VS Code / Inspector)
python server.py http 8000    # Streamable HTTP at http://127.0.0.1:8000/mcp
```
`server.py` reads the database from the `CUSTOMER_DB` environment variable, or from `customers.db` in the same folder.

### Test with the MCP Inspector (needs Node.js)
```bash
npx @modelcontextprotocol/inspector python server.py
```
Open the link it prints, click **Connect**, then try **Tools**, **Resources** and **Prompts**.

### Connect to Claude Desktop
**Settings → Developer → Edit Config**, then add an `mcpServers` entry **next to** the settings already in the file
(do not delete the existing content). Replace the paths with your own:

```json
{
  "mcpServers": {
    "customer-data": {
      "command": "C:\\path\\to\\python.exe",
      "args": ["C:\\path\\to\\server.py"],
      "env": { "CUSTOMER_DB": "C:\\path\\to\\customers.db" }
    }
  }
}
```
Quit Claude Desktop completely, reopen it, and check **Settings → Developer** shows `customer-data` as **Running**.
Then in a new chat ask: *"Using the customer-data tools, which subscription plan has the highest churn rate?"*

### Connect to VS Code
Create `.vscode/mcp.json`:
```json
{
  "servers": {
    "customer-data": { "type": "stdio", "command": "python", "args": ["C:\\path\\to\\server.py"] }
  }
}
```
Then use Copilot Chat in **Agent** mode.

---

## Results

**Claude Desktop (real conversation)**: Claude discovered the tools, wrote its own SQL, called `query_customers`, and answered:

| Plan | Customers | Churned | Churn rate (via MCP) | Direct SQL check |
|---|---|---|---|---|
| Standard | 106 | 38 | 35.8% | 35.8% ✅ |
| Basic | 155 | 49 | 31.6% | 31.6% ✅ |
| Premium | 39 | 6 | 15.4% | 15.4% ✅ |

**Other tests**

| Test | Result |
|---|---|
| Python client over stdio | 3 tools, 1 resource, 1 prompt discovered and called |
| Streamable HTTP | Same tools, `count_by(city)` in 0.08 s |
| `DELETE FROM customers` / `SELECT 1; DROP TABLE ...` / bad column name | All blocked by the server |
| Claude Desktop: "delete all customers" | Refused (tools are read-only) |
| My harness + poisoned server | Poisoned tool **rejected** by the description scanner |
| My harness + injection note in row C0042 | Reported as data; `send_email` **never called** |

Screenshots:

![Server running in Claude Desktop](screenshots/claude_desktop_running.png)
![Real tool call: SQL request and JSON response](screenshots/claude_desktop_tool_call.png)
![Delete request refused](screenshots/claude_desktop_security.png)

---

## Security

| Risk | Mitigation |
|---|---|
| **Prompt injection** (instructions hidden in data) | Harness rule: tool output is data, never instructions; sensitive tools need human approval; default policy *deny* |
| **Tool poisoning** (instructions hidden in a tool description) | Description scanner + tool allowlist in the harness; review a server before connecting it |
| **Over-broad permissions** | Database opened **read-only**; single `SELECT` statements only; column names validated; max 200 rows |
| **Data leaking out** | No network tools on the server; `send_email` in the harness is sensitive and simulated |
| **Untrusted servers** | Only run servers you have read; tool names are prefixed by server (`customers__...`) |
| **Remote HTTP servers** | Bound to `127.0.0.1`; production needs OAuth authentication and HTTPS |

These layers form **defence in depth**: the model, the tool description, the server code and the read-only database each block a harmful action.

**Never commit:** API keys or `.env` files, your real `claude_desktop_config.json` (it contains personal paths and account IDs),
Inspector session tokens, logs, or real customer data. See `.gitignore`.

---

## Built with
- [Model Context Protocol](https://modelcontextprotocol.io) – Python SDK (FastMCP, `mcp` 1.x)
- SQLite, pandas
- Google Colab, Gemini (for the agent harness)
