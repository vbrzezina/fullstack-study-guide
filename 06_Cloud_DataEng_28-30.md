# Cloud & Data Engineering (28-30)
### Cloud & Serverless, AWS Services, Data Engineering & ETL

---

# 28. Cloud & Serverless

## 28.1 What "Serverless" Actually Means

"Serverless" is a misnomer — there are still servers, you just don't provision, patch, or scale them. The provider runs the infrastructure and bills you only for actual usage (per request and per millisecond of compute), and capacity scales automatically from zero to thousands of concurrent executions without you configuring anything. The defining characteristics:

- **No server management** — no instances to size, patch, or keep alive.
- **Scale-to-zero** — when there's no traffic, there's nothing running and nothing to pay for. Contrast a VM or container that costs money 24/7 whether or not it's handling requests.
- **Event-driven** — functions are triggered by events (an HTTP request, a file landing in S3, a message on a queue, a cron schedule) rather than running a long-lived process.
- **Pay-per-use, fine-grained** — billed per invocation and execution time, not per provisioned hour.

Serverless spans two layers. **FaaS (Functions as a Service)** — AWS Lambda, Azure Functions, Google Cloud Functions, Cloudflare Workers — is the compute model: your code runs in short-lived, stateless function instances. **BaaS (Backend as a Service)** — managed databases (DynamoDB, Aurora Serverless), auth (Cognito), storage (S3), messaging (SQS/SNS) — provides the surrounding building blocks so you assemble an application mostly from managed pieces. A "serverless architecture" usually combines both: Lambda functions glued together by managed services.

## 28.2 Advantages

- **No capacity planning** — the platform handles scaling, including sudden spikes, automatically.
- **Cost efficiency for spiky/low traffic** — you pay nothing when idle. Ideal for unpredictable workloads, internal tools, cron jobs, and event processing.
- **Faster time to market** — no infrastructure to stand up; deploy a function and it's live behind an endpoint.
- **Reduced operational burden** — patching, OS maintenance, and availability are the provider's problem.
- **Built-in fault isolation** — each invocation is independent; one failing request doesn't take down a shared process.

## 28.3 Drawbacks and When NOT to Use It

Serverless is not a default-good choice — its trade-offs are real and interviewers expect you to name them.

- **Cold starts** — when a function hasn't run recently, the platform must spin up a new execution environment (load the runtime, your code, init connections), adding latency from tens of milliseconds to several seconds (worst for JVM/.NET, large bundles, and VPC-attached functions). Mitigations: smaller bundles, provisioned concurrency, lighter runtimes. A problem for latency-critical synchronous paths.
- **Statelessness** — functions keep no reliable in-memory state between invocations, so anything stateful (sessions, connection pools, caches) must live in external services. This forces a particular architecture and complicates things like database connection management (a flood of concurrent functions can exhaust DB connections — hence RDS Proxy / connection pooling).
- **Execution limits** — caps on duration (Lambda: 15 min), memory, and payload size make serverless unsuitable for long-running jobs, heavy batch processing, or anything needing persistent connections (raw WebSocket servers, though API Gateway has a managed WebSocket mode).
- **Vendor lock-in** — your code is wired to a specific provider's triggers, IAM, and managed services; portability is limited. (OTel, containers-on-Lambda, and frameworks like the Serverless Framework reduce but don't eliminate this.)
- **Observability and local development** — debugging distributed functions is harder than attaching to one process; local emulation is imperfect. You lean heavily on tracing (X-Ray/OTel).
- **Cost crossover at high, steady load** — pay-per-use is cheap when idle but *more* expensive than a reserved instance under constant heavy traffic. A service pinned at high utilization 24/7 is often cheaper on a container/VM.

**Don't go serverless when:** you have steady high-volume traffic (containers are cheaper), need consistent ultra-low latency (cold starts hurt), run long or compute-heavy jobs (hit execution limits), or require fine OS/runtime control. **Do go serverless when:** traffic is spiky or unpredictable, the workload is event-driven or scheduled, you want minimal ops, or you're prototyping and want speed.

