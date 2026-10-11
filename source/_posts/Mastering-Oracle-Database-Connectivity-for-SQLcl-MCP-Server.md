---
title: Mastering Oracle Database Connectivity for SQLcl MCP Server
date: 2026-02-03 15:07:17
tags: [Oracle, SQLcl, MCP, Database Connectivity, AI]
---

# Mastering Oracle Database Connectivity for SQLcl MCP Server

The **SQLcl MCP Server** is a powerful bridge that brings the capabilities of Oracle SQLcl to the world of AI agents and large language models (LLMs). By leveraging the **Model Context Protocol (MCP)**, it allows your AI environment to interact directly with Oracle Databases. However, for this interaction to be seamless and secure, proper connection management is essential.

In this post, we’ll explore how to configure and persist database connections using the SQLcl connection store, ensuring your MCP server is always ready to execute queries.

---

## Prerequisites: Getting SQLcl

Before you can configure your connections, you need to have SQLcl installed on your system. 

To download the latest version using a direct link, go to:
[https://download.oracle.com/otn_software/java/sqldeveloper/sqlcl-latest.zip](https://download.oracle.com/otn_software/java/sqldeveloper/sqlcl-latest.zip)

This direct link is particularly useful for automating the setup as part of a script or a container build process.

### Installation and Extraction

Once the download is complete, simply unzip the package to your desired location:

```bash
# Example for Linux/macOS
unzip sqlcl-latest.zip -d ~/

# Add to your PATH
export PATH=$PATH:~/sqlcl/bin
```

For Windows users, you can use your favorite extraction tool and add the `bin` directory to your System Environment Variables.

---

## The Heart of the Configuration: `~/.dbtools`


The SQLcl MCP Server doesn't just "guess" how to connect to your database. It relies on a standardized connection store located in your `~/.dbtools` directory. This directory acts as the central repository for your saved connection descriptors and credentials.

To manage these connections, we use two primary tools within SQLcl:
- `connect`: The standard command to establish a session.
- `connmgr`: The Connection Manager used to list, save, and organize your connections.

---

## Step-by-Step: Persistent Connection Setup

Setting up a connection involves moving from a "one-off" login to a "stored" configuration that the MCP server can reuse.

### 1. Launch SQLcl
Start by opening SQLcl without a specific connection to enter the interactive shell:
```bash
❯ sql /nolog
```

### 2. Configure Cloud Credentials (for Autonomous Databases)
If you are connecting to an Oracle Autonomous Database (ADB), you likely have a wallet file. Use the `SET CLOUDCONFIG` command to point SQLcl to your wallet:
```sql
SQL> SET CLOUDCONFIG /home/kamus/Wallet_AORA23AI_OSAKA.zip
```

### 3. Establish and Save the Connection
This is the most critical step. You need to connect and then tell SQLcl to **persist** this connection with the password so the MCP server can use it later without manual intervention.

```sql
SQL> connect admin@aora23ai_high
Password? (**********?) ***************
Connected.

-- Save the connection with a recognizable name (e.g., sqlcl_mcp)
SQL> conn -save sqlcl_mcp -savepwd admin@aora23ai_high
```
*   `-save sqlcl_mcp`: Assigns a friendly alias to this connection.
*   `-savepwd`: Stores the password securely in the connection store.

### 4. Verify with Connection Manager
Once saved, you can verify that the connection is correctly registered in the store:
```sql
SQL> connmgr list
.
└── sqlcl_mcp
```

Now, the entry `sqlcl_mcp` contains the username, the TNS descriptor, and the encrypted password, all mapped to a single name.

---

## Putting it into Action: An Agent Interaction

### Configure MCP to Enable the SQLcl Server

Before the agent can call SQLcl tools, make sure your MCP configuration enables the `sqlcl` server:

```json
"sqlcl": {
  "args": [
    "-mcp"
  ],
  "command": "/your/path/to/sqlcl/bin/sql",
  "disabled": false
}
```

Once your connections are saved, an AI agent can autonomously discover and use them. Here’s a look at how an interaction unfolds when an agent needs to connect to your database:

### The Discovery Phase

> **User**: Use SQLcl MCP to connect to my database.
>
> **Agent**: I need to connect to your database using SQLcl MCP. Let me check for available connections.
>
> *The agent first tries a common default name:*
> `mcp__sqlcl__connect(connection_name: "orcl")` → ❌ **Error**: Connection not found.
>
> **Agent**: I couldn't find a default connection. Could you specify the name? Or I can try to find it myself.
>
> **User**: List connections by yourself.
>
> **Agent**: Scanning available connections...
> `mcp__sqlcl__list-connections(show_details: true)` → ✅ **Found**: `sqlcl_mcp`

### Establishing the Connection

> **Agent**: Great! I found a saved connection named `sqlcl_mcp`. Establishing the session now...
> `mcp__sqlcl__connect(connection_name: "sqlcl_mcp")` → ✅ **Success**!
> 
> **Agent**: Connected successfully to Oracle Database 23ai in READ WRITE mode.

### Querying Data

> **User**: List my tables.
>
> **Agent**: Let me retrieve the list of tables for you.
> `mcp__sqlcl__run-sql(sql: "SELECT owner, table_name FROM all_tables ...")`
>
> **Agent**: You have several tables in your `ADMIN` schema:
> - **Movies & Entertainment**: `MOVIES`, `ACTORS`, `STREAMS`, `GENRE`
> - **Business**: `CUSTOMER`, `PIZZA_SHOP`
> - **System/Logs**: `DBTOOLS$EXECUTION_HISTORY`, `DBTOOLS$MCP_LOG`

---

## Why This Matters for AI Agents


When an AI agent (like Antigravity or any MCP-compatible client) uses the SQLcl MCP server, it needs a reliable way to authenticate. By naming your connection `sqlcl_mcp` (or any name you prefer), you can simply tell your MCP server to use that specific alias.

This setup offers several benefits:
1.  **Security**: Passwords are not stored in plain text in your agent's configuration files.
2.  **Simplicity**: You don't need to pass complex TNS strings or wallet paths every time.
3.  **Portability**: Your connection details stay in your local `~/.dbtools` directory, while your code/agent configurations remain clean.

## Conclusion

Persisting your connections is the final piece of the puzzle in building a robust AI-to-Database workflow. By mastering the `connmgr` and the `-save` flags in SQLcl, you ensure that your SQLcl MCP Server is a reliable, high-performance gateway to your Oracle data.

**Happy Querying!**