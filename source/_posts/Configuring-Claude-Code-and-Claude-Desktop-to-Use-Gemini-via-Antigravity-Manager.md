---
title: Configuring Claude Code and Claude Desktop to Use Gemini via Antigravity Manager
date: 2026-09-19 19:15:00
tags: [Claude, Gemini, Antigravity, LLM, Proxy, Windows, macOS, AI-Tools]
---

As large language models rapidly evolve, developers often subscribe to different model ecosystems for distinct strengths—such as combining a Google AI Ultra subscription with the refined developer experience of Anthropic's **Claude Code CLI** and **Claude Desktop App**.

However, integrating Claude-native clients with Google Gemini or Vertex AI backends presents two immediate challenges: protocol mismatch (Claude clients speak Anthropic Messages API, while Gemini speaks Google's native API) and strict client-side validation (Claude Desktop blocks non-loopback HTTP and rejects non-`claude-*` model identifiers).

This guide walks through configuring **Claude Code CLI** and **Claude Desktop App** on **Windows** and **macOS** to seamlessly route through **Antigravity Manager**, an account-pooling reverse proxy running on a local server/NAS, backed by **Gemini 3.8 Flash**.

---

## 1. What is Antigravity Manager?

**Antigravity Manager** (community Docker image `lbjlaq/antigravity-manager`, powered by `antigravity-tools`) is an account-pooling and protocol-translation gateway tailored for Google Antigravity, Google AI Ultra, and Vertex AI backends.

### Key Capabilities
1. **Multi-Account & Quota Pooling**: Centrally aggregates multiple Google accounts (including Google AI Ultra/One subscriptions, OAuth sessions, and Vertex AI service credentials). It handles automatic session keep-alive, quota distribution, and seamless failover.
2. **Standardized Protocol Bridge**: Translates Google backend responses into industry-standard **Anthropic Messages API (`/v1/messages`)** and **OpenAI API (`/v1/chat/completions`)** formats. This allows any client built exclusively for Claude or OpenAI to consume Gemini models without code modifications.
3. **Dynamic Model Aliasing (`custom_mapping`)**: Allows wildcard routing rules at the gateway level. Incoming requests for `claude*` or `claude-3-7-sonnet` can be dynamically mapped to `gemini-3.8-flash-high`, cleanly bypassing client-side model validation and mitigating upstream rate limits (HTTP 429).
4. **Web Administration Console**: Provides a built-in web dashboard (default port `8045`) to inspect health metrics, monitor query logs, configure model aliases, and set gateway access tokens.

> [!NOTE]
> **Understanding the Gateway API Key**  
> Google does not provide an "Anthropic API Key" for your Google account. In this setup, the API key entered into Claude Code or Claude Desktop is a **self-configured proxy token** (`proxy.api_key` in Antigravity Manager). It protects your gateway from unauthorized local network access.

---

## 2. Architecture & Core Constraints

![Architecture & Data Flow](https://files.seeusercontent.com/2026/09/19/x9yD/architecture_flowchart.png)

### Essential Constraints to Know
1. **Network Constraint**: Claude Desktop enforces a strict origin policy: `baseUrl: must use https (or http on loopback)`. Supplying a plain LAN IP address (such as `http://192.168.68.55:8045`) will be rejected by client validation. Forwarding to the local loopback (`http://127.0.0.1:8045`) satisfies this requirement.
2. **Model Identifier Constraint**: Claude Desktop uses a hardcoded regex pattern requiring all model routes to begin with `claude-*` or `anthropic/claude-*`. Manually entering custom model names like `gemini-3.8-flash-high` triggers validation warnings and disables the "Apply Changes" button. Transparent routing must instead be handled by the proxy's `custom_mapping`.

---

## 3. Step 1: Gateway & Network Configuration

### 3.1 Configure Model Mapping on the Gateway

To allow Claude clients requesting `claude` or `claude-3-7-sonnet` to be automatically served by Gemini 3.8 Flash without quota exhaustion, set up mapping rules on your server.

**Configuration file path**: `/home/<user>/.antigravity_tools/gui_config.json`

Add or update the `custom_mapping` section under `proxy`:

```json
{
  "proxy": {
    "custom_mapping": {
      "claude*": "gemini-3.8-flash-high",
      "claude": "gemini-3.8-flash-high",
      "claude-3-5-sonnet-*": "gemini-3.8-flash-high",
      "claude-3-7-sonnet-*": "gemini-3.8-flash-high",
      "claude-3-opus-*": "gemini-3.8-flash-high",
      "claude-opus-4-*": "gemini-3.8-flash-high",
      "claude-sonnet*": "gemini-3.8-flash-high",
      "claude-haiku-*": "gemini-3.8-flash-high"
    }
  }
}
```

Restart the container to apply the changes:
```bash
sudo docker restart antigravity-manager
```
*(Alternatively, navigate to `http://<SERVER_IP>:8045` in your browser and update the mapping via **Proxy Settings** → **Custom Model Mapping**.)*

---

### 3.2 Retrieve Your Gateway API Key

Your clients require an authorization key (`<ANTIGRAVITY_MANAGER_API_KEY>`) to authenticate against Antigravity Manager:
- **Via Web UI**: Open `http://<SERVER_IP>:8045`, go to **Proxy Settings**, and copy or set the **API Key / Access Token**.
- **Via Configuration File**: Inspect the `"api_key"` property inside `gui_config.json`:
  ```json
  {
    "proxy": {
      "api_key": "your_custom_proxy_key_here"
    }
  }
  ```

---

### 3.3 Set Up Local Loopback Port Forwarding

Forward port `8045` on your server to `127.0.0.1:8045` on your local workstation.

#### Windows (Native Portproxy, Persistent Across Reboots)
Open **PowerShell as Administrator** and execute:

```powershell
netsh interface portproxy add v4tov4 listenport=8045 listenaddress=127.0.0.1 connectport=8045 connectaddress=192.168.68.55
```

- **Verify connection**:
  ```powershell
  netsh interface portproxy show all
  Test-NetConnection -ComputerName 127.0.0.1 -Port 8045
  ```
- **Remove rule (if needed later)**:
  ```powershell
  netsh interface portproxy delete v4tov4 listenport=8045 listenaddress=127.0.0.1
  ```

#### macOS (SSH Local Tunnel)
- **Interactive Session**:
  ```bash
  ssh -N -L 8045:127.0.0.1:8045 <server_ssh_host>
  ```
- **Background Daemon (LaunchAgent)**:
  Create `~/Library/LaunchAgents/com.user.antigravity-tunnel.plist`:
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
  <plist version="1.0">
  <dict>
      <key>Label</key>
      <string>com.user.antigravity-tunnel</string>
      <key>ProgramArguments</key>
      <array>
          <string>/usr/bin/ssh</string>
          <string>-N</string>
          <string>-L</string>
          <string>8045:127.0.0.1:8045</string>
          <string><server_ssh_host></string>
      </array>
      <key>RunAtLoad</key>
      <true/>
      <key>KeepAlive</key>
      <true/>
  </dict>
  </plist>
  ```
  Load and start the service:
  ```bash
  launchctl load ~/Library/LaunchAgents/com.user.antigravity-tunnel.plist
  ```

---

## 4. Configuring Claude Code CLI

> [!WARNING]
> Claude Code prioritizes its `settings.json` environment block over current shell environment variables. Setting temporary shell exports (`export ANTHROPIC_BASE_URL=...`) will be overridden by the config file. Modify the file directly.

### 4.1 Configuration File Setup
- **Windows**: `C:\Users\<YourUsername>\.claude\settings.json`
- **macOS / Linux**: `~/.claude/settings.json`

Update the `env` section (replacing `<ANTIGRAVITY_MANAGER_API_KEY>` with your gateway key):

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:8045",
    "ANTHROPIC_AUTH_TOKEN": "<ANTIGRAVITY_MANAGER_API_KEY>",
    "ANTHROPIC_MODEL": "gemini-3.8-flash-high",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "gemini-3.8-flash-high",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "gemini-3.8-flash-high",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "gemini-3.8-flash-high"
  },
  "permissions": {
    "defaultMode": "bypassPermissions"
  }
}
```

*(Note: If a standalone `"model": "sonnet[1m]"` line exists at the root of `settings.json`, delete it so `ANTHROPIC_MODEL` within `env` handles routing.)*

### 4.2 Verification Test
Run a quick probe command in any terminal:

```bash
claude -p "Say 'Hello from Gemini 3.8 Flash'"
```

**Expected output**:
```text
Hello from Gemini 3.8 Flash
```

---

## 5. Configuring Claude Desktop App

### 5.1 Third-Party Inference Setup

1. Open Claude Desktop, open **Settings** → **Configure third-party inference**.
2. Select **Gateway** in the top dropdown.
3. Under **GATEWAY CREDENTIALS**:
   - **Gateway base URL**: `http://127.0.0.1:8045` *(do not append `/v1`)*
   - **Gateway API key**: `<ANTIGRAVITY_MANAGER_API_KEY>`
   - **Gateway auth scheme**: `bearer`
   - **Custom inference headers**: Leave empty
   - **Credential kind**: `Static API key`
4. Under **MODELS**:
   - **Model discovery**: **Enable (turn on)**. The proxy natively implements `GET /v1/models`. Enabling this lets the client automatically discover `claude-*` model aliases.
   - **Model list**: **Keep completely empty** (if you manually added custom entries previously, remove them using the `×` icon).

### 5.2 Bypassing UI Validation
Entering custom non-Anthropic model names manually causes Claude Desktop to display a validation error:
> `Doesn't look like an Anthropic model: expected a gateway model route referencing an Anthropic model...`

This locks the **Apply Changes** button. Keeping the model list empty and relying on Model Discovery allows the gateway's `custom_mapping` to handle transparent rewrites behind the scenes.

Click **Test connection** followed by **Test model discovery**, and then click **Apply Changes**.

### 5.3 Starting a Chat
Start a new chat in Claude Desktop. Keep the default model selected (or pick any Sonnet variant). All queries will now route seamlessly through Antigravity Manager to Gemini 3.8 Flash.

---

## 6. Troubleshooting & FAQ

### Q1: Error: `Model discovery — Invalid custom3p managed config: baseUrl: must use https (or http on loopback)`
- **Cause**: The Base URL was configured with a LAN IP (e.g., `http://192.168.68.55:8045`). Claude Desktop enforces HTTPS for non-loopback addresses.
- **Fix**: Use local port forwarding and configure the client with `http://127.0.0.1:8045`.

### Q2: Chat shows `Model overloaded · Retrying (2/10) · 15s`
- **Cause**: Requests to default Claude models hit rate limits (HTTP 429) on Anthropic accounts.
- **Fix**: Ensure `custom_mapping` in the server's `gui_config.json` maps `"claude*": "gemini-3.8-flash-high"`, and restart the Docker container.

### Q3: Claude Code CLI returns `Failed to authenticate. API Error: 401 Invalid API Key`
- **Cause**: The value in `ANTHROPIC_AUTH_TOKEN` does not match the `proxy.api_key` configured in Antigravity Manager, or stale credentials remained in `~/.claude/settings.json`.
- **Fix**: Verify `proxy.api_key` in `gui_config.json` and ensure `settings.json` matches exactly.

### Q4: Connection drops after rebooting Windows
- **Checks**:
  1. Confirm the Windows `iphlpsvc` (IP Helper) service is running.
  2. Run `netsh interface portproxy show all` in an elevated terminal to verify the forwarding rule is intact.
  3. Ensure the server's LAN IP address has not changed due to DHCP lease renewal.