## 28.4 Serverless vs Containers vs VMs vs PaaS

These are points on a spectrum from "manage everything" to "manage nothing," trading control for operational burden.

| | VM (EC2) | Containers (ECS/EKS) | PaaS (Beanstalk/Heroku) | Serverless (Lambda) |
|---|---|---|---|---|
| **You manage** | OS, runtime, app, scaling | Container + orchestration config | Just the app + config | Just the function code |
| **Scaling** | Manual / ASG | You configure (HPA, target tracking) | Mostly automatic | Fully automatic, to zero |
| **Idle cost** | Pay 24/7 | Pay 24/7 (unless scale-to-zero) | Pay 24/7 | $0 |
| **Cold start** | None | None (warm) | None | Yes |
| **Best for** | Full control, legacy, special runtimes | Microservices, steady load, portability | Standard web apps, fast deploy | Event-driven, spiky, glue logic |
| **Granularity** | Hours | Tasks/pods | Dynos/instances | Per-request / per-ms |

A pragmatic mental model: **VMs** when you need full control or have legacy constraints; **containers** for steady-state microservices and portability across clouds; **PaaS** when you want to ship a conventional web app without touching infra; **serverless** for event-driven workloads, bursty traffic, and the connective tissue between managed services. Mature systems mix them — e.g. containers for the always-on API, Lambda for async event handlers and cron.

---


# 29. AWS Services

AWS has hundreds of services, but a senior full-stack engineer needs fluency in roughly two dozen and the judgment to know which to reach for. This section is a grouped catalog — **what each service is, when to use it, and the gotchas** — with deeper treatment and code for the ones you'll touch most. The recurring theme is **managed over self-hosted**: AWS's value proposition is offloading undifferentiated heavy lifting (patching, scaling, replication) so you focus on your application.

> Cross-references: DynamoDB single-table design is covered in §19.6, Kinesis/SQS/SNS/EventBridge messaging semantics in §21, Elasticsearch/OpenSearch in §19.8, and observability tooling in §26.4. This section focuses on the AWS-specific framing.

## 29.1 Compute

**EC2 (Elastic Compute Cloud)** — raw virtual machines. The original AWS building block: you pick an instance type (CPU/memory/network), an AMI (OS image), and you own everything above the hypervisor. Use it when you need full control, special runtimes, or to run software that isn't container/serverless-friendly. Scale with **Auto Scaling Groups** behind a load balancer. Gotcha: you're responsible for patching, and idle instances cost money. Pricing models matter — **On-Demand** (flexible, priciest), **Reserved/Savings Plans** (commit 1–3 yrs for big discounts), **Spot** (spare capacity up to ~90% off but can be reclaimed — great for fault-tolerant batch).

**Lambda** — serverless functions (FaaS). You upload code; AWS runs it in response to triggers (API Gateway, S3, SQS, EventBridge, DynamoDB Streams, cron). No servers, scales automatically, pay per invocation + duration. The workhorse of AWS-native event-driven architectures.

