# Serverless

**Category:** Cloud / distributed · **Also known as:** FaaS (Functions as a Service), Event-triggered functions, Backend as a Service (managed-services side) · **Related:** Event-Driven, Microservices, Task Queue, Idempotent Receiver, Saga, Bulkhead

## Intent
Run code as short-lived functions that the cloud provider starts on demand in response to events (HTTP requests, queue messages, file uploads, schedules), scaling automatically, including to zero, and billing per invocation and duration instead of for idle servers.

## Problem
- Spiky or low traffic (a webhook receiver, a nightly report, an image thumbnailer) runs on servers that are idle 95% of the time but still billed and patched.
- Glue code between managed services ("when a file lands in S3, parse it and write rows to DynamoDB") needs a host, deployment pipeline and monitoring disproportionate to its size.
- Sudden bursts (marketing campaign, end-of-month batch) require capacity planning and autoscaling configuration.

Scenario: users upload receipts (images) to object storage; each upload must be OCR'd, the result stored, and the user notified. Volume ranges from 0 at night to 2,000 per minute at month end.

## Solution and structure
```
 event source ──► trigger ──► function instance (stateless handler) ──► managed services
 (HTTP, queue,     (provider       │ cold start: init runtime + your     (DB, queue, storage,
  storage, cron)   scales N)       │ module-level code once per instance   other functions)
                                   └ warm: reuse instance, run handler only
```
- **Function**: a stateless handler `(event, context) -> result`; module-level code runs once per instance and is reused across warm invocations (put clients and config there).
- **Trigger / event source**: API gateway or function URL, queue (SQS, Pub/Sub), stream (Kinesis, Kafka), storage events, scheduler.
- **Managed state**: all persistent state lives in external services (DB, object storage, cache).
- **Concurrency model**: typically one request per instance at a time on AWS Lambda (on-demand mode); some platforms (Cloud Run, Azure Flex Consumption, Lambda managed-instance options) allow concurrent requests per instance. Check the current platform docs, as these options change.
- **Limits** shape the design: max duration (Lambda: 15 min), payload size, memory/CPU coupling, ephemeral disk, concurrency quotas, cold-start latency.

Serverless containers (Cloud Run, Azure Container Apps, AWS Fargate with scale-to-zero frameworks) offer a middle ground: you ship a normal web app container, the platform scales it.

## When to use
- Event-driven glue and background processing: file processing, webhooks, queue consumers, scheduled jobs.
- Spiky or unpredictable traffic, low average utilization, scale-to-zero cost savings.
- Small teams that want to avoid operating servers.
- Prototypes and APIs with modest latency requirements.

## When NOT to use
- Steady high throughput: per-invocation pricing usually exceeds reserved containers or VMs at constant load. Do the math.
- Latency-critical paths where cold starts (tens of ms to seconds depending on runtime, package size and VPC) are unacceptable and provisioned concurrency is too costly.
- Long-running or stateful work (WebSocket servers holding state, long ML training, jobs over the max duration): use containers, batch services, or a workflow engine that splits the work.
- Heavy relational DB access from thousands of concurrent instances without a connection proxy: you will exhaust DB connections (use RDS Proxy / pooled serverless drivers).
- When local reproducibility and portability matter more than ops savings; vendor-specific triggers and IAM create lock-in.

## Example — Python
AWS Lambda handler for an SQS queue of receipt uploads, using partial batch failure reporting so one bad message does not force reprocessing of the whole batch. (Requires `ReportBatchItemFailures` on the event source mapping.)
```python
import json, os
import boto3

# Module scope: runs once per cold start, reused by warm invocations.
s3 = boto3.client("s3")
table = boto3.resource("dynamodb").Table(os.environ["RECEIPTS_TABLE"])

def extract_text(image: bytes) -> str:
    return ""                                    # call an OCR service here

def process(record: dict) -> None:
    msg = json.loads(record["body"])
    receipt_id = msg["receipt_id"]
    obj = s3.get_object(Bucket=msg["bucket"], Key=msg["key"])
    text = extract_text(obj["Body"].read())
    table.put_item(                              # idempotent: same receipt_id overwrites, never duplicates
        Item={"receipt_id": receipt_id, "text": text, "status": "parsed"},
    )

def handler(event: dict, context) -> dict:
    failures = []
    for record in event["Records"]:
        try:
            process(record)
        except Exception:
            failures.append({"itemIdentifier": record["messageId"]})  # only these are retried
    return {"batchItemFailures": failures}
```
For FastAPI on Lambda, an adapter such as Mangum (or the AWS Lambda Web Adapter for containers) translates API Gateway events to ASGI; Powertools for AWS Lambda (Python) adds structured logging, idempotency and batch helpers.

