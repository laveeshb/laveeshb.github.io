---
layout: post
title: "What's New in Logic Apps MCP v0.2: Debugging Tools"
date: 2025-12-24
order: 4
categories: [azure]
tags: [mcp, logic-apps, typescript, ai, debugging]
excerpt: "New tools for searching runs, inspecting action I/O, testing connections, and accessing workflow history."
---

I just shipped v0.2.0 of the <a href="https://github.com/laveeshb/logicapps-mcp" target="_blank">Logic Apps MCP server</a> with 5 new debugging tools. The original release had 18 tools for inspecting workflows and run history. This update focuses on making troubleshooting easier.

Here's what's new.

**Contents:**
- [Search Runs Without OData](#search-runs-without-odata)
- [Get Actual Action Inputs and Outputs](#get-actual-action-inputs-and-outputs)
- [Test Connections](#test-connections)
- [Get Connection Details](#get-connection-details)
- [Access Workflow Versions](#access-workflow-versions)
- [Updated Tool Summary](#updated-tool-summary)

## Search Runs Without OData

The original `list_run_history` tool supported OData filters, but constructing the right filter string takes effort. The new `search_runs` tool exposes friendly parameters:

```typescript
{
  name: "search_runs",
  description:
    "Search run history with friendly parameters instead of raw OData filter syntax.",
  inputSchema: {
    type: "object",
    properties: {
      status: {
        type: "string",
        enum: ["Succeeded", "Failed", "Cancelled", "Running"],
        description: "Filter by run status",
      },
      startTime: {
        type: "string",
        description: "Filter runs starting after this ISO timestamp",
      },
      endTime: {
        type: "string",
        description: "Filter runs starting before this ISO timestamp",
      },
      clientTrackingId: {
        type: "string",
        description: "Filter by correlation/tracking ID",
      },
    },
  },
}
```

Now you can ask: "Find all failed runs from the last hour" and the AI doesn't need to construct `$filter=status eq 'Failed' and startTime ge '2025-12-24T10:00:00Z'`.

Under the hood, it builds the OData filter for you:

```typescript
const filterParts: string[] = [];

if (status) {
  filterParts.push(`status eq '${status}'`);
}
if (startTime) {
  filterParts.push(`startTime ge ${startTime}`);
}
if (endTime) {
  filterParts.push(`startTime le ${endTime}`);
}

const filter = filterParts.length > 0 ? filterParts.join(" and ") : undefined;
```

## Get Actual Action Inputs and Outputs

The existing `get_run_actions` tool returns action metadata, but the actual inputs and outputs are behind separate URLs (`inputsLink`, `outputsLink`). The new `get_action_io` tool follows those links:

```typescript
{
  name: "get_action_io",
  description:
    "Get the actual input/output content for a run action. Fetches the content from inputsLink/outputsLink URLs. Essential for debugging data issues.",
  inputSchema: {
    type: "object",
    properties: {
      type: {
        type: "string",
        enum: ["inputs", "outputs", "both"],
        description: "Which content to fetch (default: both)",
      },
      // ... standard params: subscriptionId, resourceGroupName, etc.
    },
  },
}
```

The implementation fetches the action metadata, then follows the content links:

```typescript
// Fetch inputs if requested
if ((type === "inputs" || type === "both") && action.properties.inputsLink?.uri) {
  result.inputs = await fetchContentLink(action.properties.inputsLink.uri);
}

// Fetch outputs if requested
if ((type === "outputs" || type === "both") && action.properties.outputsLink?.uri) {
  result.outputs = await fetchContentLink(action.properties.outputsLink.uri);
}
```

The content link URLs include SAS tokens, so no additional auth is needed. This is essential for debugging — when a workflow fails, you want to see what data was actually passed to each action, not just that the action ran.

## Test Connections

Logic Apps use API connections for external services (SQL, SharePoint, etc.). When things break, the first question is often: "Is the connection still valid?"

```typescript
{
  name: "test_connection",
  description:
    "Test if an API connection is valid and healthy. Checks connection status and attempts to validate using test links if available.",
  inputSchema: {
    type: "object",
    properties: {
      subscriptionId: { type: "string" },
      resourceGroupName: { type: "string" },
      connectionName: { type: "string" },
    },
    required: ["subscriptionId", "resourceGroupName", "connectionName"],
  },
}
```

The tool checks the connection status and, if test links are available, actually calls them:

```typescript
// If status is Connected, it's valid
if (details.status === "Connected") {
  result.isValid = true;
  return result;
}

// If there are test links, try to use them
if (details.testLinks && details.testLinks.length > 0) {
  const testLink = details.testLinks[0];
  const response = await fetch(url, {
    method: testLink.method,
    headers: { Authorization: `Bearer ${token}` },
  });

  if (response.ok) {
    result.isValid = true;
    result.status = "Connected";
  }
}
```

Quick way to rule out (or confirm) connection issues.

## Get Connection Details

Beyond testing, sometimes you need to inspect a connection's configuration:

```typescript
{
  name: "get_connection_details",
  description:
    "Get detailed information about a specific API connection including status, configuration, and test links.",
  inputSchema: {
    type: "object",
    properties: {
      subscriptionId: { type: "string" },
      resourceGroupName: { type: "string" },
      connectionName: { type: "string" },
    },
    required: ["subscriptionId", "resourceGroupName", "connectionName"],
  },
}
```

Returns the full connection metadata including API type, status, creation time, parameter values, and test links.

## Access Workflow Versions

Logic Apps (Consumption SKU) keeps a version history. The new `get_workflow_version` tool retrieves a specific version's definition:

```typescript
{
  name: "get_workflow_version",
  description:
    "Get a specific historical version's definition. Only available for Consumption Logic Apps. Use list_workflow_versions to see available versions.",
  inputSchema: {
    type: "object",
    properties: {
      subscriptionId: { type: "string" },
      resourceGroupName: { type: "string" },
      logicAppName: { type: "string" },
      versionId: { type: "string", description: "Version ID (from list_workflow_versions)" },
    },
    required: ["subscriptionId", "resourceGroupName", "logicAppName", "versionId"],
  },
}
```

Useful when you need to compare what changed between versions, or understand what the workflow looked like at a specific point in time.

## Updated Tool Summary

The server now has 23 tools:

| Category | Tools |
|----------|-------|
| Discovery | `list_subscriptions`, `list_logic_apps`, `list_workflows` |
| Definitions | `get_workflow_definition`, `get_workflow_swagger`, `list_workflow_versions`, `get_workflow_version` |
| Triggers | `get_workflow_triggers`, `get_trigger_history`, `get_trigger_callback_url` |
| Run History | `list_run_history`, `get_run_details`, `get_run_actions`, `search_runs` |
| Debugging | `get_action_io`, `get_action_repetitions`, `get_scope_repetitions`, `get_action_request_history`, `get_expression_traces` |
| Connections | `get_connections`, `get_connection_details`, `test_connection` |
| Diagnostics | `get_host_status` |

The focus for this release was debugging workflows. With these tools, an AI assistant can now help you:

- Search for problematic runs without writing OData
- Inspect what data actually flowed through each action
- Verify connections are healthy
- Compare workflow versions to spot changes

If you're already using the server, update with `npm update -g logicapps-mcp`. If not, check out the <a href="https://github.com/laveeshb/logicapps-mcp" target="_blank">repo</a> to get started.