```typescript
// A Lambda handler triggered by API Gateway (HTTP)
import { APIGatewayProxyHandler } from "aws-lambda";
import { DynamoDBClient, GetItemCommand } from "@aws-sdk/client-dynamodb";

const db = new DynamoDBClient({});          // ← declared OUTSIDE the handler:
                                            //   reused across warm invocations (avoid re-init per call)

export const handler: APIGatewayProxyHandler = async (event) => {
  const userId = event.pathParameters?.id;
  const { Item } = await db.send(new GetItemCommand({
    TableName: "users",
    Key: { pk: { S: `USER#${userId}` } },
  }));
  return {
    statusCode: Item ? 200 : 404,
    body: JSON.stringify(Item ?? { error: "not found" }),
  };
};
```

Lambda gotchas: **cold starts** (init outside the handler, keep bundles small, consider provisioned concurrency); **15-minute max** duration; **stateless** (no local state between calls); **concurrency × DB connections** can exhaust a relational DB — use **RDS Proxy** or DynamoDB. Configured memory also scales CPU, so bumping memory can make a function both faster *and* cheaper.

**ECS (Elastic Container Service)** — AWS's own container orchestrator. Simpler than Kubernetes and tightly integrated with AWS. Runs tasks defined in task definitions, either on EC2 (you manage the host fleet) or on **Fargate** (serverless containers — no hosts to manage). Use ECS+Fargate when you want containers without K8s complexity.

**EKS (Elastic Kubernetes Service)** — managed Kubernetes. Choose it over ECS when you want the Kubernetes ecosystem (Helm, operators, portability across clouds) or already have K8s expertise. More powerful, more operational overhead.

**Fargate** — the serverless compute engine *under* ECS/EKS: run containers without provisioning or managing servers. You specify CPU/memory per task and pay for what runs. The "serverless containers" middle ground between Lambda and managing EC2.

**Elastic Beanstalk** — PaaS. You hand it your application code (Node, Python, Java, Docker) and it provisions and manages the EC2 instances, load balancer, auto-scaling, and health checks for you. Fast path for a conventional web app when you don't want to think about infrastructure. Largely superseded by ECS/Fargate and serverless for new projects, but still common.

| Need | Reach for |
|---|---|
| Full OS control / legacy | EC2 |
| Event-driven, spiky, glue | Lambda |
| Containers, no K8s, no servers | ECS + Fargate |
| Kubernetes ecosystem / multi-cloud | EKS |
| Ship a standard web app fast | Elastic Beanstalk |

## 29.2 Storage

**S3 (Simple Storage Service)** — object storage: virtually unlimited, 11-nines durability, the default home for files, static assets, backups, logs, data-lake storage, and ML datasets. Objects live in **buckets**, addressed by key. Not a filesystem — it's a key-value store for blobs.

Key features to know: **storage classes** (Standard → Infrequent Access → Glacier/Deep Archive trade retrieval speed/cost for storage cost; **lifecycle policies** auto-transition or expire objects), **versioning** (keep object history, protect against deletes), **presigned URLs** (grant time-limited upload/download without exposing credentials — the standard pattern for letting browsers upload directly), **event notifications** (object-created events trigger Lambda/SQS/SNS — the backbone of many pipelines), **static website hosting** (often fronted by CloudFront), and **encryption** (SSE-S3/SSE-KMS at rest). Gotcha: bucket misconfiguration is a classic data-leak source — block public access by default.

```typescript
// Presigned URL: let a browser upload directly to S3, no creds exposed
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3 = new S3Client({});
const url = await getSignedUrl(
  s3,
  new PutObjectCommand({ Bucket: "uploads", Key: `user-123/avatar.png` }),
  { expiresIn: 300 },                       // ← URL valid for 5 minutes
);
// Client does: fetch(url, { method: "PUT", body: file })
```

**EBS** (block storage / virtual disks attached to EC2) and **EFS** (managed elastic NFS shared across instances) round out storage — know they exist: EBS for a single instance's disk, EFS for shared file access across many instances.

## 29.3 Networking & Edge

**Route 53** — managed DNS plus domain registration. Beyond basic records, it does **health-check-based failover**, **latency-based** and **geolocation routing**, and **weighted** routing (useful for canary/blue-green at the DNS level). It's how you point a domain at a CloudFront distribution, load balancer, or S3 site.

**CloudFront** — global CDN. Caches content at edge locations close to users to cut latency, and offloads origin traffic. Fronts S3 (static sites/assets), load balancers, and APIs. Supports **Lambda@Edge / CloudFront Functions** for running logic at the edge (auth, redirects, header manipulation), TLS termination, and DDoS protection (with AWS Shield/WAF). Gotcha: caching means you must manage **invalidations** when content changes.

**API Gateway** — managed front door for APIs. Terminates HTTPS, routes requests to Lambda/HTTP/other AWS services, and handles **authentication** (Cognito, Lambda authorizers, IAM), **throttling/rate limiting**, **request/response validation/transformation**, API keys, and usage plans. Two main flavors: **REST API** (feature-rich, more expensive) and **HTTP API** (cheaper, lower latency, fewer features) — plus a **WebSocket** mode for managed real-time. It's the standard pairing with Lambda to build serverless HTTP APIs.

## 29.4 Data

**RDS / Aurora** — managed relational databases. **RDS** runs managed PostgreSQL, MySQL, MariaDB, SQL Server, or Oracle (automated backups, patching, multi-AZ failover, read replicas) — you get a standard engine without operating it. **Aurora** is AWS's cloud-native reimplementation of MySQL/PostgreSQL with a distributed storage layer: higher performance, faster failover, storage that auto-grows, and up to 15 read replicas. **Aurora Serverless v2** auto-scales capacity with load (fits the serverless story). Choose RDS for standard engine compatibility/cost, Aurora for scale and availability.

**DynamoDB** — serverless NoSQL key-value/document store with single-digit-millisecond latency at any scale. The default database for AWS-native serverless apps. (Deep dive incl. single-table design in **§19.6**.) Pairs naturally with Lambda; **DynamoDB Streams** emit change events to trigger functions.

**ElastiCache** — managed Redis or Memcached for caching and low-latency data (sessions, leaderboards, rate-limiting). The managed version of the Redis patterns in §19.4 / §34.7.

## 29.5 Application Integration (messaging & orchestration)

These wire services together asynchronously. Messaging semantics are in **§21**; here's the AWS framing and when to pick which.

- **SQS** — managed queue. Decouple producers from consumers; absorb spikes; background jobs. Standard vs FIFO. (See §21.4.)
- **SNS** — managed pub/sub. Fan-out one message to many subscribers (SQS queues, Lambda, HTTP, email/SMS). (See §21.4.)
- **EventBridge** — event bus with content-based routing rules, a schema registry, and SaaS integrations. The modern choice for routed, event-driven architectures. (See §21.6.)
- **Kinesis** — streaming log for high-volume real-time data (clickstreams, telemetry, ingestion to a data lake via Firehose). (See §21.5.)
- **Step Functions** — managed **workflow orchestration**. Define a state machine (as JSON/ASL) that coordinates multiple Lambdas/services with built-in retries, error handling, parallel branches, waits, and human-approval steps. Use it when a business process spans several steps that need reliable sequencing and visibility (order fulfillment, ETL orchestration, saga coordination) — instead of chaining Lambdas by hand. Two types: **Standard** (long-running, exactly-once, audited) and **Express** (high-volume, short, cheaper).

```json
// Step Functions state machine (Amazon States Language) — sketch
{
  "StartAt": "ChargeCard",
  "States": {
    "ChargeCard": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:chargeCard",
      "Retry": [{ "ErrorEquals": ["TransientError"], "MaxAttempts": 3 }],
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "RefundAndFail" }],
      "Next": "ReserveInventory"
    },
    "ReserveInventory": { "Type": "Task", "Resource": "arn:...:reserve", "Next": "Done" },
    "RefundAndFail": { "Type": "Task", "Resource": "arn:...:refund", "Next": "Done" },
    "Done": { "Type": "Succeed" }
  }
}
```

**Choosing among them:** point-to-point work → **SQS**; broadcast → **SNS**; routed/filtered events + integrations → **EventBridge**; high-volume streaming/replay → **Kinesis**; multi-step coordinated workflow → **Step Functions**.

## 29.6 Security & Configuration

**Secrets Manager** — stores and **automatically rotates** secrets (DB credentials, API keys), with fine-grained IAM access and encryption via KMS. Use it for sensitive credentials that should rotate.

**Parameter Store (SSM)** — stores configuration and secrets too, cheaper and simpler than Secrets Manager but without built-in rotation. Rule of thumb: **Parameter Store for config**, **Secrets Manager for rotating credentials**.

**IAM (Identity and Access Management)** — the permission system underpinning all of AWS: **users, roles, and policies** (JSON documents granting/denying actions on resources). The principle that matters: **least privilege** — grant only what's needed. Services assume **roles** to call other services (a Lambda's execution role grants it DynamoDB access) rather than embedding credentials.

**KMS (Key Management Service)** — managed encryption keys used by S3, EBS, RDS, Secrets Manager, etc. for encryption at rest. You rarely call it directly; you reference keys other services use.

## 29.7 Search

**OpenSearch Service** — managed OpenSearch/Elasticsearch for full-text search and log analytics (the AWS-managed version of the engine in **§19.8**). **CloudSearch** is AWS's older, simpler managed search service — fully managed and easy but far less flexible than OpenSearch; OpenSearch is the modern default for new projects.

## 29.8 Frontend & App Services

**Amplify** — a framework + hosting platform for full-stack web/mobile apps. It scaffolds and connects backend resources (auth via Cognito, GraphQL/REST APIs, storage, functions) and provides Git-based CI/CD hosting for the frontend. Aimed at rapid development of serverless-backed apps; great for getting a full stack running fast, less favored once you outgrow its conventions.

**AppSync** — managed **GraphQL** (and Pub/Sub) service. Provides a GraphQL endpoint with resolvers that connect to DynamoDB, Lambda, RDS, OpenSearch, and HTTP sources, plus **real-time subscriptions** and offline sync. The serverless way to expose a GraphQL API on AWS without running an Apollo server. (GraphQL itself is covered in §17.2.)

## 29.9 Observability

**CloudWatch** — the native monitoring backbone: collects **logs** (Lambda/ECS/app logs), **metrics** (built-in per service + custom), **alarms** (trigger on thresholds), **dashboards**, and **EventBridge-style scheduled rules**. **CloudWatch Logs Insights** queries logs. **X-Ray** adds distributed tracing across Lambda/API Gateway/ECS. Together they're the zero-setup observability layer for AWS-native systems. (Broader APM landscape — Datadog, New Relic, Grafana — in **§26.4**.)

## 29.10 A Serverless Reference Architecture

How the pieces fit for a typical serverless API:

```
Browser
  → Route 53 (DNS)
  → CloudFront (CDN, TLS, caches static assets from S3)
  → API Gateway (HTTPS, auth via Cognito, throttling)
  → Lambda (business logic)
        → DynamoDB (primary data)        ← DynamoDB Streams → Lambda (async reactions)
        → Secrets Manager (3rd-party API keys)
  Async side:
  Lambda → EventBridge (domain events) → SQS → worker Lambda
  Long workflow: Step Functions orchestrates multiple Lambdas
  Observability: CloudWatch (logs/metrics/alarms) + X-Ray (traces)
  Static frontend: S3 + CloudFront (or Amplify Hosting)
