---
layout: post
title: "Building an MCP Server for Azure Logic Apps"
date: 2025-12-24
categories: [azure]
tags: [mcp, logic-apps, typescript, ai]
excerpt: "How I built an MCP server that lets AI assistants interact with Azure Logic Apps."
---

I recently built an [MCP server for Azure Logic Apps](https://github.com/laveeshb/logicapps-mcp) that lets AI assistants like Claude and GitHub Copilot query workflows, run history, and more. [Model Context Protocol](https://modelcontextprotocol.io/) is the glue that makes this possible — it gives AI tools a structured way to discover and call external services.

Here's how the server works and some design decisions along the way.

## The Architecture

<div class="architecture-diagram">
  <table>
    <tr>
      <td><span class="box">AI Assistant</span></td>
      <td><span class="arrow">⟷</span></td>
      <td><span class="box">MCP Server</span></td>
      <td><span class="arrow">⟷</span></td>
      <td><span class="box">Azure Logic Apps</span></td>
    </tr>
    <tr>
      <td></td>
      <td class="label">stdio</td>
      <td></td>
      <td class="label">REST API</td>
      <td></td>
    </tr>
  </table>
</div>

Here's the entry point:

```typescript
const server = new Server(
  { name: "logicapps-mcp", version: "0.1.0" },
  { capabilities: { tools: {} } }
);

registerTools(server);

const transport = new StdioServerTransport();
await server.connect(transport);
```

MCP uses `stdio` for communication — the AI assistant spawns the server process and communicates via `stdin/stdout`. No HTTP server, no ports to configure.

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

## Calling the Azure APIs

Most operations go through the ARM REST API. The HTTP client is a thin wrapper that handles auth injection and pagination:

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

For Standard Logic Apps, some operations (like host diagnostics) call the app directly using the workflow management API rather than ARM.

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

The server exposes 23 read-only tools:

| Category | Tools |
|----------|-------|
| Discovery | `list_subscriptions`, `list_logic_apps`, `list_workflows` |
| Definitions | `get_workflow_definition`, `get_workflow_swagger`, `list_workflow_versions`, `get_workflow_version` |
| Triggers | `get_workflow_triggers`, `get_trigger_history`, `get_trigger_callback_url` |
| Run History | `list_run_history`, `search_runs`, `get_run_details`, `get_run_actions`, `get_action_io` |
| Debugging | `get_action_repetitions`, `get_scope_repetitions`, `get_action_request_history`, `get_expression_traces` |
| Connections | `get_connections`, `get_connection_details`, `test_connection` |
| Diagnostics | `get_host_status` |

## What I Learned

<div class="callout callout-tip">
<p><strong>MCP is simpler than expected.</strong> The protocol is minimal — just tool definitions and a request/response pattern over stdio. Most of the complexity is in your actual tool implementations.</p>
</div>

<div class="callout callout-tip">
<p><strong>Azure CLI auth is underrated.</strong> Reusing CLI tokens means zero credential management. Users don't need to create service principals or store secrets.</p>
</div>

<div class="callout callout-note">
<p><strong>TypeScript + Node works well for this.</strong> The <code>@modelcontextprotocol/sdk</code> handles all the protocol details. You just register handlers and implement your logic.</p>
</div>

## Try It

If you work with Logic Apps, give it a try:

```bash
npm install -g github:laveeshb/logicapps-mcp
```

Add it to Claude Desktop or VS Code Copilot, and you can start querying your Logic Apps with natural language.

The code is on [GitHub](https://github.com/laveeshb/logicapps-mcp) — contributions welcome.
