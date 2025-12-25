---
layout: post
title: "Azure Functions Extension for Amazon SQS"
date: 2025-12-24
categories: [azure, aws]
tags: [azure-functions, sqs, dotnet, serverless]
excerpt: "An Azure Functions extension that lets you trigger functions from SQS queues and send messages back."
---

If you're on <a href="https://learn.microsoft.com/azure/azure-functions/functions-overview" target="_blank">Azure Functions</a> but have data sitting in <a href="https://aws.amazon.com/sqs/" target="_blank">Amazon SQS</a> queues, this <a href="https://github.com/laveeshb/azure-functions-sqs-extension" target="_blank">extension</a> bridges that gap. It provides native bindings for SQS — trigger functions when messages arrive, send messages out, all with familiar Azure Functions patterns.

**Contents:**
- [Why This Exists](#why-this-exists)
- [How It Works](#how-it-works)
- [Two Flavors](#two-flavors)
- [Getting Started](#getting-started)
- [Authentication](#authentication)
- [Tuning the Listener](#tuning-the-listener)
- [Under the Hood](#under-the-hood)
- [Local Development](#local-development)
- [Source Code](#source-code)

## Why This Exists

Real-world systems span services and clouds — data doesn't always live where your compute does. You might have:

- Legacy systems that push to SQS
- Third-party integrations that only support AWS
- Partner data landing in SQS queues
- Teams using different cloud providers

Whatever the reason, if your compute is on Azure Functions and your messages are in SQS, you need a way to connect them.

## How It Works

The extension polls SQS using long polling (20-second wait), receives messages, and triggers your function. If your function succeeds, the message is deleted. If it fails, the message becomes visible again after the visibility timeout — built-in retry for free.

## Two Flavors

Azure Functions has two hosting models, and this extension supports both:

| Model             | Package                                                                                                                                       | Status                          |
|-------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------|
| In-Process        | <a href="https://www.nuget.org/packages/Extensions.Azure.WebJobs.SQS" target="_blank">Extensions.Azure.WebJobs.SQS</a>                        | Supported (retiring Nov 2026)   |
| Isolated Worker   | <a href="https://www.nuget.org/packages/Extensions.Azure.Functions.Worker.SQS" target="_blank">Extensions.Azure.Functions.Worker.SQS</a>      | Recommended                     |

New projects should use the isolated worker model. It runs in a separate process, supports any .NET version, and is the future of Azure Functions.

## Getting Started

### Isolated Worker (Recommended)

See a complete example in the <a href="https://github.com/laveeshb/azure-functions-sqs-extension/tree/main/dotnet/test/Extensions.SQS.Test.Isolated" target="_blank">sample project</a>.

Add the NuGet package:

```bash
dotnet add package Extensions.Azure.Functions.Worker.SQS
```

Register the extension in `Program.cs`:

```csharp
var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults(worker =>
    {
        worker.UseSqs();
    })
    .Build();

host.Run();
```

Write a function:

```csharp
public class SqsFunction
{
    private readonly ILogger<SqsFunction> _logger;

    public SqsFunction(ILogger<SqsFunction> logger)
    {
        _logger = logger;
    }

    [Function("ProcessMessage")]
    public void Run([SqsTrigger("%SQS_QUEUE_URL%")] Message message)
    {
        _logger.LogInformation("Received: {MessageId}", message.MessageId);
        _logger.LogInformation("Body: {Body}", message.Body);

        // Process the message...
        // If this method completes without exception, the message is deleted
    }
}
```

### In-Process Model

See a complete example in the <a href="https://github.com/laveeshb/azure-functions-sqs-extension/tree/main/dotnet/test/Extensions.SQS.Test.InProcess" target="_blank">sample project</a>.

Add the NuGet package:

```bash
dotnet add package Extensions.Azure.WebJobs.SQS
```

The extension auto-registers via WebJobs startup. Just write your function:

```csharp
public class SqsFunction
{
    [FunctionName("ProcessMessage")]
    public void Run(
        [SqsQueueTrigger(QueueUrl = "%SQS_QUEUE_URL%")] Message message,
        ILogger log)
    {
        log.LogInformation("Received: {Body}", message.Body);
    }
}
```

### Sending Messages

For the in-process model, there's an output binding:

```csharp
[FunctionName("SendMessage")]
public async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req,
    [SqsQueueOut(QueueUrl = "%SQS_QUEUE_URL%")] IAsyncCollector<SqsQueueMessage> messages)
{
    await messages.AddAsync(new SqsQueueMessage
    {
        Body = "Hello from Azure Functions!",
        MessageAttributes = new Dictionary<string, MessageAttributeValue>
        {
            ["Source"] = new MessageAttributeValue
            {
                DataType = "String",
                StringValue = "AzureFunctions"
            }
        }
    });

    return new OkObjectResult("Message sent");
}
```

For isolated worker, use the output binding on the return type:

```csharp
[Function("SendMessage")]
[SqsOutput("%SQS_QUEUE_URL%")]
public string Run([HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req)
{
    return "Hello from Azure Functions!";
}
```

For more control (batch sends, message attributes, delays), use the AWS SDK directly:

```csharp
[Function("SendMessageWithAttributes")]
public async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req,
    [FromServices] IAmazonSQS sqsClient)
{
    await sqsClient.SendMessageAsync(new SendMessageRequest
    {
        QueueUrl = Environment.GetEnvironmentVariable("SQS_QUEUE_URL"),
        MessageBody = "Hello!",
        DelaySeconds = 5,
        MessageAttributes = new Dictionary<string, MessageAttributeValue>
        {
            ["Source"] = new MessageAttributeValue
            {
                DataType = "String",
                StringValue = "AzureFunctions"
            }
        }
    });

    return new OkResult();
}
```

### Configuration

Set your queue URL in `local.settings.json`:

```json
{
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "SQS_QUEUE_URL": "https://sqs.us-east-1.amazonaws.com/123456789/my-queue"
  }
}
```

## Authentication

The extension uses the <a href="https://docs.aws.amazon.com/sdk-for-net/v3/developer-guide/creds-assign.html" target="_blank">AWS SDK credential chain</a>. If you don't provide explicit credentials, the SDK looks for them in this order:

1. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
2. Shared credentials file (`~/.aws/credentials`)
3. IAM role for EC2/ECS/EKS

### Local Development

For local development, set environment variables in `local.settings.json`:

```json
{
  "Values": {
    "AWS_ACCESS_KEY_ID": "your-access-key",
    "AWS_SECRET_ACCESS_KEY": "your-secret-key",
    "AWS_REGION": "us-east-1"
  }
}
```

Or use the AWS CLI to configure a profile:

```bash
aws configure
```

The SDK will pick up credentials from `~/.aws/credentials` automatically.

### Workload Identity Federation (Recommended for Production)

The best approach for Azure-hosted functions is <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html" target="_blank">OIDC federation</a> between Azure AD and AWS. No stored credentials — your function uses its Azure Managed Identity to assume an AWS IAM role.

1. Create an OIDC Identity Provider in AWS IAM that trusts Azure AD
2. Create an IAM role with a trust policy for your Azure Function's managed identity
3. Use `AssumeRoleWithWebIdentity` to get temporary AWS credentials

This eliminates credential management entirely. See <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html" target="_blank">AWS docs</a> for setup details.

### Azure Key Vault (Alternative)

If federation isn't an option, store your AWS credentials in <a href="https://learn.microsoft.com/azure/app-service/app-service-key-vault-references" target="_blank">Azure Key Vault</a> and reference them in app settings:

```
AWS_ACCESS_KEY_ID=@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/aws-key-id/)
AWS_SECRET_ACCESS_KEY=@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/aws-secret-key/)
```

This keeps secrets out of your code, though you'll need to manage rotation.

### IAM Roles (Self-Hosted on AWS)

If you're self-hosting Azure Functions on AWS infrastructure (EC2, ECS, EKS), you can use IAM roles instead of managing credentials.

**EC2**: Attach an IAM role with `sqs:ReceiveMessage`, `sqs:DeleteMessage`, and `sqs:GetQueueAttributes` permissions.

**ECS/Fargate**: Use a <a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html" target="_blank">task IAM role</a>.

**EKS/Kubernetes**: Use <a href="https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html" target="_blank">IAM Roles for Service Accounts (IRSA)</a>.

Example IAM policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:my-queue"
    }
  ]
}
```

## Tuning the Listener

Configure polling behavior in `host.json`:

```json
{
  "version": "2.0",
  "extensions": {
    "sqsQueue": {
      "maxNumberOfMessages": 10,
      "pollingInterval": "00:00:05",
      "visibilityTimeout": "00:00:30"
    }
  }
}
```

| Setting                 | Default      | Description                                          |
|-------------------------|--------------|------------------------------------------------------|
| `maxNumberOfMessages`   | `1`          | Number of messages to retrieve per poll (1-10)       |
| `pollingInterval`       | `00:00:05`   | Delay between polls when the queue is empty          |
| `visibilityTimeout`     | `00:00:30`   | How long a message stays hidden while being processed |

## Under the Hood

The listener uses a sequential polling loop rather than a timer:

```
┌─────────────────────────────────────────────────────────────────┐
│                         Polling Loop                            │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  ReceiveMessage     │
                    │  (long poll, 20s)   │
                    └─────────────────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
              ▼                                 ▼
     ┌─────────────────┐              ┌─────────────────┐
     │  Messages?  Yes │              │  Messages?  No  │
     └─────────────────┘              └─────────────────┘
              │                                 │
              ▼                                 ▼
     ┌─────────────────┐              ┌─────────────────┐
     │ Execute Function│              │ Wait interval   │
     └─────────────────┘              │ (5s default)    │
              │                       └─────────────────┘
              │                                 │
      ┌───────┴───────┐                        │
      │               │                        │
      ▼               ▼                        │
┌──────────┐   ┌──────────┐                    │
│ Success  │   │ Failure  │                    │
└──────────┘   └──────────┘                    │
      │               │                        │
      ▼               ▼                        │
┌──────────┐   ┌──────────────┐                │
│ Delete   │   │ Leave in     │                │
│ message  │   │ queue (retry)│                │
└──────────┘   └──────────────┘                │
      │               │                        │
      └───────┬───────┘                        │
              │                                │
              └────────────────┬───────────────┘
                               │
                               ▼
                         (repeat loop)
```

```csharp
while (_isRunning && !cancellationToken.IsCancellationRequested)
{
    var messagesReceived = await PollAndProcessAsync(cancellationToken);

    if (messagesReceived == 0)
        await Task.Delay(_options.PollingInterval, cancellationToken);
}
```

This avoids overlapping polls that can happen with timer-based approaches. The full implementation is in <a href="https://github.com/laveeshb/azure-functions-sqs-extension/blob/main/dotnet/src/Azure.WebJobs.Extensions.SQS/Trigger/SqsQueueTriggerListener.cs" target="_blank">SqsQueueTriggerListener.cs</a>.

Long polling (20-second wait time) reduces API calls significantly — instead of hitting SQS every second, the SDK waits for messages to arrive or the timeout to expire.

## Local Development

The extension works with <a href="https://localstack.cloud/" target="_blank">LocalStack</a> for local development without real AWS credentials. Setup scripts are in the repo:

```bash
# Unix/Mac
./dotnet/localstack/unix/setup-localstack.sh

# Windows (PowerShell)
.\dotnet\localstack\windows\setup-localstack.ps1
```

This starts LocalStack and creates test queues. Configure your `local.settings.json`:

```json
{
  "Values": {
    "AWS_ACCESS_KEY_ID": "test",
    "AWS_SECRET_ACCESS_KEY": "test",
    "AWS_REGION": "us-east-1",
    "AWS_ENDPOINT_URL": "http://localhost:4566",
    "AWS:ServiceURL": "http://localhost:4566",
    "SQS_QUEUE_URL": "http://localhost:4566/000000000000/test-queue"
  }
}
```

The `AWS:ServiceURL` setting is needed for the in-process model to connect to LocalStack. See the <a href="https://github.com/laveeshb/azure-functions-sqs-extension/tree/main/dotnet/localstack" target="_blank">LocalStack guide</a> in the repo for detailed setup instructions.

## Source Code

The extension is <a href="https://github.com/laveeshb/azure-functions-sqs-extension" target="_blank">open source on GitHub</a>. Key files:

- <a href="https://github.com/laveeshb/azure-functions-sqs-extension/tree/main/dotnet/src/Azure.WebJobs.Extensions.SQS" target="_blank">In-process extension</a>
- <a href="https://github.com/laveeshb/azure-functions-sqs-extension/tree/main/dotnet/src/Azure.Functions.Worker.Extensions.SQS" target="_blank">Isolated worker extension</a>
- <a href="https://github.com/laveeshb/azure-functions-sqs-extension/tree/main/dotnet/test" target="_blank">Sample functions</a>

Contributions welcome.