```

This is fully managed end-to-end, scales to zero, and has no servers to patch — the canonical "serverless full-stack on AWS."

## AWS Priority Summary

| Service | Priority |
|---|---|
| **Compute** | |
| Lambda (cold starts, limits, stateless) | **Critical** |
| ECS / Fargate | **Important** |
| EC2 (+ Spot/Reserved) | **Important** |
| EKS | **Learn** |
| Elastic Beanstalk | **Know** |
| **Storage** | |
| S3 (classes, presigned URLs, events) | **Critical** |
| **Networking/Edge** | |
| API Gateway | **Critical** |
| CloudFront | **Important** |
| Route 53 | **Learn** |
| **Data** | |
| DynamoDB (→ §19.6) | **Critical** |
| RDS / Aurora | **Important** |
| ElastiCache | **Learn** |
| **Integration** | |
| SQS / SNS / EventBridge (→ §21) | **Critical** |
| Step Functions | **Important** |
| Kinesis (→ §21.5) | **Important** |
| **Security** | |
| IAM (roles, least privilege) | **Critical** |
| Secrets Manager / Parameter Store | **Important** |
| KMS | **Learn** |
| **Search/Frontend/Observability** | |
| OpenSearch / CloudSearch | **Learn** |
| AppSync / Amplify | **Know** |
| CloudWatch + X-Ray | **Important** |

---


# 30. Data Engineering & ETL

## 30.1 ETL vs ELT and Core Principles

Most non-trivial systems eventually need to move data from where it's produced (operational databases, event streams, SaaS APIs, files) to where it's analyzed (a warehouse, lake, or search index). That movement is **data engineering**, and the classic shape is **ETL: Extract → Transform → Load**.

- **Extract** — pull data from sources (DB replication/CDC, API calls, file drops, streams).
- **Transform** — clean, deduplicate, join, reshape, enrich, and conform it to the target schema.
- **Load** — write it to the destination (warehouse, lake, index).

**ETL vs ELT** is the key modern distinction. Traditional **ETL** transforms data *before* loading (on a dedicated processing tier). **ELT** loads raw data into a powerful warehouse/lake *first*, then transforms it *in place* using the warehouse's compute (e.g. SQL/dbt in Snowflake, BigQuery, Redshift). ELT has become dominant because cloud warehouses are cheap and massively parallel, raw data is preserved for re-processing, and transformations become version-controlled SQL rather than opaque pipeline code.

**Principles that separate robust pipelines from fragile ones:**
- **Idempotent & re-runnable** — re-running a job for the same period must produce the same result, not duplicates. (Use upserts/merge, partition overwrites, or dedup keys.) Pipelines fail and get retried; non-idempotent pipelines corrupt data.
- **Incremental over full reload** — process only new/changed data (by timestamp, CDC, or partition) rather than reprocessing everything each run — essential at scale.
- **Schema management** — handle schema drift deliberately. **Schema-on-write** (validate/enforce structure when loading — warehouses) vs **schema-on-read** (store raw, impose structure at query time — data lakes). 
- **Data quality & observability** — validate row counts, null rates, and constraints; alert on anomalies. Bad data silently flowing downstream is worse than a failed job.
- **Lineage** — track where each dataset came from and what transformed it, for debugging and compliance.

## 30.2 Batch vs Streaming and Storage Targets

**Batch** processing runs on a schedule over bounded chunks (hourly, nightly) — simpler, cheaper, fine when freshness of minutes/hours is acceptable (reporting, billing). **Streaming** processes events continuously as they arrive (via Kafka/Kinesis + a stream processor like Flink/Spark Streaming) — needed for real-time dashboards, fraud detection, and low-latency features, at higher complexity. Many architectures do both (the "Lambda architecture": a batch layer for accuracy + a streaming layer for freshness; or the simpler "Kappa architecture": one streaming pipeline).

**Where the data lands:**
- **Data Lake** — raw/semi-structured data stored cheaply (S3) in open formats (Parquet/ORC). Schema-on-read, flexible, ingest-anything. Risk: becomes a "data swamp" without governance.
- **Data Warehouse** — structured, modeled data optimized for analytical SQL (Redshift, Snowflake, BigQuery). Schema-on-write, fast queries, governed.
- **Lakehouse** — converges the two: warehouse-like management/ACID over lake storage (Delta Lake, Apache Iceberg, Hudi). The current direction of travel.

## 30.3 AWS Glue

AWS Glue is AWS's serverless data-integration / ETL service — the managed way to build pipelines without running Spark clusters yourself. Its components:

- **Crawlers** — scan data sources (S3, RDS, etc.), infer schema, and populate the **Glue Data Catalog**.
- **Data Catalog** — a central metadata store (databases, tables, schemas) used by Glue, Athena, Redshift Spectrum, and EMR. It's effectively a Hive-compatible metastore for your data lake — "what data exists and what shape is it."
- **Glue Jobs** — serverless **Apache Spark** (or Python shell) jobs that do the actual transform. You write PySpark/Scala (or use the visual **Glue Studio**) and Glue provisions and scales the Spark workers.
- **Glue DataBrew** — a no-code visual tool for data cleaning/normalization.

```python
# A Glue PySpark job: read raw JSON from S3, transform, write Parquet to the lake
import sys
from awsglue.context import GlueContext
from pyspark.context import SparkContext

