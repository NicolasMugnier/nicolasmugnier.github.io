---
tags: [mcp, typescript, vercel, docker, claude, ai, architecture]
author: Nicolas Mugnier
categories: architecture
title: "Building a \"Can I Deploy?\" MCP Server with TypeScript and Vercel"
description: "How I built a small MCP server that tells you whether it's safe to deploy to production, using TypeScript, Docker, and Vercel."
image: /assets/img/mcp-deploy-check.webp
locale: en_US
---

One of the unwritten rules of software engineering: **never deploy on a Friday**. Or after 5pm. Or on weekends. Everyone knows it, yet somehow it keeps happening.

Instead of relying on willpower, I built a small MCP (Model Context Protocol) server that answers one question: *can I deploy to production right now?* You ask Claude, Claude asks the server, the server checks the clock — done.

Here's how I built it, from a local Docker POC to a production deployment on Vercel with a custom domain.

---

## What is MCP?

MCP (Model Context Protocol) is an open protocol developed by Anthropic that lets you extend AI assistants like Claude with custom tools. Instead of just chatting, Claude can call your server to fetch data, run checks, or trigger actions — and use the results to answer your question.

It's a simple client-server pattern: you define tools (with a name, description, and optional parameters), and Claude decides when to call them based on context.

---

## The idea

A single tool: `can_i_deploy`. No parameters needed — the server just inspects the current day and time and returns a yes or no with a reason.

The rules:
- **No weekends** — close your laptop
- **No Fridays** — the classic rule, non-negotiable
- **Business hours only** — 9:00 to 17:00, so the team is around if something goes wrong
- **Otherwise** — green light, with a countdown of hours left in the window

---

## Step 1: The TypeScript server (stdio transport)

The MCP TypeScript SDK makes this straightforward. The core logic lives in a single file:

```typescript
// src/check.ts
export function checkDeploy(): { allowed: boolean; reason: string } {
  const now = new Date();
  const day = now.getDay(); // 0=Sun, 5=Fri, 6=Sat
  const hour = now.getHours();

  if (day === 0 || day === 6) {
    return { allowed: false, reason: `It's ${dayName}. Nobody deploys on weekends.` };
  }
  if (day === 5) {
    return { allowed: false, reason: `It's Friday. Step away from the keyboard.` };
  }
  if (hour < 9 || hour >= 17) {
    return { allowed: false, reason: `Outside business hours. Next window: ${nextDeployWindow()}.` };
  }
  return { allowed: true, reason: `You're good to go! ~${17 - hour}h left in the window.` };
}
```

The MCP server wires this into a tool:

```typescript
// src/index.ts
const server = new McpServer({ name: "mcp-deploy-check", version: "1.0.0" });

server.tool("can_i_deploy", "Checks if it's safe to deploy to production.", {}, async () => {
  const { allowed, reason } = checkDeploy();
  return {
    content: [{ type: "text", text: `${allowed ? "✅ YES" : "🚫 NO"} — ${reason}` }],
  };
});

await server.connect(new StdioServerTransport());
```

The stdio transport means the server communicates over stdin/stdout — Claude Code launches it as a child process.

---

## Step 2: Running it with Docker

No local Node.js installs needed. A multi-stage Dockerfile keeps the runtime image lean:

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY tsconfig.json ./
COPY src ./src
RUN npm run build

FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

And a `.mcp.json` at the project root tells Claude Code how to launch it:

```json
{
  "mcpServers": {
    "deploy-check": {
      "command": "docker",
      "args": ["run", "--rm", "-i", "mcp-deploy-check"],
      "type": "stdio"
    }
  }
}
```

At this point the POC works locally — you can ask Claude *"can I deploy?"* and it calls the tool.

---

## Step 3: Deploying to Vercel

The stdio transport only works locally. For a remote deployment, MCP supports **Streamable HTTP transport** — each request is handled independently, making it a perfect fit for serverless.

The key change is in the transport layer. The deploy logic (`check.ts`) stays untouched:

```typescript
// api/mcp.ts
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";

export default async function handler(req: IncomingMessage, res: ServerResponse) {
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined, // stateless mode
  });

  const server = createServer(); // same tool, same logic
  await server.connect(transport);

  let body: unknown = undefined;
  if (req.method === "POST") {
    body = await parseBody(req);
  }

  await transport.handleRequest(req, res, body);
  await server.close();
}
```

Vercel compiles `api/*.ts` files natively — no build configuration needed. A minimal `vercel.json` routes all traffic to the handler:

```json
{
  "outputDirectory": ".",
  "rewrites": [{ "source": "/(.*)", "destination": "/api/mcp" }]
}
```

After connecting the GitHub repository to Vercel and deploying, the server is live. The `.mcp.json` switches from Docker to HTTP:

```json
{
  "mcpServers": {
    "deploy-check": {
      "type": "http",
      "url": "https://mcp-deploy-check.anyvoid.dev/mcp"
    }
  }
}
```

> **Note:** the `type: "http"` field is required — omitting it causes a schema validation error in Claude Code.

---

## Testing it

The Streamable HTTP transport expects clients to declare support for both response formats:

```bash
curl -X POST "https://mcp-deploy-check.anyvoid.dev/mcp" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"can_i_deploy","arguments":{}}}'
```

Response on a Monday evening:

```
event: message
data: {"result":{"content":[{"type":"text","text":"🚫 NO — It's 21:47 — end of business hours. Next window: Wednesday at 9:00."}]}}
```

---

## What's next

This is a minimal POC, but the pattern opens up more interesting possibilities:

- **Timezone support** — pass a timezone argument so distributed teams get the right answer
- **Deployment freeze periods** — block deploys during release freezes or holidays via a config file
- **On-call check** — call PagerDuty or OpsGenie to verify someone is actually on-call before allowing a deploy
- **Audit log** — record every check so you have a history of who asked and when

The core idea stays the same: a single, focused tool that Claude can call when the question comes up naturally in conversation. No dashboards, no context switching — just ask.

---

## Final architecture

```
Claude Code
    │
    │  HTTP (Streamable HTTP transport)
    ▼
Vercel Serverless Function (api/mcp.ts)
    │
    ▼
checkDeploy() — inspects day + time
    │
    ▼
"🚫 NO — It's Friday. Step away from the keyboard."
```

The full source is available at [github.com/NicolasMugnier/mcp-deploy-check](https://github.com/NicolasMugnier/mcp-deploy-check).
