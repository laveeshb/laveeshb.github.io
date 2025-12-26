---
layout: post
title: "Building Azure Functions in Go with Custom Handlers"
date: 2025-12-25
order: 5
categories: [azure]
tags: [azure-functions, go, serverless, custom-handlers]
excerpt: "Write Azure Functions in Go using the Custom Handler pattern — no SDK required, just standard net/http."
---

I built sample <a href="https://github.com/laveeshb/azure-functions-go-worker" target="_blank">Azure Functions in Go</a> using the <a href="https://learn.microsoft.com/azure/azure-functions/functions-custom-handlers" target="_blank">Custom Handler</a> pattern. No special SDK, no code generation — just standard Go with `net/http`.

Here's how it works and why I chose this approach.

**Contents:**
- [Why Custom Handlers](#why-custom-handlers)
- [The Architecture](#the-architecture)
- [Writing a Function](#writing-a-function)
- [The QR Generator Sample](#the-qr-generator-sample)
- [Deploying to Azure](#deploying-to-azure)
- [Try It](#try-it)

## Why Custom Handlers

Azure Functions officially supports languages like C#, JavaScript, Python, and Java. For Go, the recommended approach is **Custom Handlers** — the host starts your HTTP server and forwards requests.

Why this works well:

- **No SDK required** — Use standard `net/http`, the same code runs anywhere
- **Simple architecture** — Just an HTTP server, nothing special
- **Easy debugging** — Test locally with `go run` or any HTTP client
- **Production ready** — Microsoft officially supports and maintains this pattern
- **Portable** — Your code isn't tied to Azure, it's just a Go web server

## The Architecture

The Azure Functions Host starts your Go binary and forwards HTTP requests:

```
Internet → Azure Functions Host → Your Go Binary (net/http server)
```

Your binary reads the port from an environment variable and serves requests (<a href="https://github.com/laveeshb/azure-functions-go-worker/blob/main/samples/hello-world/src/main.go" target="_blank">main.go</a>):

```go
func main() {
    port := os.Getenv("FUNCTIONS_CUSTOMHANDLER_PORT")
    if port == "" {
        port = "8080"
    }

    http.HandleFunc("/api/hello", helloHandler)
    http.HandleFunc("/api/health", healthHandler)

    log.Printf("Starting server on port %s", port)
    log.Fatal(http.ListenAndServe(":"+port, nil))
}
```

The <a href="https://github.com/laveeshb/azure-functions-go-worker/blob/main/samples/hello-world/src/host.json" target="_blank">host.json</a> tells Azure Functions to use your binary as a Custom Handler:

```json
{
  "version": "2.0",
  "customHandler": {
    "description": {
      "defaultExecutablePath": "handler"
    },
    "enableForwardingHttpRequest": true
  }
}
```

With `enableForwardingHttpRequest: true`, the host passes through HTTP requests directly — your Go code sees standard `http.Request` and `http.ResponseWriter`.

## Writing a Function

A simple hello function:

```go
func helloHandler(w http.ResponseWriter, r *http.Request) {
    name := r.URL.Query().Get("name")
    if name == "" {
        name = "World"
    }

    response := map[string]string{
        "message": fmt.Sprintf("Hello, %s!", name),
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}
```

Each function needs a <a href="https://github.com/laveeshb/azure-functions-go-worker/blob/main/samples/hello-world/src/Hello/function.json" target="_blank">function.json</a> that defines its trigger:

```json
{
  "bindings": [
    {
      "authLevel": "anonymous",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["get", "post"],
      "route": "hello"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "$return"
    }
  ]
}
```

That's it. Standard Go HTTP handlers, standard JSON responses.

## The QR Generator Sample

The repo includes a <a href="https://github.com/laveeshb/azure-functions-go-worker/tree/main/samples/qr-generator" target="_blank">complete QR code generator</a> with a web UI (<a href="https://github.com/laveeshb/azure-functions-go-worker/blob/main/samples/qr-generator/main.go" target="_blank">main.go</a>):

```go
func handleGenerate(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodGet {
        // Serve the web UI
        w.Header().Set("Content-Type", "text/html")
        fmt.Fprint(w, landingPageHTML)
        return
    }

    // POST: Generate QR code
    var req GenerateRequest
    json.NewDecoder(r.Body).Decode(&req)

    png, _ := qrcode.Encode(req.Content, qrcode.Medium, req.Size)

    json.NewEncoder(w).Encode(GenerateResponse{
        Image:   base64.StdEncoding.EncodeToString(png),
        Content: req.Content,
        Size:    req.Size,
    })
}
```

The web UI lets users enter text and see the generated QR code:

- `GET /generate` — Interactive web page
- `POST /generate` — API returns base64-encoded PNG
- `GET /health` — Health check endpoint

## Deploying to Azure

Each sample includes deployment scripts (<a href="https://github.com/laveeshb/azure-functions-go-worker/blob/main/samples/qr-generator/deploy.sh" target="_blank">deploy.sh</a>, <a href="https://github.com/laveeshb/azure-functions-go-worker/blob/main/samples/qr-generator/deploy.ps1" target="_blank">deploy.ps1</a>). For the QR generator:

```bash
cd samples/qr-generator

# Deploy to Azure (creates everything: resource group, storage, function app)
./deploy.sh -g my-resource-group -l westus2

# Or on Windows
.\deploy.ps1 -ResourceGroupName my-resource-group -Location westus2
```

The script:
1. Cross-compiles to Linux (`GOOS=linux GOARCH=amd64`)
2. Creates Azure resources (resource group, storage account, function app)
3. Deploys with `func azure functionapp publish`

For local development:

```bash
# Build
go build -o handler .

# Run with Azure Functions Core Tools
func start
```

## Try It

Clone the repo and try the samples:

```bash
git clone https://github.com/laveeshb/azure-functions-go-worker.git
cd azure-functions-go-worker

# Hello World - basic HTTP function
cd samples/hello-world/src
go build -o handler .
func start

# QR Generator - web UI + API
cd samples/qr-generator
go build -o handler .
func start
```

The code is on <a href="https://github.com/laveeshb/azure-functions-go-worker" target="_blank">GitHub</a>. The Custom Handler pattern works with any language — the same approach works for Rust, Swift, or anything that can serve HTTP.
