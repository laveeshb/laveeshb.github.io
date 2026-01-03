---
layout: post
title: "What's New in Logic Apps MCP v0.3: Write Operations & Knowledge Tools"
date: 2026-01-02
order: 7
categories: [azure]
tags: [mcp, logic-apps, typescript, ai]
excerpt: "The Logic Apps MCP server goes read-write with workflow management, connector creation, and built-in knowledge for AI assistants."
---

I just shipped v0.3.0 of the <a href="https://github.com/laveeshb/logicapps-mcp" target="_blank">Logic Apps MCP server</a> with 14 new tools. The previous releases focused on reading and debugging workflows. This update adds write operations — you can now create, modify, and manage workflows through your AI assistant.

Here's what's new.

**Contents:**
- [Write Operations](#write-operations)
- [Workflow Lifecycle Management](#workflow-lifecycle-management)
- [Connector Management](#connector-management)
- [Knowledge Tools](#knowledge-tools)
- [Performance Improvements](#performance-improvements)
- [Updated Tool Summary](#updated-tool-summary)

## Write Operations

The biggest change in v0.3 is the shift from read-only to read-write. Three new tools let you manage workflow definitions:

- **`create_workflow`** — Creates a new workflow. For Consumption SKU, this creates a new Logic App resource. For Standard SKU, it adds a workflow to an existing Logic App.
- **`update_workflow`** — Modifies an existing workflow's definition. Handles the API differences between SKUs automatically.
- **`delete_workflow`** — Removes a workflow entirely. Use with caution.

Now you can ask your AI assistant things like:

- *"Create a workflow that triggers on HTTP request and sends an email"*
- *"Add a condition to check if the order total exceeds $1000"*
- *"Update the workflow to retry failed API calls"*

The AI can build the workflow definition and deploy it directly.

## Workflow Lifecycle Management

v0.3 adds two new tools for controlling workflow state, complementing the existing `run_trigger` and `cancel_run`:

- **`enable_workflow`** — Enables a workflow that was previously disabled.
- **`disable_workflow`** — Disables a workflow without deleting it. Useful for maintenance or temporarily stopping processing.
- **`run_trigger`** — Manually fires a workflow trigger for testing. No need to wait for a schedule or craft an HTTP request.
- **`cancel_run`** — Cancels a workflow run that's in `Running` or `Waiting` status. Handy when a workflow is stuck or you need to stop a long-running process.

Example prompts:

- *"Disable the order-processing workflow while I update the database"*
- *"Run the daily-report workflow now so I can test it"*
- *"Cancel all running instances of the batch-import workflow"*

## Connector Management

Logic Apps connect to external services through API connections. v0.3 adds tools to work with connectors programmatically:

- **`create_connection`** — Creates a new API connection for managed connectors like SQL Server, Service Bus, or SharePoint. Supports both OAuth and parameter-based authentication.
- **`get_connector_swagger`** — Fetches the OpenAPI spec for any managed connector, so the AI understands what operations and parameters are available.
- **`invoke_connector_operation`** — Calls dynamic connector operations like `GetTables`, `GetQueues`, or `GetSchemas` through an existing connection.

This enables conversational flows like:

- *"Connect to my SQL database and show me the available tables"*
- *"What operations does the Service Bus connector support?"*
- *"List all queues in my Service Bus namespace"*

The AI can discover schemas and structures, then use that information to build workflows that reference the right tables and fields.

## Knowledge Tools

One challenge with AI assistants is they don't always know the specifics of Logic Apps — expression syntax, connector quirks, SKU differences. v0.3 bundles documentation directly into the server through four knowledge tools:

| Tool | Purpose |
|------|---------|
| `get_troubleshooting_guide` | Common errors and how to fix them — expression failures, connection issues, run errors |
| `get_authoring_guide` | Best practices for workflow design, connector usage, and deployment patterns |
| `get_reference` | Tool catalog and SKU differences between Consumption and Standard |
| `get_workflow_instructions` | Step-by-step guides for common tasks like diagnosing failures or creating workflows |

The guides are markdown files in the <a href="https://github.com/laveeshb/logicapps-mcp/tree/main/knowledge" target="_blank">knowledge folder</a>. When you install the MCP server, these files are bundled with it — no separate download needed. When you ask *"Why is my expression failing?"*, the AI pulls the relevant troubleshooting docs and gives you targeted advice instead of generic suggestions.

## Performance Improvements

v0.3 includes several reliability improvements:

- **Retry with exponential backoff** — Transient failures (429s, 5xx errors) are automatically retried with jitter to avoid thundering herd issues.
- **SKU detection caching** — The server caches whether a Logic App is Consumption or Standard, avoiding repeated API calls.
- **Parallelized operations** — `get_connector_swagger` fetches connector metadata in parallel, reducing response time.
- **Content link limits** — Fetching action inputs/outputs has a 30-second timeout and 10MB size limit to prevent hanging on large payloads.

## Updated Tool Summary

The server now has 37 tools:

| Category | Tools |
|----------|-------|
| Discovery | `list_subscriptions`, `list_logic_apps`, `list_workflows` |
| Definitions | `get_workflow_definition`, `get_workflow_swagger`, `list_workflow_versions`, `get_workflow_version` |
| Triggers | `get_workflow_triggers`, `get_trigger_history`, `get_trigger_callback_url` |
| Run History | `list_run_history`, `get_run_details`, `get_run_actions`, `search_runs` |
| Debugging | `get_action_io`, `get_action_repetitions`, `get_scope_repetitions`, `get_action_request_history`, `get_expression_traces` |
| Connections | `get_connections`, `get_connection_details`, `test_connection`, `create_connection`, `get_connector_swagger`, `invoke_connector_operation` |
| Diagnostics | `get_host_status` |
| Write Operations | `enable_workflow`, `disable_workflow`, `run_trigger`, `cancel_run`, `create_workflow`, `update_workflow`, `delete_workflow` |
| Knowledge | `get_troubleshooting_guide`, `get_authoring_guide`, `get_reference`, `get_workflow_instructions` |

The shift to write operations opens up new possibilities. Your AI assistant can now help you not just understand and debug workflows, but build and manage them. Combined with the knowledge tools, it has the context to make better suggestions.

If you're already using the server, update with `npm update -g @laveeshb/logicapps-mcp`. If not, check out the <a href="https://github.com/laveeshb/logicapps-mcp" target="_blank">repo</a> to get started.
