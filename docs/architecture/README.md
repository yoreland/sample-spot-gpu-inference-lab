# Architecture

This document describes the runtime architecture of the sample. The diagram below is
committed as Mermaid source so it stays in version control and renders directly on GitHub.
It reflects the actual CDK stack (`cdk/lib/tpot-booking-stack.ts`) and the four Python
Lambda handlers under `cdk/lambda/`.

> Mermaid is the committed default. If you want a raster export for a slide deck or blog
> post, paste the block below into the [Mermaid Live Editor](https://mermaid.live) and
> export a PNG/SVG.

## Components at a glance

- **Static SPA** — React/Vite build served from an **S3** bucket behind **CloudFront**.
- **API** — **CloudFront** forwards `/api/*` to **API Gateway** (a CloudFront Function
  rewrites `/api/...` → `/<stage>/...`). API Gateway uses a **Lambda authorizer** doing
  HTTP Basic auth against an **SSM** SecureString parameter, then invokes the **`api`
  Lambda**.
- **State — DynamoDB (3 tables)**:
  - `TpotBookingTable` — bookings + their status history (with an
    `instanceType-status-index` GSI).
  - `TpotNotificationConfigTable` — the Feishu webhook config (`configId=global`).
  - `TpotDeploymentPlanTable` — deployment plans.
- **`poller` Lambda** — EventBridge triggers it every minute. It scans for bookings in
  `polling` state and directly launches a **Spot GPU** instance (`RunInstances` with Spot
  market options) in the first region/AZ that has capacity, using an AZ→subnet map from
  `SUBNET_MAP_CONFIG`. B300/B200/H200 use a **direct-launch** path (probe-then-launch is too
  slow for short B300 capacity windows). On success it flips the booking to `launching`
  (conditional write to avoid duplicate launches) and asynchronously invokes the deployer.
- **`deployer` Lambda** — two-phase. Phase 1 (`deploy`, invoked by the poller) waits for
  SSM reachability, then sends a long-running **SSM** command that sets up **NVMe RAID**,
  downloads the model, and runs **`docker compose` pull + up** for the selected SGLang
  topology. Phase 2 (`check_progress`, EventBridge every 2 min) polls the SSM command and
  the service health endpoint, then marks the booking `ready` or `failed`.
- **`orphan-cleaner` Lambda** — EventBridge every 15 min. Reaps stray tagged Spot instances
  that are no longer tracked by an active booking, as a cost safety net.
- **Notifications** — every state-change Lambda pushes to the **Feishu (Lark) webhook**
  (with SNS as a fallback topic). The webhook URL is read at runtime from the DynamoDB
  notification-config table.
- **Compose files** — 7 SGLang `docker-compose-*.yaml` topologies are bundled with the
  deployer and deployed to an S3 compose bucket; they can be updated in S3 without a
  redeploy (see [`../../cdk/docs/UPDATE-COMPOSE.md`](../../cdk/docs/UPDATE-COMPOSE.md)).

## Diagram

```mermaid
flowchart TB
  user([User / Browser])

  subgraph edge[Edge and static delivery]
    cf[CloudFront distribution]
    s3f[(S3: frontend SPA bucket)]
  end

  subgraph api_layer[API layer]
    apigw[API Gateway REST API]
    authz[Lambda: Basic-auth authorizer]
    ssm_auth[(SSM SecureString:<br/>basic-auth-credentials)]
    apil[Lambda: api]
  end

  subgraph state[State - DynamoDB]
    ddb_book[(TpotBookingTable<br/>bookings + history + GSI)]
    ddb_notif[(TpotNotificationConfigTable<br/>Feishu webhook config)]
    ddb_plan[(TpotDeploymentPlanTable<br/>deployment plans)]
  end

  subgraph sched[EventBridge schedules]
    eb_poll{{every 1 min}}
    eb_dep{{every 2 min: check_progress}}
    eb_orph{{every 15 min}}
  end

  poller[Lambda: poller]
  deployer[Lambda: deployer]
  orphan[Lambda: orphan-cleaner]

  subgraph compute[GPU compute]
    ec2[[EC2 Spot GPU instance<br/>H200 / B200 / B300<br/>docker-compose SGLang on NVMe]]
  end

  s3c[(S3: compose-files bucket)]
  feishu[[Feishu / Lark webhook]]

  user --> cf
  cf -->|static assets| s3f
  cf -->|/api/* rewrite to /stage| apigw
  apigw --> authz
  authz -.reads.-> ssm_auth
  apigw --> apil
  apil --> ddb_book
  apil --> ddb_notif
  apil --> ddb_plan
  apil -->|reuse on override| deployer

  eb_poll --> poller
  eb_dep --> deployer
  eb_orph --> orphan

  poller -->|scan polling bookings| ddb_book
  poller -->|RunInstances Spot<br/>probe-then-launch; B300 direct| ec2
  poller -->|async invoke| deployer

  deployer -->|SSM: NVMe RAID +<br/>docker compose up| ec2
  deployer -->|pull topology| s3c
  deployer --> ddb_book

  orphan -->|reap stray instances| ec2
  orphan --> ddb_book

  poller --> feishu
  deployer --> feishu
  orphan --> feishu
  apil --> feishu
```

## Key CloudFormation outputs

The stack emits (see `cdk/lib/tpot-booking-stack.ts`):

| Output | Meaning |
| --- | --- |
| `ApiUrl` | API Gateway base URL |
| `CloudFrontDomain` | CloudFront domain that serves the SPA and proxies `/api/*` |
| `FrontendBucketName` | S3 bucket to sync the built SPA into |
| `ComposeBucketName` | S3 bucket holding the docker-compose topologies |
| `BookingTableName` | bookings + history DynamoDB table |
| `NotificationConfigTableName` | Feishu webhook config table |
| `DeploymentPlanTableName` | deployment plans table |
| `NotificationTopicArn` | SNS fallback topic |
