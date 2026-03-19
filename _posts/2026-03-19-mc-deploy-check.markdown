---
tags: [mcp, typescript, vercel, docker, claude, ai, architecture]
author: Nicolas Mugnier
categories: architecture
title: "Building a \"Can I Deploy?\" MCP Server with TypeScript and Vercel"
description: "How I built a small MCP server that tells you whether it's safe to deploy to production, using TypeScript, Docker, and Vercel."
image: /assets/img/mcp-deploy-check.webp
locale: en_US
---

I wanted to understand MCP (Model Context Protocol) hands-on — not just read the docs, but actually build something, hit the rough edges, and see how it feels to extend Claude with a custom tool.

For the experiment I picked a deliberately simple idea: a server that answers one question — *can I deploy to production right now?* — based on the day and time. The ["never deploy on Friday"](https://medium.com/openclassrooms-product-design-and-engineering/do-not-deploy-on-friday-92b1b46ebfe6) rule is a classic piece of engineering folklore, which made it a fun and relatable subject for a first MCP project. The server is intentionally opinionated (and a bit tongue-in-cheek): it blocks Fridays, weekends, and anything outside business hours, and gives you a reason either way.

The goal here was to learn, not to enforce policy.

Here's what I built, how it works, and what I learned along the way — from a local Docker POC to a production deployment on Vercel with a custom domain.

---

## What is MCP?

MCP (Model Context Protocol) is an open protocol developed by Anthropic that lets you extend AI assistants like Claude with custom tools. Instead of just chatting, Claude can call your server to fetch data, run checks, or trigger actions — and use the results to answer your question.

Before MCP, connecting an LLM to external capabilities meant either baking everything into the prompt, writing custom glue code per-application, or relying on proprietary plugin systems tied to a specific platform. MCP standardizes this: any client that speaks the protocol (Claude Code, Claude Desktop, or your own app built on the SDK) can connect to any compliant server.

The architecture is a simple client-server pattern:

- **You** define tools on the server — each tool has a name, a description, and an optional JSON Schema for parameters.
- **Claude** reads the tool list at startup and decides autonomously when to call them, based on the conversation context.
- **The server** executes the tool and returns structured content (text, images, or other types).
- **Claude** uses the result to formulate its response.

The description of each tool is critical — it's what Claude uses to decide *when* to invoke it. A well-written description ("Checks if it's currently safe to deploy to production based on the day and time") is more reliable than a vague one.

Two transport modes exist depending on where your server runs:
- **stdio** — the client spawns the server as a child process and communicates over stdin/stdout. Simple, zero network config, ideal for local tools.
- **Streamable HTTP** — the server exposes an HTTP endpoint. Required for remote/shared deployments.

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

  const dayName = now.toLocaleDateString('en-US', { weekday: 'long' });

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

## Step 2: Containerizing and testing locally

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

Build the image once:

```bash
docker build -t mcp-deploy-check .
```

Then drop the `.mcp.json` at the root of any project where you want the tool available. Claude Code picks it up automatically on next launch.

### Testing locally with Claude Code

Once the server is registered, just open Claude Code in that project and ask naturally:

```
You: can I deploy?

Claude: I'll check that for you.
        [Calling tool: can_i_deploy]
        🚫 NO — It's Friday. Step away from the keyboard.
```

Or on a Wednesday morning:

```
You: is now a good time to push to prod?

Claude: [Calling tool: can_i_deploy]
        ✅ YES — You're good to go! ~6h left in the window.
```

Claude decides on its own to call the tool — you don't have to say "use the deploy-check tool". The tool description is enough for it to figure out the intent.

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
    body = await new Promise((resolve) => {
      let data = "";
      req.on("data", (chunk) => (data += chunk));
      req.on("end", () => resolve(JSON.parse(data)));
    });
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

After connecting the GitHub repository to Vercel and deploying, the server is live.

---

## Step 4: Testing it

The server repo itself keeps its Docker-based `.mcp.json` — that's for developing and testing the tool locally. The deployed endpoint is meant to be consumed from *other* projects.

### Configure the client

There are two ways to point Claude Code at the live server.

**Option 1 — manually**, add a `.mcp.json` in the project directory, or in your global Claude Code configuration to have it available everywhere:

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

**Option 2 — via the `claude mcp add` command**, which writes the configuration for you:

```bash
# Available only in the current project
claude mcp add --transport http deploy-check https://mcp-deploy-check.anyvoid.dev/mcp

# Available globally across all your projects
claude mcp add --transport http --scope user deploy-check https://mcp-deploy-check.anyvoid.dev/mcp
```

> **Note:** in both options, specifying the transport is required — `"type": "http"` in the JSON file, `--transport http` in the CLI. Omitting it causes a schema validation error in Claude Code.

### Verify with curl

Before using it through Claude, you can hit the endpoint directly. The Streamable HTTP transport expects clients to declare support for both response formats:

```bash
curl -X POST "https://mcp-deploy-check.anyvoid.dev/mcp" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"can_i_deploy","arguments":{}}}'
```

Response on a Tuesday evening:

```
event: message
data: {"result":{"content":[{"type":"text","text":"🚫 NO — It's 21:47 — end of business hours. Next window: Wednesday at 9:00."}]}}
```

### Use it through Claude Code

Once configured, the interaction is the same as with the local Docker server — just ask naturally:

```
You: can I deploy?

Claude: [Calling tool: can_i_deploy]
        🚫 NO — It's Friday. Step away from the keyboard.
```

The difference is that the tool now runs on Vercel rather than in a local Docker container — but from Claude's perspective, nothing changed.

---

## Final architecture

```
                    Claude Code
                        │
          ┌─────────────┴──────────────┐
          │ local (stdio)              │ remote (HTTP)
          ▼                            ▼
  Docker container             Vercel Serverless
  node dist/index.js           Function (api/mcp.ts)
          │                            │
          └─────────────┬──────────────┘
                        ▼
              checkDeploy() — inspects day + time
                        │
                        ▼
          "🚫 NO — It's Friday. Step away from the keyboard."
```

---

## Conclusion

The deploy-check tool is deliberately trivial — checking the day and time is not a hard problem. But that's exactly what made it a good first MCP project: the logic was simple enough to stay out of the way, and I could focus entirely on understanding the protocol, the transport layers, the tool registration, and how Claude actually decides to invoke a tool.

What I took away is less about this specific server and more about what the pattern makes possible. MCP lets you give Claude contextual knowledge about your environment — your infrastructure, your processes, your team's constraints — without building a dedicated interface for it. The assistant doesn't just answer general questions anymore; it can answer *your* questions, grounded in *your* reality.

That's the value. Not the automation, but the reduction of friction between a question and a reliable answer.

Using this naturally raises a broader question: when does a problem call for an MCP server, rather than a prompt or a `CLAUDE.md` file? That's worth a dedicated post.

---

## References

- [mcp-deploy-check — GitHub repository](https://github.com/NicolasMugnier/mcp-deploy-check)
- [Model Context Protocol — official documentation](https://modelcontextprotocol.io)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
