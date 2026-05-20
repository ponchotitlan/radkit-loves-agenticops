# ⚙️ Enhanced WxT ChatOps for my RADKit Workflow (n8n + Webex Teams)

<div align="center">
<img src="../images/chatops_wxt_workflow_read.png"/>
<img src="../images/chatops_wxt_workflow_write.png"/>
</div>

A practical **low-code, agentic ChatOps workflow** built in **n8n** for safe network automation via **Webex Teams**.  
Uses **multiple specialized agents**, structured guardrails, and real device interaction through the **official Cisco RADKit MCP server**.

---

## ✨ What this gives you

- 💬 Operate your network from Webex Teams (ChatOps)
- 🧠 Agentic architecture (separation of responsibilities)
- 🕵️ Device validation gate before any execution
- 🛡️ Guardrails for risky operations
- 👤 Human approval via Webex Adaptive Cards before changes
- 🔁 Rollback-aware execution
- 🌐 Publicly reachable webhooks via Cloudflare Tunnel (required by Webex)
- 🧩 Fully self-hosted (n8n + Anthropic Claude + MCP)

---

## 🧠 Architecture at a glance

| Agent | Responsibility | Can Execute Writes? | MCP Mode |
|------|----------------|----------------------|----------|
| Planning AI Agent | Classifies `read` vs `commit`, generates plan + risk assessment | ❌ | Read-only |
| Commit And Verify AI Agent | Validates approved changes and applies configuration | ✅ (after user approval) | Full |

**Execution separation:**
- All **pre-approval logic** runs in read-only MCP mode (`http://host.docker.internal:8083/mcp`)
- Only **Commit Agent** accesses full MCP mode after Adaptive Card confirmation (`http://host.docker.internal:8082/mcp`)
- Memory buffers maintain conversation context per thread/room ID

This ensures **no single agent can execute changes without human approval**.

---

## 🔐 Safety & Guardrails

Built-in by design:

- **Dual MCP endpoints**: Read-only and full modes enforce execution boundaries
- **Intent classification**: Structured JSON extraction (`read` vs `commit` intent)
- **Planning-only pre-approval**: All agents generate plans before any write operation
- **Memory-based session handling**: Per-thread context tracking with LangChain buffer window
- **No configuration executed without**:
  - Device validation pass
  - Generated plan with safety verdict
  - Risk assessment
  - Rollback strategy
  - Explicit Webex Adaptive Card button confirmation from user
- **Card interaction validation**: Separate webhook for `attachmentActions` ensures intent confirmation
- **JSON contracts**: Strict schema validation between workflow nodes

Result: **predictable, inspectable, auditable automation with clear human-in-the-loop boundaries**.

---

## 🔄 End-to-end flow

1. User mentions the bot in a Webex Teams space
2. `When the RADKeteer bot is mentioned` webhook triggered: message content fetched via Webex API
3. "Please Standby" random phrase sent to user
4. Full message retrieved and passed to **Planning AI Agent**
5. Planning Agent classifies intent (`read` vs `commit`) via LLM + read-only MCP
6. Planning Agent returns structured JSON: `{ mode, reply_markdown, cards[] }`
7. JSON transformation validates and normalizes response
8. Switch by mode:
   - `read` → reply posted to thread immediately (no card)
   - `commit` → Adaptive Card with plan, risk level, and action buttons posted to thread
9. User clicks button on card:
   - ❌ Cancel → card deleted, "cancelled" reply posted
   - ✅ Confirm → standby message posted, card deleted
10. **Commit And Verify AI Agent** executes via full MCP mode
11. Agent applies configuration, re-fetches state, verifies result
12. Verification markdown posted back to original thread

---

## 🛠️ Setup

### 1. Cloudflare Tunnel and Cisco RADKit MCP servers containers

Webex Teams webhooks require a **publicly reachable HTTPS endpoint**. This project includes a Docker Compose service for Cloudflare tunnel + custom domain options for n8n.

The Cloudflare flow looks like this:

```
n8n.yourdomain.com  →  Cloudflare Tunnel  →  n8n (container, port 5678)
```

To set this up along with the Cisco RADKit MCP servers as containers, check [this guide](../../docs/HOWTO-RADKIT-MCP.md#-docker-based-deployment-of-the-mcp-server).

### 2. Webex Setup
✅💬 See [this frustration-free guide!](../../docs/WEBEX-SETUP.md)

### 3. n8n workflow import
1. Navigate to your n8n instance on a web browser
2. Create a new workflow
3. Import the file [WxT ChatOps for my RADKit.json](WxT%20ChatOps%20for%20my%20RADKit.json) included in this repository

---

