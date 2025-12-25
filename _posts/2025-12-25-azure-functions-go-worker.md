---
layout: post
title: "Building a Native Go Worker for Azure Functions"
date: 2025-12-25
order: 5
categories: [azure]
tags: [azure-functions, go, grpc, serverless]
excerpt: "A native Go language worker that brings first-class Go support to Azure Functions via gRPC."
---

I built a <a href="https://github.com/laveeshb/azure-functions-go-worker" target="_blank">native Go worker</a> for <a href="https://learn.microsoft.com/azure/azure-functions/functions-overview" target="_blank">Azure Functions</a>. It's an out-of-process worker that speaks the same gRPC protocol as the official Python, Node.js, and Java workers.

Here's how it works and why it matters.

**Contents:**
- [How It Differs from Custom Handlers](#how-it-differs-from-custom-handlers)
- [The Architecture](#the-architecture)
- [The Developer Experience](#the-developer-experience)
- [How Functions Get Executed](#how-functions-get-executed)
- [Panic Recovery](#panic-recovery)
- [Try It](#try-it)

## How It Differs from Custom Handlers

Custom Handlers are a supported way to run any language in Azure Functions. The host starts your binary and forwards requests over HTTP. It works well and requires minimal setup.

A native language worker takes a different approach — it connects to the host via gRPC bidirectional streaming, the same protocol used by the official Python, Node.js, and Java workers. One persistent connection handles function loading, invocations, logging, and status checks.

| Aspect | Custom Handlers | Native Worker |
|--------|-----------------|---------------|
| Protocol | HTTP per request | gRPC streaming |
| Connection | Per invocation | Single persistent connection |
| Setup | Minimal | Requires gRPC implementation |
| Logging | Separate | Integrated via gRPC |

## The Architecture

The worker runs as a separate process and communicates with the Azure Functions Host over gRPC:

```
Azure Functions Host <──gRPC──> Go Worker <──> Your Functions
```

The host sends messages (init, load function, invoke), and the worker responds. All over a single bidirectional stream.

The core components:

```
cmd/worker/          # Entry point
pkg/azfunc/          # Public SDK (what developers use)
internal/
  ├── rpc/           # gRPC client and message handlers
  ├── registry/      # Function registration and execution
  └── bindings/      # Type conversions (protobuf ↔ Go types)
```

The gRPC client establishes the connection and routes messages:

```go
func (c *Client) processMessages(stream rpc.FunctionRpc_EventStreamClient) error {
    for {
        msg, err := stream.Recv()
        if err != nil {
            return err
        }

        var response *rpc.StreamingMessage
        switch content := msg.Content.(type) {
        case *rpc.StreamingMessage_WorkerInitRequest:
            response = c.handlers.HandleWorkerInit(msg.RequestId, content.WorkerInitRequest)
        case *rpc.StreamingMessage_FunctionLoadRequest:
            response = c.handlers.HandleFunctionLoad(msg.RequestId, content.FunctionLoadRequest)
        case *rpc.StreamingMessage_InvocationRequest:
            response = c.handlers.HandleInvocation(msg.RequestId, content.InvocationRequest)
        }

        if response != nil {
            stream.Send(response)
        }
    }
}
```

## The Developer Experience

Writing a function looks like this:

```go
package main

import (
    "github.com/laveeshb/azure-functions-go-worker/pkg/azfunc"
)

func init() {
    azfunc.RegisterHttpFunction("hello", helloHandler)
}

func helloHandler(ctx *azfunc.Context, req *azfunc.HttpRequest) (*azfunc.HttpResponse, error) {
    name := req.Query.Get("name")
    if name == "" {
        name = "World"
    }

    ctx.Log("Processing request for: %s", name)

    return azfunc.OK(map[string]string{
        "message": "Hello, " + name + "!",
    }), nil
}

func main() {
    azfunc.Start()
}
```

Functions register themselves in `init()`, which runs before `main()`. The SDK hides all the gRPC and protobuf complexity — developers work with familiar Go types.

Helper functions make common responses concise:

```go
return azfunc.OK(data)                    // 200
return azfunc.Created(data)               // 201
return azfunc.BadRequest("invalid input") // 400
return azfunc.NotFound("not found")       // 404
return azfunc.InternalServerError(err)    // 500
```

## How Functions Get Executed

When the host invokes a function:

1. Host sends `InvocationRequest` with trigger data
2. Worker looks up the registered handler
3. Bindings convert protobuf `TypedData` to Go types
4. Handler executes with panic recovery
5. Response converts back to protobuf
6. Worker sends `InvocationResponse`

The registry handles lookup and execution:

```go
func (r *Registry) Execute(functionID string, req *rpc.InvocationRequest) *rpc.InvocationResponse {
    r.mu.RLock()
    entry, exists := r.functions[functionID]
    r.mu.RUnlock()

    if !exists {
        return errorResponse(req.InvocationId, "function not found")
    }

    // Execute with panic recovery
    result, err := r.executeWithRecovery(entry, req)
    if err != nil {
        return errorResponse(req.InvocationId, err.Error())
    }

    return successResponse(req.InvocationId, result)
}
```

## Panic Recovery

Go panics shouldn't crash the worker. Each invocation is wrapped:

```go
func (r *Registry) executeWithRecovery(entry *functionEntry, req *rpc.InvocationRequest) (result interface{}, err error) {
    defer func() {
        if rec := recover(); rec != nil {
            stack := debug.Stack()
            err = fmt.Errorf("panic: %v\n%s", rec, stack)
        }
    }()

    return entry.handler(ctx, httpReq)
}
```

A panic in one function returns an error response for that invocation. The worker continues handling other requests.

## Try It

The <a href="https://github.com/laveeshb/azure-functions-go-worker/tree/main/samples/hello-world" target="_blank">hello-world sample</a> includes deployment scripts for Azure. Works on Windows, Mac, or Linux — the scripts cross-compile to Linux for Azure.

```bash
git clone https://github.com/laveeshb/azure-functions-go-worker.git
cd azure-functions-go-worker/samples/hello-world

# Deploy to Azure (creates resource group, storage, function app)
./deploy.sh

# Or run locally with Azure Functions Core Tools
func start
```

The code is on <a href="https://github.com/laveeshb/azure-functions-go-worker" target="_blank">GitHub</a>. Currently supports HTTP triggers, with Timer and Queue triggers on the roadmap.