glueContext = GlueContext(SparkContext.getOrCreate())

# Read using the Data Catalog table the crawler created
raw = glueContext.create_dynamic_frame.from_catalog(
    database="events_db", table_name="raw_clickstream"
)

df = raw.toDF().dropDuplicates(["event_id"])          # ← idempotent: dedupe on a stable key
df = df.filter(df.event_type == "purchase")           # ← transform: keep only purchases

# Write partitioned Parquet (columnar, cheap to query in Athena/Redshift)
df.write.mode("overwrite").partitionBy("dt").parquet("s3://lake/curated/purchases/")
```

Use Glue when you want serverless Spark ETL in the AWS ecosystem feeding a data lake/warehouse, with the Catalog as the connective metadata layer.

## 30.4 Apache Airflow / Amazon MWAA

Airflow is the dominant open-source **workflow orchestrator** for data pipelines. You define pipelines as **DAGs (Directed Acyclic Graphs)** in Python — nodes are **tasks** (operators), edges are dependencies — and Airflow schedules them, runs them, retries failures, backfills historical runs, and gives you a UI of every run's status. It orchestrates *other* systems (run a Glue job, then a Redshift query, then notify Slack); it's the conductor, not usually the heavy compute itself. **Amazon MWAA (Managed Workflows for Apache Airflow)** is the AWS-managed version so you don't operate the Airflow cluster.

```python
# Airflow DAG: orchestrate a daily pipeline
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

