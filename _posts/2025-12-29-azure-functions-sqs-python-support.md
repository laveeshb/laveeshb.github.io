---
layout: post
title: "Adding Python Support to Azure Functions SQS Extension"
date: 2025-12-29
order: 6
categories: [azure, aws, python]
tags: [azure-functions, sqs, python, serverless]
excerpt: "The Azure Functions SQS extension now supports Python — trigger functions from SQS queues, send messages with output bindings, and batch send with collectors."
---

The <a href="https://github.com/laveeshb/azure-functions-sqs-extension" target="_blank">Azure Functions SQS Extension</a> now supports Python! You can trigger Azure Functions from Amazon SQS queues, send messages with output bindings, and batch send efficiently — all using Python's decorator-based syntax.

**Contents:**
- [What's New](#whats-new)
- [How It Works](#how-it-works)
- [Installation](#installation)
- [Authentication](#authentication)
- [SQS Trigger](#sqs-trigger)
- [SQS Output Binding](#sqs-output-binding)
- [Batch Sending with SqsCollector](#batch-sending-with-sqscollector)
- [FIFO Queue Support](#fifo-queue-support)
- [Configuration Options](#configuration-options)
- [Local Development with LocalStack](#local-development-with-localstack)
- [Samples](#samples)

## What's New

The original SQS extension supported .NET (both in-process and isolated worker models). This update adds full Python support with:

- **SQS Trigger** — Process messages from SQS queues with automatic deletion on success
- **SQS Output Binding** — Send messages to SQS as a function return value
- **SQS Collector** — Batch send multiple messages efficiently (like `IAsyncCollector` in .NET)
- **FIFO Queue Support** — Full support for FIFO queues with message groups and deduplication
- **Environment Variable Resolution** — Use `%VAR_NAME%` syntax for credentials and configuration

## How It Works

The Python extension mirrors the .NET implementation:

1. **Trigger**: Polls SQS using long polling (20-second wait), receives messages, and invokes your function
2. **Auto-delete**: If your function succeeds, the message is deleted automatically
3. **Retry**: If your function throws an exception, the message becomes visible again after the visibility timeout
4. **Output**: Return values from decorated functions are sent to SQS queues
5. **Batching**: The collector buffers messages and sends in batches of 10 (SQS limit)

## Installation

Install the Python package:

```bash
pip install azure-functions-sqs
```

Configure your queue URL in `local.settings.json`:

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "python",
    "AWS_REGION": "us-east-1",
    "SQS_QUEUE_URL": "https://sqs.us-east-1.amazonaws.com/123456789/my-queue"
  }
}
```

## Authentication

The extension uses the <a href="https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html" target="_blank">AWS default credential chain</a>, which means you can authenticate without passing credentials directly.

### IAM Roles (Recommended for Production)

For production deployments, use IAM roles — no access keys needed:

```python
from azure_functions_sqs import SqsTrigger

# No credentials specified - uses IAM role automatically
trigger = SqsTrigger(
    queue_url="%SQS_QUEUE_URL%",
    # Credentials are resolved from:
    # 1. IAM role attached to the Azure Function (via OIDC federation)
    # 2. EC2 instance profile (if running on EC2)
    # 3. ECS task role (if running in ECS)
    # 4. Environment variables (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)
)
```

To use IAM roles with Azure Functions, set up <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html" target="_blank">OIDC federation</a> between Azure AD and AWS IAM. This eliminates the need to manage long-lived credentials.

### Access Keys (For Development/Testing)

For local development or when IAM roles aren't available, you can use access keys:

```python
trigger = SqsTrigger(
    queue_url="%SQS_QUEUE_URL%",
    aws_key_id="%AWS_ACCESS_KEY_ID%",
    aws_access_key="%AWS_SECRET_ACCESS_KEY%",
)
```

Store credentials in `local.settings.json` (never commit to source control):

```json
{
  "Values": {
    "AWS_ACCESS_KEY_ID": "AKIA...",
    "AWS_SECRET_ACCESS_KEY": "..."
  }
}
```

## SQS Trigger

Process messages from an SQS queue:

```python
import json
import logging
from datetime import timedelta

from azure_functions_sqs import SqsTrigger, SqsMessage, SqsTriggerOptions

trigger = SqsTrigger(
    queue_url="%SQS_QUEUE_URL%",  # Reads from environment variable
    # Credentials resolved automatically via IAM role or environment variables
    options=SqsTriggerOptions(
        max_number_of_messages=10,  # Batch size (1-10)
        visibility_timeout=timedelta(seconds=30),  # Message lock timeout
        polling_interval=timedelta(seconds=5),  # Delay between polls
    ),
)

@trigger
def process_sqs_message(message: SqsMessage) -> None:
    logging.info(f"Processing message: {message.message_id}")
    logging.info(f"Body: {message.body}")
    
    # Access message attributes
    if message.message_attributes:
        for name, attr in message.message_attributes.items():
            logging.info(f"Attribute {name}: {attr.string_value}")
    
    # Parse JSON body
    data = json.loads(message.body)
    logging.info(f"Order ID: {data.get('order_id')}")
    
    # Message is automatically deleted after successful processing
    # If an exception is raised, message becomes visible again for retry
```

### Message Properties

The `SqsMessage` class provides access to all SQS message properties:

```python
@trigger
def process(message: SqsMessage) -> None:
    # Core properties
    print(message.message_id)
    print(message.body)
    print(message.receipt_handle)
    print(message.md5_of_body)
    
    # System attributes
    print(message.attributes.get("SentTimestamp"))
    print(message.attributes.get("ApproximateReceiveCount"))
    
    # FIFO queue properties (from attributes)
    print(message.attributes.get("MessageGroupId"))
    print(message.attributes.get("MessageDeduplicationId"))
    print(message.attributes.get("SequenceNumber"))
    
    # Custom message attributes
    for name, attr in message.message_attributes.items():
        print(f"{name}: {attr.string_value} ({attr.data_type})")
```

## SQS Output Binding

Send messages to SQS by decorating a function — the return value becomes the message body:

```python
import azure.functions as func
from azure_functions_sqs import SqsOutput
import json

app = func.FunctionApp()

output = SqsOutput(
    queue_url="%OUTPUT_QUEUE_URL%",
    # Uses IAM role or environment credentials automatically
)

@app.function_name("SendToSqs")
@app.route(route="send", methods=["POST"])
@output
def send_to_sqs(req: func.HttpRequest) -> str:
    body = req.get_json()
    
    # Return value is automatically sent to SQS
    return json.dumps({
        "timestamp": body.get("timestamp"),
        "data": body.get("data"),
        "source": "azure-function",
    })
```

## Batch Sending with SqsCollector

For sending multiple messages, use `SqsCollector` — it batches messages efficiently (similar to `IAsyncCollector` in .NET):

```python
import azure.functions as func
from azure_functions_sqs import SqsCollector
import json

app = func.FunctionApp()

@app.function_name("BatchSendToSqs")
@app.route(route="batch", methods=["POST"])
def batch_send(req: func.HttpRequest) -> func.HttpResponse:
    collector = SqsCollector(
        queue_url="%OUTPUT_QUEUE_URL%",
        # Uses IAM role or environment credentials automatically
    )
    
    items = req.get_json().get("items", [])
    
    # Add messages to the collector
    for item in items:
        collector.add({
            "item_id": item.get("id"),
            "action": item.get("action", "process"),
        })
    
    # Flush sends all messages in batches of 10 (SQS limit)
    sent_count = collector.flush()
    
    return func.HttpResponse(
        json.dumps({"messages_sent": sent_count}),
        mimetype="application/json",
    )
```

## FIFO Queue Support

For FIFO queues (queue URL ends with `.fifo`), specify a message group ID:

```python
from azure_functions_sqs import SqsOutput, SqsOutputOptions

fifo_output = SqsOutput(
    queue_url="%FIFO_QUEUE_URL%",  # Must end with .fifo
    options=SqsOutputOptions(
        message_group_id="order-processing",  # Required for FIFO queues
    ),
)

@app.route(route="fifo", methods=["POST"])
@fifo_output
def send_to_fifo(req: func.HttpRequest) -> str:
    return json.dumps(req.get_json())
```

## Configuration Options

### Trigger Options

| Option | Default | Description |
|--------|---------|-------------|
| `max_number_of_messages` | 10 | Messages to receive per poll (1-10) |
| `visibility_timeout` | 30 seconds | How long a message is hidden from other consumers |
| `polling_interval` | 5 seconds | Delay between polls when queue is empty |

### Output Options

| Option | Default | Description |
|--------|---------|-------------|
| `delay_seconds` | 0 | Delay before message becomes visible (0-900) |
| `message_group_id` | None | Required for FIFO queues |

### Credentials

The extension uses the AWS default credential chain (see [Authentication](#authentication)):

1. **IAM Role** (recommended for production) — No credentials needed, uses OIDC federation or instance profiles
2. **Environment variables** — Set `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`, reference with `%VAR_NAME%` syntax
3. **Direct values** — For local testing only (never in production)

## Local Development with LocalStack

Test locally without AWS using LocalStack:

```python
from azure_functions_sqs import SqsTrigger

trigger = SqsTrigger(
    queue_url="http://localhost:4566/000000000000/test-queue",
    region="us-east-1",  # Must specify for LocalStack
    aws_key_id="test",
    aws_access_key="test",
)
```

See the <a href="https://github.com/laveeshb/azure-functions-sqs-extension/tree/main/dotnet/localstack" target="_blank">LocalStack testing guide</a> for Docker setup instructions.

## Samples

Complete working examples are available in the repository:

| Sample | Description |
|--------|-------------|
| <a href="https://github.com/laveeshb/azure-functions-sqs-extension/blob/main/python/samples/function_app.py" target="_blank">function_app.py</a> | Full function app with trigger, output binding, batch sending, and FIFO queue examples |
| <a href="https://github.com/laveeshb/azure-functions-sqs-extension/blob/main/python/samples/host.json" target="_blank">host.json</a> | Azure Functions host configuration |
| <a href="https://github.com/laveeshb/azure-functions-sqs-extension/blob/main/python/samples/local.settings.json" target="_blank">local.settings.json</a> | Local development settings template |

The full Python implementation is in the <a href="https://github.com/laveeshb/azure-functions-sqs-extension/tree/main/python" target="_blank">python directory</a>.

Questions or feedback? Open an issue on <a href="https://github.com/laveeshb/azure-functions-sqs-extension" target="_blank">GitHub</a>!
