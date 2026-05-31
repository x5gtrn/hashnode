---
title: "Unlocking OmniFocus: A Guide to Secure Remote Access with Cloudflare Tunnel and MCP"
datePublished: 2026-01-01T06:41:41.834Z
cuid: cmjv2ud3e000f02k1hk50eqig
slug: article-2026-01-01-1540
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1767249023628/eb3e08a9-0fa4-4932-8153-8a412c46a226.jpeg
tags: cloudflare, ai, task-management, claude, mcp, mcp-server, mcp-client

---

[OmniFocus](https://www.omnigroup.com/omnifocus/) is a powerhouse for task management, but its reliance on local access can feel restrictive in an era of cloud-native workflows. For engineers who live in the terminal and collaborate across distributed systems, the inability to programmatically interact with their task manager from anywhere is a significant bottleneck. What if you could bridge this gap—securely, and without punching a single hole in your firewall?

This guide provides a comprehensive walkthrough for building a robust, secure, and persistent bridge between your always-on Mac running OmniFocus and a remote client like Claude Desktop. By leveraging the **Model Context Protocol (MCP)**, the excellent [**omnifocus-mcp-enhanced**](https://github.com/jqlts1/omnifocus-mcp-enhanced) \[1\] server, and the power of **Cloudflare Tunnel** \[2\], you can create a personal, AI-enabled productivity endpoint. We'll go beyond a simple proof-of-concept to build a production-ready setup that is both powerful and secure.

%[https://speakerdeck.com/x5gtrn/remote-omnifocus-access-via-cloudflare-tunnel-plus-remote-mcp] 

## The Problem: OmniFocus in a Cloud-First World

OmniFocus is built on a philosophy of local-first data ownership, which is admirable from a privacy and control perspective. However, this design choice creates friction for modern workflows. Consider these scenarios:

* You're traveling without your Mac and need to quickly add a task based on a conversation.
    
* You want to integrate OmniFocus with a CI/CD pipeline to automatically create tasks from failed builds.
    
* You'd like to use AI assistants like Claude to intelligently query, filter, and organize your tasks using natural language.
    
* You need to build custom dashboards or analytics on top of your task data.
    

Traditional solutions like screen sharing or VNC are clunky and insecure. Cloud sync services like OmniSync solve device synchronization but don't provide programmatic access. This is where the **Model Context Protocol** comes in.

## Understanding the Model Context Protocol (MCP)

The Model Context Protocol is an open standard developed by Anthropic \[3\] that allows AI applications to securely connect to external data sources and tools. Think of it as a universal adapter that lets language models interact with your local services, databases, and applications in a structured, secure way.

MCP operates on a client-server model. The **MCP server** exposes a set of tools (functions) that can be invoked by an **MCP client** (like Claude Desktop). Communication happens via JSON-RPC, and the protocol supports both stdio (standard input/output) for local processes and HTTP/SSE (Server-Sent Events) for network communication.

The beauty of MCP is its simplicity and extensibility. By wrapping OmniFocus's AppleScript interface in an MCP server, we can expose rich task management capabilities to any MCP-compatible client, anywhere in the world.

## The Architectural Blueprint: From Local to Global

At its core, our goal is to expose a local-only service to the internet securely. The architecture consists of three main layers:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767249220531/6084b501-3dbd-4e3d-8233-f7bfd0cfe524.jpeg align="center")

### Layer 1: The Local Server (Your Mac)

This is the heart of the operation. It runs four key components:

1. **OmniFocus Pro**: The source of truth for your tasks. It must be running for the MCP server to interact with it via AppleScript.
    
2. **omnifocus-mcp-enhanced**: A Node.js-based MCP server that translates MCP tool calls into AppleScript commands. It provides 16 tools, including task creation, querying, batch operations, and custom perspective access.
    
3. **mcp-remote**: A proxy that converts the stdio-based MCP server into an HTTP/SSE endpoint. This is crucial because Cloudflare Tunnel works with HTTP services, not stdio.
    
4. **cloudflared**: The Cloudflare Tunnel daemon that establishes a secure, outbound-only connection to the Cloudflare network.
    

### Layer 2: The Cloud Layer (Cloudflare)

This is our secure bridge. Instead of opening inbound ports, the `cloudflared` daemon on your Mac creates a persistent, encrypted tunnel to the Cloudflare network. This tunnel is established via an outbound connection, which means:

* **No port forwarding**: Your home router remains locked down.
    
* **IP address masking**: Your public IP is never exposed.
    
* **DDoS protection**: Cloudflare's network absorbs malicious traffic.
    
* **Zero Trust access**: You can layer on authentication and authorization policies.
    

The tunnel is assigned a public hostname (e.g., [`omnifocus.yourdomain.com`](http://omnifocus.yourdomain.com)), which routes all HTTPS traffic through the encrypted tunnel back to your local server.

### Layer 3: The Remote Client (Claude Desktop)

This is your interaction point. Claude Desktop (or any MCP-compatible client) communicates via standard HTTPS to the public hostname provided by Cloudflare. The client sends JSON-RPC requests, which are routed through the tunnel, converted by `mcp-remote`, and executed by the MCP server.

This model is fundamentally more secure than traditional port forwarding, as it never exposes your home network directly to the internet.

## Prerequisites: What You'll Need

Before we begin, ensure you have the following in place:

| Category | Requirement |
| --- | --- |
| **Hardware & OS** | An always-on Mac (Intel or Apple Silicon) running macOS 11.0+ |
| **Software** | OmniFocus Pro (v3+), Node.js (v18+), and a terminal client (e.g., iTerm2). |
| **Accounts** | A free Cloudflare account. |
| **Network** | Stable internet connection with outbound HTTPS access. |

**Why OmniFocus Pro?** The Pro version is required for custom perspective access, which is one of the most powerful features of `omnifocus-mcp-enhanced`. Custom perspectives allow you to create highly specific views of your tasks (e.g., "all available tasks under 30 minutes with no dependencies"), and the MCP server can query these programmatically.

## Step 1: Setting Up the OmniFocus MCP Server

The foundation of this setup is `omnifocus-mcp-enhanced`, a powerful Node.js server that acts as a bridge to OmniFocus's AppleScript interface. It exposes a rich set of tools for task management.

### Installation

First, install it via npm. Using `npx` is a clean way to run it without global installation conflicts:

```bash
# This command adds the server to Claude's MCP list and runs it via npx
claude mcp add omnifocus-enhanced -- npx -y omnifocus-mcp-enhanced
```

If you prefer a global installation for easier debugging:

```bash
npm install -g omnifocus-mcp-enhanced
claude mcp add omnifocus-enhanced -- omnifocus-mcp-enhanced
```

### What Tools Does It Provide?

The server exposes 16 tools across four categories:

**Database & Task Management (8 tools):**

* `dump_database`: Get the complete OmniFocus database state.
    
* `add_omnifocus_task`: Create tasks with full support for subtasks, due dates, tags, and notes.
    
* `add_project`: Create new projects.
    
* `remove_item`: Delete tasks or projects.
    
* `edit_item`: Modify existing tasks or projects.
    
* `batch_add_items`: Bulk create tasks and subtasks.
    
* `batch_remove_items`: Bulk delete items.
    
* `get_task_by_id`: Query specific task information.
    

**Built-in Perspectives (5 tools):**

* `get_inbox_tasks`: Access your Inbox.
    
* `get_flagged_tasks`: View all flagged tasks.
    
* `get_forecast_tasks`: See tasks due or deferred in the next N days.
    
* `get_tasks_by_tag`: Filter by tag name.
    
* `filter_tasks`: Advanced filtering with unlimited combinations (status, estimates, due dates, notes, etc.).
    

**Custom Perspectives (2 tools - NEW):**

* `list_custom_perspectives`: List all your custom perspectives.
    
* `get_custom_perspective_tasks`: Access a custom perspective with hierarchical task display.
    

**Analytics (1 tool):**

* `get_today_completed_tasks`: View today's completed tasks.
    

### Testing the Server

To verify the installation, you can test it locally. Run the server in stdio mode and send a JSON-RPC request:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"get_inbox_tasks","params":{}}' | npx omnifocus-mcp-enhanced
```

You should see a JSON response with your inbox tasks.

## Step 2: The Protocol Bridge - From Stdio to HTTP/SSE

To make the stdio-based MCP server accessible over a network, we need a proxy. This proxy will listen for HTTP requests and translate them into stdio commands for the MCP server, then return the responses. We'll use `mcp-remote` for this.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767249294492/d0af088e-5288-4919-8aff-b88e4f2899ca.jpeg align="center")

This diagram illustrates the flow: a standard JSON-RPC request over stdio is wrapped into an HTTP POST request, and the response is streamed back via Server-Sent Events (SSE). SSE is ideal for this use case because it allows the server to push updates to the client in real-time, which is useful for long-running operations or streaming responses.

### Installing mcp-remote

First, install `mcp-remote` globally:

```bash
npm install -g mcp-remote
```

### Running the Proxy

Run the following command in your terminal. This tells `mcp-remote` to act as a proxy, listening on port 3000 and forwarding all communication to our `omnifocus-mcp-enhanced` server process.

```bash
# Start the proxy server on port 3000
mcp-remote --stdio "npx omnifocus-mcp-enhanced" --port 3000
```

At this point, you have a local HTTP server running on [`http://localhost:3000`](http://localhost:3000) that can control OmniFocus. You can test this with `curl`:

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"get_inbox_tasks","params":{}}' \
  http://localhost:3000
```

You should receive a JSON response with your inbox tasks. This confirms that the protocol conversion is working correctly.

### Understanding the Data Flow

Let's break down what happens when you make a request:

1. **Client sends HTTP POST**: The client (e.g., Claude Desktop) sends a JSON-RPC request to [`http://localhost:3000`](http://localhost:3000).
    
2. **mcp-remote receives the request**: The proxy parses the HTTP body and extracts the JSON-RPC payload.
    
3. **Proxy forwards to stdio**: The proxy writes the JSON-RPC request to the stdin of the `omnifocus-mcp-enhanced` process.
    
4. **MCP server executes**: The server parses the request, executes the corresponding AppleScript, and writes the result to stdout.
    
5. **Proxy reads stdout**: The proxy reads the JSON-RPC response from stdout.
    
6. **Proxy sends HTTP response**: The proxy wraps the response in an HTTP 200 OK with `Content-Type: text/event-stream` and streams it back to the client via SSE.
    

This design is elegant because it maintains the simplicity of stdio for local communication while enabling network access.

## Step 3: Building the Secure Tunnel with Cloudflare

Now, we'll expose our local server to the internet without opening any ports. This is where Cloudflare Tunnel shines.

### Why Cloudflare Tunnel?

Traditional remote access methods have significant drawbacks:

* **Port forwarding**: Exposes your home IP and requires router configuration. Vulnerable to attacks.
    
* **VPN**: Requires client-side setup and can be complex to manage.
    
* **ngrok/localtunnel**: Great for testing, but free tiers have limitations and URLs change frequently.
    

Cloudflare Tunnel solves these problems by creating a secure, persistent, outbound-only connection from your Mac to Cloudflare's network. Traffic is routed through Cloudflare's global edge, which provides DDoS protection, caching, and Zero Trust access controls—all on the free tier.

### Installation and Setup

1. **Install** `cloudflared`: If you don't have it, install the daemon using Homebrew:
    
    ```bash
    brew install cloudflared
    ```
    
2. **Authenticate**: Log in to your Cloudflare account. This will open a browser window for authentication.
    
    ```bash
    cloudflared tunnel login
    ```
    
    After successful login, a certificate file is saved to `~/.cloudflared/cert.pem`.
    
3. **Create a Tunnel**: Give your tunnel a memorable name.
    
    ```bash
    cloudflared tunnel create omnifocus-mcp
    ```
    
    This will generate:
    
    * A UUID for your tunnel (e.g., `abc123-def456-ghi789`).
        
    * A credentials file at `~/.cloudflared/<UUID>.json`.
        
    
    Save the UUID—you'll need it for the configuration file.
    
4. **Configure the Tunnel**: Create a `config.yml` file in `~/.cloudflared/`. This file tells the daemon how to route traffic.
    
    ```yaml
    tunnel: abc123-def456-ghi789  # Replace with your tunnel UUID
    credentials-file: /Users/yourname/.cloudflared/abc123-def456-ghi789.json
    
    ingress:
      # Route traffic from omnifocus.yourdomain.com to localhost:3000
      - hostname: omnifocus.yourdomain.com
        service: http://localhost:3000
      # Catch-all rule to prevent unconfigured hostnames from exposing your service
      - service: http_status:404
    ```
    
    The `ingress` rules define how traffic is routed. The first rule matches the hostname and forwards to your local service. The catch-all rule returns a 404 for any other hostname, which prevents accidental exposure.
    
5. **Route DNS**: Link your tunnel to a public DNS record. This command creates a CNAME record in your Cloudflare DNS that points to the tunnel.
    
    ```bash
    cloudflared tunnel route dns omnifocus-mcp omnifocus.yourdomain.com
    ```
    
6. **Run the Tunnel**: Start the tunnel to begin proxying traffic.
    
    ```bash
    cloudflared tunnel run omnifocus-mcp
    ```
    
    You should see output indicating the tunnel is connected:
    
    ```plaintext
    2025-01-01T12:00:00Z INF Connection registered connIndex=0 location=SFO
    2025-01-01T12:00:00Z INF Registered tunnel connection connIndex=0 location=SFO
    ```
    

Your local server on port 3000 is now securely accessible at [`https://omnifocus.yourdomain.com`](https://omnifocus.yourdomain.com)!

### Testing the Tunnel

From any device with internet access, test the tunnel:

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"get_inbox_tasks","params":{}}' \
  https://omnifocus.yourdomain.com
```

You should receive your inbox tasks, confirming that the entire pipeline is working.

## Step 4: Configuring the Remote Client

With the tunnel active, the final step is to tell your remote client (Claude Desktop) how to connect. Claude Desktop uses a configuration file to define MCP servers.

### Locating the Configuration File

The configuration file is located at:

* **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
    
* **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`
    

### Adding the Server

Edit the file to add your remote MCP server:

```json
{
  "mcpServers": {
    "omnifocus": {
      "url": "https://omnifocus.yourdomain.com",
      "type": "http"
    }
  }
}
```

### Restarting Claude Desktop

Restart Claude Desktop to load the new configuration. You should now see "omnifocus" listed in the MCP servers section.

### Testing the Integration

Open a conversation in Claude and try a command:

```plaintext
Can you show me my inbox tasks?
```

Claude will invoke the `get_inbox_tasks` tool and display the results. You can also try more complex queries:

```plaintext
Create a task called "Review Q1 metrics" in the "Work" project, due next Friday, with a 2-hour estimate.
```

Claude will parse your natural language request and call the appropriate MCP tool with the correct parameters.

## Step 5: Ensuring Persistence with `launchd`

For a truly "always-on" experience, you need the `mcp-remote` proxy and the `cloudflared` tunnel to run continuously and restart automatically. On macOS, `launchd` is the perfect tool for this.

### Creating a `launchd` Service for mcp-remote

1. **Create the** `.plist` file: Save this to `~/Library/LaunchAgents/com.omnifocus.mcp.proxy.plist`.
    
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
        <key>Label</key>
        <string>com.omnifocus.mcp.proxy</string>
        <key>ProgramArguments</key>
        <array>
            <string>/usr/local/bin/mcp-remote</string>
            <string>--stdio</string>
            <string>npx omnifocus-mcp-enhanced</string>
            <string>--port</string>
            <string>3000</string>
        </array>
        <key>RunAtLoad</key>
        <true/>
        <key>KeepAlive</key>
        <true/>
        <key>StandardOutPath</key>
        <string>/tmp/mcp-remote.log</string>
        <key>StandardErrorPath</key>
        <string>/tmp/mcp-remote.error.log</string>
    </dict>
    </plist>
    ```
    
    **Key fields:**
    
    * `Label`: Unique identifier for the service.
        
    * `ProgramArguments`: The command to run. Update `/usr/local/bin/mcp-remote` to the actual path (find it with `which mcp-remote`).
        
    * `RunAtLoad`: Start the service when the user logs in.
        
    * `KeepAlive`: Restart the service if it crashes.
        
    * `StandardOutPath` and `StandardErrorPath`: Log files for debugging.
        
2. **Load the service**:
    
    ```bash
    launchctl load ~/Library/LaunchAgents/com.omnifocus.mcp.proxy.plist
    ```
    
3. **Verify it's running**:
    
    ```bash
    launchctl list | grep omnifocus
    ```
    
    You should see the service listed with a PID.
    

### Creating a `launchd` Service for cloudflared

Cloudflare provides a built-in command to install the tunnel as a service:

```bash
sudo cloudflared service install
```

This creates a system-level `launchd` service that runs at boot. To start it immediately:

```bash
sudo launchctl start com.cloudflare.cloudflared
```

### Handling macOS Sleep

One challenge with always-on services on macOS is sleep management. If your Mac goes to sleep, the tunnel will disconnect. To prevent this, you can:

1. **Disable sleep**: Go to **System Preferences &gt; Energy Saver** and set "Prevent your Mac from automatically sleeping when the display is off."
    
2. **Use** `caffeinate`: Run `caffeinate -s` to prevent sleep while the terminal is open.
    
3. **Use a third-party tool**: Apps like Amphetamine can prevent sleep based on custom rules.
    

For a true server setup, consider running this on a Mac Mini or an old MacBook with the lid closed and power management configured for 24/7 operation.

## Real-World Use Cases

Now that your setup is live, what can you do with it? Here are some practical examples:

### 1\. AI-Powered Task Triage

Use Claude to intelligently filter and prioritize your tasks:

```plaintext
Show me all available tasks under 30 minutes that are due this week, sorted by project.
```

Claude will call the `filter_tasks` tool with the appropriate parameters and present a clean, organized list.

### 2\. Automated Task Creation from External Systems

Integrate with webhooks or CI/CD pipelines to automatically create tasks. For example, when a GitHub issue is assigned to you, a webhook can POST to your MCP server:

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"add_omnifocus_task","params":{"name":"Fix bug #123","projectName":"Engineering","tags":["bug","urgent"]}}' \
  https://omnifocus.yourdomain.com
```

### 3\. Custom Dashboards

Build a web dashboard that queries your OmniFocus data via the MCP server. You could visualize:

* Tasks completed per day/week.
    
* Time estimates vs. actuals.
    
* Project progress.
    

### 4\. Voice-Activated Task Management

Combine this setup with a voice assistant (e.g., Siri Shortcuts or a custom Alexa skill) to add tasks hands-free.

### 5\. Cross-Platform Access

Since the MCP server is now accessible via HTTPS, you can build clients for any platform—iOS, Android, web, or even a command-line tool.

## A Multi-Layered Security Approach

This architecture is secure by design, but you can harden it further.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767249340105/29335392-7338-444a-85cd-0e8512aacb53.jpeg align="center")

### Layer 1: Cloudflare DDoS Protection

Cloudflare's network automatically mitigates large-scale traffic floods and volumetric attacks, ensuring your service remains available even under attack.

### Layer 2: HTTPS/TLS Encryption

All traffic between the client and Cloudflare, and between Cloudflare and your Mac, is encrypted using TLS. This prevents eavesdropping and tampering.

### Layer 3: Zero Trust Access

Cloudflare Access allows you to add an authentication layer before any traffic reaches your tunnel. You can require users to log in with Google, GitHub, or any OIDC provider. You can also restrict access to specific IP ranges or countries.

To enable Cloudflare Access:

1. Go to **Cloudflare Dashboard &gt; Zero Trust &gt; Access &gt; Applications**.
    
2. Click **Add an Application** and select **Self-hosted**.
    
3. Set the application domain to [`omnifocus.yourdomain.com`](http://omnifocus.yourdomain.com).
    
4. Configure an access policy (e.g., "Allow emails ending in @[yourcompany.com](http://yourcompany.com)").
    

Now, anyone trying to access your MCP server will be prompted to authenticate first.

### Layer 4: API Key Management

For production use, consider forking the `omnifocus-mcp-enhanced` server to add a simple API key check. You can pass the key as a header:

```bash
curl -X POST -H "Content-Type: application/json" \
  -H "X-API-Key: your-secret-key" \
  -d '{"jsonrpc":"2.0","id":1,"method":"get_inbox_tasks","params":{}}' \
  https://omnifocus.yourdomain.com
```

The server can validate the key before processing the request.

### Layer 5: Local Firewall

Ensure your Mac's firewall is enabled and only allows outbound connections. Since the tunnel is outbound-only, you don't need to open any inbound ports.

### Monitoring and Logging

Regularly review the Cloudflare dashboard for traffic patterns and potential threats. The `launchd` service logs (`/tmp/mcp-remote.log`) can help you debug issues and monitor usage.

## Troubleshooting Common Issues

### Issue 1: Tunnel Not Connecting

**Symptoms**: `cloudflared tunnel run` fails with "connection refused" or "authentication failed."

**Solutions**:

* Verify your credentials file path in `config.yml`.
    
* Ensure you're logged in with `cloudflared tunnel login`.
    
* Check that the tunnel UUID in `config.yml` matches the one created.
    

### Issue 2: MCP Server Not Responding

**Symptoms**: Requests to the tunnel return 502 Bad Gateway.

**Solutions**:

* Verify `mcp-remote` is running on port 3000 with `lsof -i :3000`.
    
* Check the logs at `/tmp/mcp-remote.log` for errors.
    
* Ensure OmniFocus is running.
    

### Issue 3: Claude Desktop Not Seeing the Server

**Symptoms**: The "omnifocus" server doesn't appear in Claude Desktop.

**Solutions**:

* Verify the `claude_desktop_config.json` syntax is correct (valid JSON).
    
* Restart Claude Desktop completely (quit and reopen).
    
* Check that the URL in the config is accessible from your current network.
    

### Issue 4: Slow Response Times

**Symptoms**: Requests take several seconds to complete.

**Solutions**:

* Check your internet connection speed.
    
* Verify the Cloudflare edge location is geographically close to you.
    
* Profile the AppleScript execution time—complex queries can be slow.
    

## Advanced Optimizations

### Caching Responses

For read-heavy operations (e.g., querying tasks), you can add a caching layer using Redis or a simple in-memory cache in the `mcp-remote` proxy. This reduces the load on OmniFocus and speeds up responses.

### Load Balancing

If you have multiple Macs, you can run the MCP server on each and use Cloudflare Load Balancing to distribute traffic. This provides redundancy and higher availability.

### Custom Tools

The `omnifocus-mcp-enhanced` server is open source, so you can fork it and add custom tools. For example, you could add a tool to export tasks to CSV, or integrate with external APIs (e.g., send a Slack notification when a task is completed).

## Conclusion: Your Productivity, Unleashed

By combining the power of local AppleScript automation with the secure, global reach of Cloudflare, you've effectively transformed OmniFocus into a cloud-aware service. This setup not only enables remote task management but also opens the door to more complex AI-driven workflows, custom integrations, and a truly sovereign productivity system.

The architecture we've built is production-ready, secure, and extensible. You've learned how to:

* Expose a local stdio-based service over HTTP using `mcp-remote`.
    
* Create a secure, persistent tunnel with Cloudflare without opening any ports.
    
* Integrate with Claude Desktop for natural language task management.
    
* Ensure 24/7 uptime with `launchd` services.
    
* Implement multi-layered security with Zero Trust access and encryption.
    

This is just the beginning. With this foundation, you can build custom clients, integrate with other tools, and create a productivity ecosystem that works exactly the way you want it to. The power of OmniFocus is no longer confined to your Mac—it's now accessible from anywhere, securely and seamlessly.

---

## References

1. [omnifocus-mcp-enhanced GitHub Repository](https://github.com/jqlts1/omnifocus-mcp-enhanced)
    
2. [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
    
3. [Model Context Protocol Overview](https://modelcontextprotocol.io/)
    
4. [Cloudflare Zero Trust Access](https://developers.cloudflare.com/cloudflare-one/policies/access/)
    
5. [Apple launchd Documentation](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)