with DAG(
    "daily_sales_etl",
    schedule="0 2 * * *",                  # ← cron: 2am daily
    start_date=datetime(2024, 1, 1),
    catchup=False,
) as dag:
    extract = PythonOperator(task_id="extract", python_callable=pull_from_source)
    transform = PythonOperator(task_id="transform", python_callable=run_glue_job)
    load = PythonOperator(task_id="load", python_callable=load_to_warehouse)

    extract >> transform >> load            # ← dependency chain (the DAG edges)
```

## 30.5 Orchestration: Glue vs Airflow vs Step Functions

These overlap, and interviewers like to probe the distinction:

| | Best at | Programming model | When to choose |
|---|---|---|---|
| **AWS Glue (workflows)** | Spark-based ETL inside AWS | Spark jobs + Glue triggers | AWS-native data-lake ETL, want serverless Spark |
| **Apache Airflow / MWAA** | General data-pipeline orchestration | Python DAGs, huge operator ecosystem | Complex multi-system pipelines, scheduling/backfill, multi-cloud |
| **Step Functions** | App/workflow orchestration | JSON state machine (ASL) | Coordinating Lambdas/AWS services, event-driven workflows, sagas |

Rough guidance: **Step Functions** for application/service workflows and serverless coordination; **Airflow/MWAA** for data-engineering pipelines with rich scheduling, dependencies, and backfills; **Glue** when the work itself is Spark ETL feeding a lake/warehouse (often *invoked by* Airflow or Step Functions).

## Data Engineering Priority Summary

| Topic | Priority |
|---|---|
| ETL vs ELT | **Critical** |
| Idempotent / incremental pipelines | **Critical** |
| Batch vs streaming | **Important** |
| Data lake / warehouse / lakehouse | **Important** |
| Schema-on-read vs schema-on-write | **Learn** |
| AWS Glue (crawlers, catalog, jobs) | **Important** |
| Airflow / MWAA (DAGs) | **Important** |
| Glue vs Airflow vs Step Functions | **Deep** |

---

---

_End of Part 6. Continue to **Part 7** (Ways of Working) in [`07_WaysOfWorking_31-33.md`](./07_WaysOfWorking_31-33.md), or return to the [README](./README.md)._