## Example — TypeScript
The same consumer on Node, plus an Angular-friendly HTTP function. Types from `@types/aws-lambda`.
```ts
import type { SQSBatchResponse, SQSEvent, APIGatewayProxyHandlerV2 } from 'aws-lambda';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, GetCommand, PutCommand } from '@aws-sdk/lib-dynamodb';

const ddb = DynamoDBDocumentClient.from(new DynamoDBClient({}));   // module scope: reused when warm
const TABLE = process.env.RECEIPTS_TABLE!;

export const onReceipt = async (event: SQSEvent): Promise<SQSBatchResponse> => {
  const batchItemFailures: SQSBatchResponse['batchItemFailures'] = [];
  for (const record of event.Records) {
    try {
      const { receiptId } = JSON.parse(record.body) as { receiptId: string };
      await ddb.send(new PutCommand({ TableName: TABLE, Item: { receiptId, status: 'parsed' } }));
    } catch {
      batchItemFailures.push({ itemIdentifier: record.messageId });
    }
  }
  return { batchItemFailures };
};

// GET /receipts/{id} via API Gateway HTTP API, called from the Angular app's HttpClient
export const getReceipt: APIGatewayProxyHandlerV2 = async (event) => {
  const id = event.pathParameters?.id;
  if (!id) return { statusCode: 400, body: 'missing id' };
  const { Item } = await ddb.send(new GetCommand({ TableName: TABLE, Key: { receiptId: id } }));
  return Item ? { statusCode: 200, body: JSON.stringify(Item) } : { statusCode: 404, body: 'not found' };
};
```
An Angular SPA itself is often hosted "serverlessly" as static files on object storage + CDN (S3 + CloudFront, Firebase Hosting, Vercel/Netlify), with SSR optionally running in functions.

## In the wild
- FaaS: AWS Lambda, Azure Functions, Google Cloud Run functions, Cloudflare Workers (V8 isolates, near-zero cold start, different runtime APIs), Vercel/Netlify Functions, Deno Deploy.
- Serverless containers: Google Cloud Run, Azure Container Apps, AWS App Runner / Fargate.
- Orchestration of functions: AWS Step Functions, Azure Durable Functions, Temporal.
- Tooling: AWS SAM, Serverless Framework, SST, CDK, Terraform; Powertools for AWS Lambda (Python, TypeScript, Java, .NET).
- Managed "serverless" data: DynamoDB, Aurora Serverless v2, Neon, Firestore, Upstash Redis.

## Trade-offs
| Pros | Cons |
|---|---|
| No servers to patch; automatic scaling, including to zero | Cold starts and per-platform limits (duration, payload, memory) |
| Pay per use; cheap at low or spiky load | Expensive at steady high load |
| Fine-grained IAM and deployment per function | Vendor lock-in through triggers, IAM, tooling |
| Natural fit for event-driven integration | Harder local testing, debugging and distributed tracing |
| Isolation per function limits blast radius | Concurrency spikes can overwhelm downstream DBs and APIs |

## Pitfalls
- **Not idempotent.** Triggers deliver at least once and retry on errors; duplicate side effects follow. Use idempotency keys (Powertools idempotency utility) and conditional writes.
- **Whole-batch failure** on queue triggers when one record fails, causing reprocessing loops. Report partial batch failures and configure a dead-letter queue.
- **Creating clients inside the handler** on every call: slow and exhausts connections. Create them at module scope.
- **Connection storms** against relational databases. Use RDS Proxy, HTTP-based drivers, or limit reserved concurrency (a bulkhead).
- **Recursive triggers** (a function writes to the bucket that triggers it) causing runaway costs. Separate input and output locations; AWS now detects some loops, but do not rely on it.
- **Huge deployment packages** inflate cold starts. Trim dependencies, use layers carefully, consider SnapStart/provisioned concurrency where supported (support varies by runtime; check docs).
- **Treating `/tmp` or globals as durable state.** Instances are recycled at any time.
- **Ignoring timeouts**: the function's timeout must exceed downstream timeouts, and the queue visibility timeout must exceed the function timeout.

## Refactoring path
- **Introduce it:** pick an isolated, event-driven piece (a cron job, a webhook, a file processor). Extract its logic into a pure function with no framework dependencies, wrap it in a thin handler, make it idempotent, add a DLQ and alarms. Keep the core logic testable without the cloud.
- **Remove it / simplify:** when load becomes steady and high, or limits get in the way, move the same handler logic into a container service (Cloud Run, ECS, Kubernetes) behind the same queue or HTTP route. Keeping business logic separate from the handler makes this move cheap.

## Related patterns
- [Event-Driven](../architectural/event-driven.md): the architectural style serverless functions plug into.
- [Microservices](../architectural/microservices.md): functions are often the deployment unit of a small service.
- [Task Queue](../emergent/task-queue.md): queue-triggered functions are a managed task queue worker.
- [Idempotent Receiver](../emergent/idempotent-receiver.md): required because triggers deliver at least once.
- [Saga](../cloud/saga.md): multi-function business flows, often orchestrated by Step Functions.
- [Bulkhead](../cloud/bulkhead.md): reserved concurrency per function isolates and protects downstreams.
