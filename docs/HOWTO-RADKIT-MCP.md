# ⚙️ The official RADKit MCP server

All these workflows make use of the Cisco RADKit official MCP server. This server connects to your RADKit service and uses the [Client API](https://radkit.cisco.com/docs/client_api/index.html) to enable AI-driven operations across your device inventory.

This MCP server exposes a single tool: __radkit_eval__. Its purpose is to generate and execute Python code based on the Client API, which optimizes how your agent interacts with your RADKit service. Instead of exposing one tool per activity (retrieve inventory, execute commands, and so on), the MCP server can generate a Python snippet that dynamically uses Client API features such as filters and many-to-many command execution across devices.

## 📦 Standalone deployment of the MCP server

To deploy and test your RADKit MCP server stand-alone (no Docker), follow these steps:

Ensure the following are installed on your host system:

- Python 3.12+
- uv

From repository root:

```bash
cd radkit-mcp
```

Create and activate a virtual environment:

```bash
uv venv -p 3.13
source .venv/bin/activate
```

Install the dependencies:

```bash
uv sync
```

Create your local service config from the example:

```bash
cp service.config.direct.example service.config.direct
```

Then edit your `service.config.direct` file with your RADKit connection values:

```json
{
  "type": "direct",
  "username": "your_user@your_domain.com",
  "password_base64": "your_E2EE_password_from_the_webui_in_base64",
  "host": "your_radkit_host",
  "rpc_port": your_radkit_rpc_port
}
```

> ⚠️ Your RADKit service needs to be enabled with RPC and have a RPC port assigned. This can be checked by navigating in your service's Web UI to `Settings > service.connectivity.port_direct_rpc`. Default is **8181**

> ⚠️ The **E2EE password** for your remote user can be found in the user details in the Web UI: `Remote Users > <your user> > E2EE validation token`. **It is NOT encoded in base64** by default. Encode it before adding it to the file.

Once the file details are populated, start the RADKit MCP server with the following command:

```bash
uv run --no-sync radkit-llm mcp-server \
  --port 8082 \
  --sandbox subprocess \
  --service-config service.config.direct \
  --profile radkit-mcp-profile.py \
  --prompt radkit-mcp-prompt.txt \
  --auto-approve \
  --mcp-log-dir logs/
```

After running this command, your MCP server should be available at `http://localhost:8082/mcp`. To use it in your n8n workflows, update the `MCP Client` block found on each with this URL.

## 🐳 Docker-based deployment of the MCP server

There is a [docker-compose](docker-compose.yml) file available in this repository to launch the MCP server inside of a container.

This Docker Compose contains the following services:

| Service | Description |
|---------|-------------|
| 🎯 **n8n** | Low-code workflow automation platform serving as the main orchestration engine |
| 🌐 **cloudflare-tunnel** | Cloudflare tunnel service for secure remote access to n8n and webhooks for Webex and Slack |
| 🤖 **radkit-mcp-full** | RADKit MCP server with full access mode for complete network operations |
| 👁️ **radkit-mcp-readonly** | RADKit MCP server in read-only mode for safe information gathering |

First of all, prepare your virtual environment:

```bash
# From repository root
cp .env.example .env
```

Update `.env` with at least the following:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `CLOUDFLARE_TUNNEL_TOKEN` | **Required** - Your Cloudflare tunnel token for remote access | (none) |
| `N8N_PUBLIC_URL` | **Required** - Public URL for n8n webhooks (e.g., `https://n8n.your-domain.example`) | (none) |
| `RADKIT_SERVICE_CONFIG` | Path to RADKit service config file | `radkit-mcp/service.config.direct` |
| `RADKIT_PROFILE` | Path to RADKit profile script | `radkit-mcp/radkit-mcp-profile.py` |
| `RADKIT_PROMPT_FILE` | Path to RADKit prompt file | `radkit-mcp/radkit-mcp-prompt.txt` |
| `RADKIT_ACCESS_MODE` | Full-access instance mode | `full` |
| `RADKIT2_ACCESS_MODE` | Read-only instance mode (same options as above) | `read` |

Now, start all the services:

```bash
docker compose up -d --build
docker compose ps
```

Expected result: four running services available in the following URLs:

| Service | Endpoint |
|---|---|
| 🎯 n8n | `https://n8n.your-domain.example` |
| 🤖 RADKit MCP - Full Access | `http://host.docker.internal:8082/mcp` |
| 👁️ RADKit MCP - Read-only Access | `http://host.docker.internal:8083/mcp` |


### 📚 About the `--profile` and `--prompt` MCP server options

The official RADKit MCP server supports capability expansion in both request processing and LLM behavior through profiles and prompts.

A **profile** is a Python script executed at sandbox startup (as documented by `radkit-llm mcp-server --help`).

Its purpose is to inject custom Python functions and policy controls into the runtime scope used by `radkit_eval`.

In this repository, [radkit-mcp-profile.py](./radkit-mcp/radkit-mcp-profile.py) is used to enforce command governance:

- It wraps `execute_cli_commands` with a prefix-based allow list.
- It enforces behavior based on `RADKIT_ACCESS_MODE` (`show`, `full`, `conf`, `debug`).
- It blocks disallowed commands before they are sent to target devices.

A **prompt** file (`--prompt`) appends additional instruction text to the MCP server's built-in system prompt.

In this repository, [radkit-mcp-prompt.txt](./radkit-mcp/radkit-mcp-prompt.txt) provides execution guidance for `radkit_eval`, including:

- Safety pre-checks (call `get_access_mode()` before command execution).
- Preferred helper usage patterns (single `execute_cli_commands(...)` call with full device/command sets).
- Network-specific selection guidance (for example, IOS_XE filtering) and HTTP helper usage for external APIs.
- Output-formatting expectations (clear formatting and summarization for large outputs).


### 🔧 Running n8n without Cloudflare

If you don't need remote access via Cloudflare tunnel, you can simplify the setup. Edit your `docker-compose.yml` and modify the `n8n` service environment variables:

```yaml
n8n:
  # ... other config ...
  environment:
    - TZ=Europe/Lisbon
    - GENERIC_TIMEZONE=Europe/Lisbon
    - N8N_USER_MANAGEMENT_DISABLED=true
    - EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
    - EXECUTIONS_DATA_SAVE_ON_ERROR=all
    - EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=true
    - NODE_EXTRA_CA_CERTS=/home/node/cisco-ca.pem
    
    # Simplified configuration for local access only
    - N8N_HOST=0.0.0.0
    - N8N_LISTEN_ADDRESS=0.0.0.0
    - N8N_PORT=5678
    - N8N_PROTOCOL=http
    # Remove WEBHOOK_URL and N8N_EDITOR_BASE_URL for local-only access
    # Remove CORS settings if not needed
```

Then simply remove or comment out the `cloudflare-tunnel` service in your `docker-compose.yml`:

```yaml
# cloudflare-tunnel:
#   image: cloudflare/cloudflared:2026.3.0
#   # ... rest of the service config commented out ...
```

Start just n8n with:

```bash
# Only n8n and the RADKit MCP servers (full mode and read-only mode)
docker compose up -d n8n radkit-mcp-full radkit-mcp-readonly
```

n8n will then be accessible at `http://localhost:5678` from your local machine.

The downside is that you won't be able to run the webhooks for the Webex nor Slack services.