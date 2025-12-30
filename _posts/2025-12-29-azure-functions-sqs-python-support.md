---
layout: post
title: "Python Support for Azure Functions SQS Extension"
date: 2025-12-29
order: 6
categories: [azure, aws, python]
tags: [azure-functions, sqs, python, serverless]
excerpt: "The Azure Functions SQS extension now supports Python — trigger functions from SQS queues and send messages back using familiar decorator patterns."
---

The <a href="/2025/12/24/azure-functions-sqs-extension/">Azure Functions SQS Extension</a> now has Python support. If you're running Azure Functions but need to process messages from Amazon SQS queues, you can now do it in Python with the same patterns you'd use in .NET.

**Contents:**
- [Getting Started](#getting-started)
- [Sending Messages](#sending-messages)
- [Authentication](#authentication)
- [Tuning the Listener](#tuning-the-listener)
- [Under the Hood](#under-the-hood)
- [Local Development](#local-development)
- [Source Code](#source-code)

## Getting Started

See a complete example in the <a href="https://github.com/laveeshb/azure-functions-sqs-extension/tree/main/python/samples" target="_blank">sample project</a>.

Install the package:

```bash
pip install azure-functions-sqs
```

Write a function that triggers on SQS messages:

```python
import json
import logging

from azure_functions_sqs import SqsTrigger, SqsMessage

trigger = SqsTrigger(queue_url="%SQS_QUEUE_URL%")

@trigger
def process_message(message: SqsMessage) -> None:
    logging.info(f"Received: {message.message_id}")
    logging.info(f"Body: {message.body}")

    # Process the message...
    data = json.loads(message.body)
    logging.info(f"Order ID: {data.get('order_id')}")

    # If this function completes without exception, the message is deleted
    # If an exception is raised, message becomes visible again for retry
```

Configure your queue URL in <a href="https://github.com/laveeshb/azure-functions-sqs-extension/blob/main/python/samples/local.settings.json" target="_blank">local.settings.json</a>:

```json
{
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "python",
    "SQS_QUEUE_URL": "https://sqs.us-east-1.amazonaws.com/123456789/my-queue"
  }
}
```

The `SqsMessage` class gives you access to all SQS message properties:

```python
@trigger
def process(message: SqsMessage) -> None:
    # Core properties
    print(message.message_id)
    print(message.body)
    print(message.receipt_handle)

    # System attributes
    print(message.attributes.get("SentTimestamp"))
    print(message.attributes.get("ApproximateReceiveCount"))

    # FIFO queue properties
    print(message.attributes.get("MessageGroupId"))
    print(message.attributes.get("SequenceNumber"))

    # Custom message attributes
    for name, attr in message.message_attributes.items():
        print(f"{name}: {attr.string_value}")
```

## Sending Messages

### Output Binding

The simplest way to send messages — decorate a function and return the message body (see <a href="https://github.com/laveeshb/azure-functions-sqs-extension/blob/main/python/samples/function_app.py#L73-L100" target="_blank">full example</a>):

```python
import azure.functions as func
from azure_functions_sqs import SqsOutput
import json

app = func.FunctionApp()
output = SqsOutput(queue_url="%OUTPUT_QUEUE_URL%")

@app.function_name("SendToSqs")
@app.route(route="send", methods=["POST"])
@output
def send_to_sqs(req: func.HttpRequest) -> str:
    body = req.get_json()
    return json.dumps({
        "timestamp": body.get("timestamp"),
        "data": body.get("data"),
        "source": "azure-function",
    })
```

### Batch Sending with SqsCollector

For sending multiple messages efficiently, use `SqsCollector` — it batches messages in groups of 10 (the SQS limit), similar to `IAsyncCollector` in .NET (see <a href="https://github.com/laveeshb/azure-functions-sqs-extension/blob/main/python/samples/function_app.py#L107-L148" target="_blank">full example</a>):

```python
from azure_functions_sqs import SqsCollector

@app.function_name("BatchSendToSqs")
@app.route(route="batch", methods=["POST"])
def batch_send(req: func.HttpRequest) -> func.HttpResponse:
    collector = SqsCollector(queue_url="%OUTPUT_QUEUE_URL%")

    for item in req.get_json().get("items", []):
        collector.add({
            "item_id": item.get("id"),
            "action": item.get("action", "process"),
        })

    sent_count = collector.flush()

    return func.HttpResponse(
        json.dumps({"messages_sent": sent_count}),
        mimetype="application/json",
    )
```

### FIFO Queues

For FIFO queues (URL ends with `.fifo`), specify a message group ID (see <a href="https://github.com/laveeshb/azure-functions-sqs-extension/blob/main/python/samples/function_app.py#L155-L175" target="_blank">full example</a>):

```python
from azure_functions_sqs import SqsOutput, SqsOutputOptions

fifo_output = SqsOutput(
    queue_url="%FIFO_QUEUE_URL%",
    options=SqsOutputOptions(message_group_id="order-processing"),
)

@app.route(route="fifo", methods=["POST"])
@fifo_output
def send_to_fifo(req: func.HttpRequest) -> str:
    return json.dumps(req.get_json())
```

## Authentication

The extension uses the <a href="https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html" target="_blank">boto3 credential chain</a>. If you don't provide explicit credentials, the SDK looks for them in this order:

1. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
2. Shared credentials file (`~/.aws/credentials`)
3. IAM role for EC2/ECS/EKS

### Local Development

Set environment variables in `local.settings.json`:

```json
{
  "Values": {
    "AWS_ACCESS_KEY_ID": "your-access-key",
    "AWS_SECRET_ACCESS_KEY": "your-secret-key",
    "AWS_REGION": "us-east-1"
  }
}
```

Or configure a profile with the AWS CLI:

```bash
aws configure
```

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

## Tuning the Listener

Configure polling behavior with `SqsTriggerOptions`:

```python
from datetime import timedelta
from azure_functions_sqs import SqsTrigger, SqsTriggerOptions

trigger = SqsTrigger(
    queue_url="%SQS_QUEUE_URL%",
    options=SqsTriggerOptions(
        max_number_of_messages=10,
        visibility_timeout=timedelta(seconds=30),
        polling_interval=timedelta(seconds=5),
    ),
)
```

| Setting | Default | Description |
|---------|---------|-------------|
| `max_number_of_messages` | 10 | Messages to receive per poll (1-10) |
| `visibility_timeout` | 30 seconds | How long a message stays hidden while processing |
| `polling_interval` | 5 seconds | Delay between polls when queue is empty |

## Under the Hood

The trigger uses a sequential polling loop:

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

```python
async def _poll_loop(self):
    while self._running:
        messages_received = await self._poll_and_process()

        if messages_received == 0:
            await asyncio.sleep(self._options.polling_interval.total_seconds())
```

Long polling (20-second wait time) reduces API calls — instead of hitting SQS every second, the SDK waits for messages to arrive or the timeout to expire.

## Local Development

The extension works with <a href="https://localstack.cloud/" target="_blank">LocalStack</a> for local development without AWS credentials:

```python
trigger = SqsTrigger(
    queue_url="http://localhost:4566/000000000000/test-queue",
    region="us-east-1",
    aws_key_id="test",
    aws_access_key="test",
)
```

See the <a href="https://github.com/laveeshb/azure-functions-sqs-extension/tree/main/dotnet/localstack" target="_blank">LocalStack guide</a> for Docker setup. The same scripts work for Python.

## Source Code

The extension is <a href="https://github.com/laveeshb/azure-functions-sqs-extension" target="_blank">open source on GitHub</a>. Key files:

- <a href="https://github.com/laveeshb/azure-functions-sqs-extension/tree/main/python/azure_functions_sqs" target="_blank">Python extension source</a>
- <a href="https://github.com/laveeshb/azure-functions-sqs-extension/tree/main/python/samples" target="_blank">Sample functions</a>
- <a href="https://github.com/laveeshb/azure-functions-sqs-extension/tree/main/python/tests" target="_blank">Tests</a>
