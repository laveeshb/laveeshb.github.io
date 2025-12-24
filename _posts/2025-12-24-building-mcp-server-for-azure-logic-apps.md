---
layout: post
title: "Building an MCP Server for Azure Logic Apps"
date: 2025-12-24
categories: [azure]
tags: [mcp, logic-apps, typescript, ai]
excerpt: "How I built an MCP server that lets AI assistants interact with Azure Logic Apps."
---

I recently built an [MCP server for Azure Logic Apps](https://github.com/laveeshb/logicapps-mcp) that enables AI assistants like Claude and GitHub Copilot to query workflows, run history, and more. Here's how it works and some interesting design decisions along the way.

## What is MCP?

[Model Context Protocol](https://modelcontextprotocol.io/) is an open standard from Anthropic that lets AI assistants call external tools. Instead of the AI trying to guess how to interact with a service, MCP provides a structured way to expose capabilities that the AI can discover and invoke.

Think of it as a plugin system for AI assistants — you define tools with JSON schemas, and the AI can call them with the right parameters.

## The Architecture

The server is surprisingly simple. Here's the entry point:

```typescript
const server = new Server(
  { name: "logicapps-mcp", version: "0.1.0" },
  { capabilities: { tools: {} } }
);

registerTools(server);

const transport = new StdioServerTransport();
await server.connect(transport);
```

MCP uses stdio for communication — the AI assistant spawns the server process and communicates via stdin/stdout. No HTTP server, no ports to configure.

## Tool Registration

Each tool is defined with a JSON schema that describes its parameters:

```typescript
{
  name: "list_run_history",
  description: "Get the run history for a workflow",
  inputSchema: {
    type: "object",
    properties: {
      subscriptionId: { type: "string" },
      logicAppName: { type: "string" },
      top: { type: "number", description: "Number of runs (default: 25)" },
      filter: { type: "string", description: "OData filter" }
    },
    required: ["subscriptionId", "logicAppName"]
  }
}
```

The AI reads these schemas and knows exactly what parameters to provide. When it calls a tool, the server validates the input and executes the corresponding function.

## Authentication Without SDKs

Rather than pulling in the Azure SDK (which adds significant weight), the server uses Azure CLI tokens directly:

```typescript
export async function getAzureCliToken(): Promise<string> {
  const result = await exec("az account get-access-token --query accessToken -o tsv");
  return result.stdout.trim();
}
```

This keeps dependencies minimal and leverages the authentication the user already has. If you're logged into Azure CLI, the MCP server just works.

## Calling Azure Resource Manager

All Logic Apps operations go through the ARM REST API. The HTTP client is a thin wrapper that handles auth injection and pagination:

```typescript
export async function armRequest<T>(path: string): Promise<T> {
  const cloud = getCloudEndpoints();
  const token = await getAccessToken();

  const response = await fetch(`${cloud.resourceManager}${path}`, {
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json"
    }
  });

  return response.json() as T;
}
```

The server supports Azure Public, Government, and China clouds — just swap the endpoint configuration.

## Handling Both SKUs

Logic Apps comes in two flavors: Consumption (serverless, single workflow) and Standard (App Service-based, multiple workflows). The API paths differ:

- **Consumption**: `/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Logic/workflows/{name}`
- **Standard**: `/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Web/sites/{name}/hostruntime/runtime/webhooks/workflow/api/management/workflows/{workflow}`

Each tool detects the SKU and routes to the appropriate API.

## What Can You Do With It?

Once configured, you can ask your AI assistant things like:

- "List all Logic Apps in my subscription"
- "Show me failed runs from the last 24 hours"
- "What's the definition of the order-processing workflow?"
- "Get the trigger callback URL for the HTTP trigger"

The AI has 18 read-only tools covering subscriptions, workflows, runs, actions, triggers, connections, and more.

## What I Learned

**MCP is simpler than expected.** The protocol is minimal — just tool definitions and a request/response pattern over stdio. Most of the complexity is in your actual tool implementations.

**Azure CLI auth is underrated.** Reusing CLI tokens means zero credential management. Users don't need to create service principals or store secrets.

**TypeScript + Node works well for this.** The @modelcontextprotocol/sdk handles all the protocol details. You just register handlers and implement your logic.

## Try It

If you work with Logic Apps, give it a try:

```bash
npm install -g github:laveeshb/logicapps-mcp
```

Add it to Claude Desktop or VS Code Copilot, and you can start querying your Logic Apps with natural language.

The code is on [GitHub](https://github.com/laveeshb/logicapps-mcp) — contributions welcome.
