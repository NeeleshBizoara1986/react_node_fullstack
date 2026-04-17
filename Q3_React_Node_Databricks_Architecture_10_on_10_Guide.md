# Q3 — React + Node.js + Databricks Architecture (10/10 Preparation Guide)

## The Interview Question

> **When integrating a React frontend with a Node.js backend that communicates with Databricks for data processing, what architectural patterns would you employ to ensure scalability and maintainability? Please walk through your design considerations, potential bottlenecks, and how you'd structure the communication flow.**

---

## Assessment Feedback (4/10) — Why You Lost Points

Your original answer:
- **Focused heavily on security** rather than architecture
- **Lacked Databricks** integration specifics (REST API, SQL API, SDK)
- **Missed communication flow** between all three layers
- **Was disorganized** — no clear layered structure

To score **10/10**, you must:
1. Lead with architecture, not security
2. Show the **async job pattern** as the core design
3. Name specific Databricks APIs and when to use each
4. Walk through a **concrete communication flow**
5. Identify **bottlenecks** and give specific mitigations
6. Show Node.js **separation of concerns** (middleware, controller, service, DAO)

---

## 10/10 Interview Answer (Memorize This)

> I would use a **three-layer architecture** where React is a thin presentation layer, Node.js acts as a **BFF (Backend for Frontend) and orchestration layer**, and Databricks is the **asynchronous data processing engine**. The cardinal rule is: **React should never communicate directly with Databricks** — Node.js owns all orchestration.
>
> When a user triggers a report, React calls a Node.js REST endpoint. Node validates the request, authenticates the user, then calls the **Databricks Jobs API** to trigger a Spark job. Databricks returns a `runId` immediately, which Node stores in Redis alongside the job status. Node returns `{ jobId, status: "PROCESSING" }` to React right away — no blocking. React then either polls a status endpoint, or listens via **SSE/WebSocket** for real-time updates. When the job completes, Databricks writes results to **Delta Lake**, and Node retrieves the output via the **Databricks SQL API** or a pre-signed URL.
>
> Inside Node.js, I maintain strict separation: **middleware** handles logging, auth, and validation; **controllers** handle HTTP; **services** contain orchestration logic; **DAO/adapters** encapsulate Databricks SDK calls, Redis calls, and database access. This makes each layer testable and replaceable.
>
> For scalability: React is static assets on a CDN. Node.js is **stateless** behind a load balancer, with Redis for shared job state. **BullMQ** queues decouple API responses from long job execution. Databricks scales independently with **auto-scaling clusters** and Delta Lake storage.
>
> Key bottlenecks I watch for: **cluster cold starts** (mitigated by warm pools or serverless compute), **polling storms** (mitigated by Redis caching + exponential backoff or SSE), **large payload transfer** (mitigated by pre-aggregation + pagination), and **tight coupling** (mitigated by the BFF + adapter pattern). I would also add **observability** with structured logging, correlation IDs flowing from React through Node to Databricks, and health-check endpoints for all layers.

---

## Why This Is a 10/10 Answer

| Criteria | Covered? |
|----------|----------|
| Three-layer architecture | ✅ React + Node BFF + Databricks |
| Async job pattern | ✅ trigger → jobId → poll/SSE → results |
| Databricks-specific APIs | ✅ Jobs API, SQL API, Delta Lake |
| Communication flow | ✅ Full end-to-end walkthrough |
| Bottlenecks + mitigations | ✅ Cold starts, polling, payloads, coupling |
| Separation of concerns | ✅ Middleware/Controller/Service/DAO |
| Scalability | ✅ CDN, stateless Node, auto-scaling Databricks |
| Observability | ✅ Correlation IDs, structured logging |

---

## Easy Mental Model

Think of it like a restaurant:

```
React      = Customer (orders food, waits for updates)
Node.js    = Waiter (takes order, tracks status, serves results)
Databricks = Kitchen (cooks the heavy meal asynchronously)
Redis      = Order board (shared status tracking)
Delta Lake = Pantry (stores raw + processed ingredients)
```

The customer **never** walks into the kitchen.

---

## High-Level Architecture Diagram

```
┌────────────────────────────────────────────────────────┐
│                     React UI (CDN)                      │
│  - forms, dashboards, job progress                      │
│  - lazy-loaded routes, code-split bundles               │
└────────────────────┬───────────────────────────────────┘
                     │ HTTPS (REST / SSE / WebSocket)
                     ▼
┌────────────────────────────────────────────────────────┐
│              Load Balancer (ALB / Nginx)                 │
└────────────────────┬───────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ Node 1  │  │ Node 2  │  │ Node 3  │  (stateless)
   └────┬────┘  └────┬────┘  └────┬────┘
        │            │            │
        └────────────┼────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌────────────────┐
   │  Redis  │  │  BullMQ  │ │  PostgreSQL DB  │
   │ (cache) │  │ (queue)  │ │  (job records)  │
   └─────────┘  └────┬─────┘ └────────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ Worker 1 │ │ Worker 2 │ │ Worker 3 │
   └────┬─────┘ └────┬─────┘ └────┬─────┘
        │            │            │
        └────────────┼────────────┘
                     │ Databricks REST API / SDK
                     ▼
┌────────────────────────────────────────────────────────┐
│                     Databricks                          │
│  ┌──────────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Jobs API     │  │ SQL API  │  │ Auto-Scaling     │  │
│  │ (run Spark)  │  │ (query)  │  │ Clusters + Pools │  │
│  └──────────────┘  └──────────┘  └──────────────────┘  │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Delta Lake (storage: raw + processed + results)   │   │
│  └──────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────┘
```

---

## Architectural Patterns

### Pattern 1: Layered Architecture

Three separately deployable layers:

| Layer | Responsibility | Scales by |
|-------|---------------|-----------|
| **React** | Presentation, UX, job progress | CDN, edge caching |
| **Node.js** | Auth, validation, orchestration | Horizontal (add instances) |
| **Databricks** | Heavy compute, ETL, analytics | Auto-scaling clusters |

### Pattern 2: BFF (Backend for Frontend)

Node.js is a **BFF**, not a proxy.

What BFF does:
- Shapes data specifically for React (DTOs)
- Hides Databricks internals from the frontend
- Aggregates from multiple backends if needed
- Applies business logic before returning to UI

### Pattern 3: Async Job Pattern (Most Critical)

This is the **core design** — mention it first in the interview.

```
1. React triggers report generation
2. Node validates, creates job record, enqueues work
3. Node returns { jobId, status: "QUEUED" } immediately
4. Worker picks job from queue
5. Worker calls Databricks Jobs API → gets runId
6. Worker monitors Databricks job → updates Redis
7. React receives updates via polling / SSE / WebSocket
8. Worker fetches results from Delta Lake / SQL API
9. React fetches final result on completion
```

**Never** do this:
```
React → Node → Databricks → wait 5 min → return response ❌
```

---

## Communication Flow — Detailed Timeline

```
React                  Node.js                  Queue/Worker             Databricks
  │                       │                          │                       │
  │ POST /api/reports     │                          │                       │
  │──────────────────────►│                          │                       │
  │                       │ validate + auth          │                       │
  │                       │ create job record in DB  │                       │
  │                       │ enqueue to BullMQ        │                       │
  │                       │─────────────────────────►│                       │
  │◄──────────────────────│                          │                       │
  │ 202 { jobId: "j-123", │                          │                       │
  │   status: "QUEUED" }  │                          │                       │
  │                       │                          │                       │
  │                       │                          │ Trigger Jobs API      │
  │                       │                          │──────────────────────►│
  │                       │                          │◄──────────────────────│
  │                       │                          │   runId = "r-456"     │
  │                       │                          │                       │
  │                       │                          │ Update Redis:         │
  │                       │                          │ j-123 → RUNNING       │
  │                       │                          │                       │
  │ GET /api/jobs/j-123   │                          │              (Spark running)
  │──────────────────────►│                          │                       │
  │                       │ Read Redis               │                       │
  │◄──────────────────────│                          │                       │
  │ { status: "RUNNING" } │                          │                       │
  │                       │                          │                       │
  │                       │                          │ Poll Databricks status│
  │                       │                          │──────────────────────►│
  │                       │                          │◄──────────────────────│
  │                       │                          │   TERMINATED (success)│
  │                       │                          │                       │
  │                       │                          │ Fetch results via     │
  │                       │                          │ SQL API / Delta read  │
  │                       │                          │──────────────────────►│
  │                       │                          │◄──────────────────────│
  │                       │                          │   result data         │
  │                       │                          │                       │
  │                       │                          │ Update Redis + DB:    │
  │                       │                          │ j-123 → COMPLETED     │
  │                       │                          │                       │
  │ GET /api/jobs/j-123   │                          │                       │
  │──────────────────────►│                          │                       │
  │                       │ Read Redis               │                       │
  │◄──────────────────────│                          │                       │
  │ { status: "COMPLETED",│                          │                       │
  │   resultUrl: "..." }  │                          │                       │
  │                       │                          │                       │
  │ GET /api/jobs/j-123/  │                          │                       │
  │     results           │                          │                       │
  │──────────────────────►│                          │                       │
  │◄──────────────────────│                          │                       │
  │ { data: [...] }       │                          │                       │
```

---

## Three Databricks APIs — Know When to Use Each

| API | Use Case | Example |
|-----|----------|---------|
| **Jobs API** | Trigger long-running Spark jobs (ETL, ML, aggregation) | `POST /api/2.1/jobs/runs/submit` |
| **SQL API** | Query pre-computed results or Delta tables | `POST /api/2.0/sql/statements` |
| **SDK** (`@databricks/sdk`) | Structured, type-safe integration from Node.js | `workspace.jobs.runNow(...)` |

### When to use Jobs API
- Heavy data processing (millions of rows)
- ETL pipelines, ML training
- Batch analytics

### When to use SQL API
- Quick reads of pre-aggregated data
- Dashboard queries
- Small ad-hoc lookups

### When NOT to use Databricks
- Simple CRUD operations → use PostgreSQL
- Real-time per-user lookups → use Redis/normal DB
- Sub-second response requirements → pre-compute and cache

---

## Node.js Code Examples

### Project Structure

```
src/
  middleware/
    logger.ts           # request logging + correlation ID
    auth.ts             # JWT verification
    validate.ts         # input validation (Zod)
    rateLimiter.ts      # rate limiting
  controllers/
    jobController.ts    # HTTP request/response handling
  services/
    jobService.ts       # business logic + orchestration
  queues/
    reportQueue.ts      # BullMQ queue setup
  workers/
    reportWorker.ts     # background job processor
  dao/
    databricksDao.ts    # Databricks API calls
    redisDao.ts         # Redis cache operations
    jobRepository.ts    # DB operations for job records
  types/
    job.ts              # shared types
```

### Controller (Thin — Only HTTP Handling)

```ts
// controllers/jobController.ts
import { Request, Response } from "express";
import { jobService } from "../services/jobService";

export async function triggerReport(req: Request, res: Response) {
  const { metric, dateRange, filters } = req.body;
  const userId = req.user.id; // from auth middleware

  const job = await jobService.createAndEnqueue({
    type: "REPORT",
    params: { metric, dateRange, filters },
    userId,
  });

  res.status(202).json({
    jobId: job.id,
    status: job.status,
    message: "Report generation started",
  });
}

export async function getJobStatus(req: Request, res: Response) {
  const { jobId } = req.params;
  const status = await jobService.getStatus(jobId);

  if (!status) {
    return res.status(404).json({ error: "Job not found" });
  }

  res.json(status);
}

export async function getJobResults(req: Request, res: Response) {
  const { jobId } = req.params;
  const results = await jobService.getResults(jobId);

  if (!results) {
    return res.status(404).json({ error: "Results not available" });
  }

  res.json(results);
}
```

### Service (Business Logic + Orchestration)

```ts
// services/jobService.ts
import { jobRepository } from "../dao/jobRepository";
import { redisDao } from "../dao/redisDao";
import { reportQueue } from "../queues/reportQueue";

export const jobService = {
  async createAndEnqueue(params: {
    type: string;
    params: Record<string, any>;
    userId: string;
  }) {
    // 1. Create job record in DB
    const job = await jobRepository.create({
      type: params.type,
      status: "QUEUED",
      params: params.params,
      userId: params.userId,
    });

    // 2. Cache initial status in Redis
    await redisDao.setJobStatus(job.id, {
      status: "QUEUED",
      createdAt: job.createdAt,
    });

    // 3. Enqueue to BullMQ
    await reportQueue.add("generate-report", {
      jobId: job.id,
      ...params.params,
    });

    return job;
  },

  async getStatus(jobId: string) {
    // Try Redis first (fast), fall back to DB
    const cached = await redisDao.getJobStatus(jobId);
    if (cached) return cached;

    return jobRepository.getStatus(jobId);
  },

  async getResults(jobId: string) {
    const job = await jobRepository.findById(jobId);
    if (!job || job.status !== "COMPLETED") return null;
    return job.results;
  },
};
```

### Databricks DAO (Encapsulates All Databricks Calls)

```ts
// dao/databricksDao.ts
import axios from "axios";

const DATABRICKS_HOST = process.env.DATABRICKS_HOST!;
const DATABRICKS_TOKEN = process.env.DATABRICKS_TOKEN!; // from Key Vault

const client = axios.create({
  baseURL: `${DATABRICKS_HOST}/api`,
  headers: { Authorization: `Bearer ${DATABRICKS_TOKEN}` },
  timeout: 30000,
});

export const databricksDao = {
  // Jobs API — trigger a Spark job
  async submitJobRun(jobId: number, params: Record<string, string>) {
    const response = await client.post("/2.1/jobs/runs/submit", {
      run_name: `report-${Date.now()}`,
      existing_cluster_id: process.env.DATABRICKS_CLUSTER_ID,
      notebook_task: {
        notebook_path: "/Workflows/GenerateReport",
        base_parameters: params,
      },
    });
    return response.data.run_id; // Databricks runId
  },

  // Jobs API — check job status
  async getRunStatus(runId: number) {
    const response = await client.get(`/2.1/jobs/runs/get?run_id=${runId}`);
    return {
      state: response.data.state.life_cycle_state,
      resultState: response.data.state.result_state,
      stateMessage: response.data.state.state_message,
    };
  },

  // SQL API — query processed results from Delta Lake
  async queryResults(sql: string) {
    const response = await client.post("/2.0/sql/statements", {
      warehouse_id: process.env.DATABRICKS_SQL_WAREHOUSE_ID,
      statement: sql,
      wait_timeout: "30s",
    });
    return response.data.result?.data_array ?? [];
  },
};
```

### Worker (Background Job Processor)

```ts
// workers/reportWorker.ts
import { Worker } from "bullmq";
import { databricksDao } from "../dao/databricksDao";
import { redisDao } from "../dao/redisDao";
import { jobRepository } from "../dao/jobRepository";

const worker = new Worker(
  "report-queue",
  async (queueJob) => {
    const { jobId, metric, dateRange } = queueJob.data;

    // 1. Update status → RUNNING
    await redisDao.setJobStatus(jobId, { status: "RUNNING" });
    await jobRepository.updateStatus(jobId, "RUNNING");

    // 2. Trigger Databricks Spark job
    const runId = await databricksDao.submitJobRun(1, {
      metric,
      date_range: dateRange,
    });

    // 3. Poll Databricks until job completes
    let status = await databricksDao.getRunStatus(runId);
    while (status.state === "PENDING" || status.state === "RUNNING") {
      await new Promise((r) => setTimeout(r, 5000)); // poll every 5s
      status = await databricksDao.getRunStatus(runId);
    }

    if (status.resultState !== "SUCCESS") {
      await redisDao.setJobStatus(jobId, { status: "FAILED", error: status.stateMessage });
      await jobRepository.updateStatus(jobId, "FAILED");
      throw new Error(`Databricks job failed: ${status.stateMessage}`);
    }

    // 4. Fetch results from Delta Lake via SQL API
    const results = await databricksDao.queryResults(
      `SELECT * FROM reports.output WHERE job_id = '${jobId}' LIMIT 1000`
    );

    // 5. Update status → COMPLETED with results
    await redisDao.setJobStatus(jobId, { status: "COMPLETED" });
    await jobRepository.updateWithResults(jobId, "COMPLETED", results);
  },
  {
    connection: { host: "localhost", port: 6379 },
    concurrency: 5,
  }
);
```

### React Hook for Job Tracking

```tsx
// hooks/useJobTracker.ts
import { useState, useEffect, useCallback, useRef } from "react";
import axios from "./axiosInstance";

type JobStatus = "QUEUED" | "RUNNING" | "COMPLETED" | "FAILED";

interface JobState {
  jobId: string | null;
  status: JobStatus | null;
  results: any;
  error: string | null;
}

export function useJobTracker() {
  const [state, setState] = useState<JobState>({
    jobId: null,
    status: null,
    results: null,
    error: null,
  });
  const intervalRef = useRef<ReturnType<typeof setInterval>>();

  const triggerJob = useCallback(async (params: Record<string, any>) => {
    const { data } = await axios.post("/api/reports", params);
    setState({ jobId: data.jobId, status: data.status, results: null, error: null });

    // Persist jobId so it survives browser refresh
    localStorage.setItem("activeJobId", data.jobId);

    return data.jobId;
  }, []);

  const pollStatus = useCallback((jobId: string) => {
    let delay = 2000; // start with 2s, exponential backoff

    const poll = async () => {
      try {
        const { data } = await axios.get(`/api/jobs/${jobId}`);
        setState((prev) => ({ ...prev, status: data.status }));

        if (data.status === "COMPLETED") {
          const { data: results } = await axios.get(`/api/jobs/${jobId}/results`);
          setState((prev) => ({ ...prev, results, status: "COMPLETED" }));
          localStorage.removeItem("activeJobId");
          return;
        }

        if (data.status === "FAILED") {
          setState((prev) => ({ ...prev, status: "FAILED", error: data.error }));
          localStorage.removeItem("activeJobId");
          return;
        }

        delay = Math.min(delay * 1.5, 15000);
        intervalRef.current = setTimeout(poll, delay);
      } catch {
        delay = Math.min(delay * 2, 30000);
        intervalRef.current = setTimeout(poll, delay);
      }
    };

    poll();
  }, []);

  // Resume polling after browser refresh
  useEffect(() => {
    const savedJobId = localStorage.getItem("activeJobId");
    if (savedJobId) {
      setState((prev) => ({ ...prev, jobId: savedJobId, status: "RUNNING" }));
      pollStatus(savedJobId);
    }
    return () => clearTimeout(intervalRef.current);
  }, [pollStatus]);

  return { ...state, triggerJob, pollStatus };
}
```

---

## Separation of Concerns — Why This Matters

```
middleware/    → Cross-cutting concerns (logging, auth, validation, rate limiting)
controllers/  → HTTP handling only (parse request, send response)
services/     → Business logic and orchestration
queues/       → Queue definitions and configuration
workers/      → Background job execution
dao/          → External system calls (Databricks, Redis, DB)
```

**Why this design is strong:**
- **Testability**: mock the DAO layer to test services without hitting Databricks
- **Replaceability**: swap Databricks for Snowflake by changing only the DAO
- **Maintainability**: a new developer knows exactly where to add business logic vs HTTP handling
- **Scalability**: workers can be deployed separately from API servers

---

## Scalability Design

### React (Easiest to Scale)

| Strategy | How |
|----------|-----|
| CDN hosting | Vercel, CloudFront, Azure CDN |
| Code splitting | `React.lazy()` per route |
| Caching | Service workers, cache-control headers |

### Node.js (Stateless + Horizontal)

| Strategy | How |
|----------|-----|
| Stateless | No session state in memory — use Redis |
| Horizontal scaling | Multiple instances behind load balancer |
| Queue decoupling | BullMQ separates API from processing |
| Health checks | `/health` endpoint for load balancer |

### Databricks (Independent Compute Scaling)

| Strategy | How |
|----------|-----|
| Auto-scaling clusters | Min/max workers based on load |
| Instance pools | Pre-warmed VMs reduce cold start |
| Serverless SQL | Instant-on SQL warehouse |
| Delta Lake | Separation of storage from compute |

---

## Bottlenecks and Mitigations

### 1. Blocking HTTP Requests

```
Problem: Node waits synchronously for Databricks job (minutes)
Impact:  Request timeout, blocked connections, poor UX
Fix:     Async job pattern — return jobId immediately, process in background
```

### 2. Cluster Cold Start

```
Problem: Databricks cluster takes 2-5 minutes to spin up
Impact:  User sees no progress for minutes
Fix:     Instance pools (warm VMs), serverless compute, or always-on small cluster
         Show "Cluster starting..." status in UI
```

### 3. Polling Storms

```
Problem: 1000 users polling /status every second = 1000 RPS just for status
Impact:  Wasted server + network resources
Fix:     - Redis cache for status (avoid DB hit)
         - Exponential backoff (2s → 3s → 5s → 10s → 15s cap)
         - SSE or WebSocket for real-time push (eliminates polling)
```

### 4. Large Payload Transfer

```
Problem: Databricks returns millions of rows
Impact:  Node.js memory spike, slow response, browser crash
Fix:     - Pre-aggregate results in Databricks (GROUP BY, TOP N)
         - Paginate API response (limit + offset)
         - Store large results in blob storage, return signed URL
         - Stream large results instead of buffering
```

### 5. Tight Coupling

```
Problem: React knows Databricks job details, or Node has hardcoded Databricks logic
Impact:  Changing Databricks = changing frontend + backend
Fix:     - BFF shapes data for React (React never sees Databricks internals)
         - DAO/adapter pattern in Node (swap Databricks for Snowflake = change one file)
         - Config-driven job parameters
```

---

## Observability and Monitoring

This is what **separates a senior answer from a mid-level answer**.

### Correlation IDs

Every request gets a unique `requestId` that flows through all layers:

```
React (header: x-request-id: "abc-123")
  → Node.js logs: [abc-123] POST /api/reports
    → Worker logs: [abc-123] triggering Databricks job
      → Databricks notebook parameter: requestId = "abc-123"
```

This allows tracing a single user action across React → Node → Queue → Worker → Databricks.

### Key Metrics to Monitor

| Layer | Metric |
|-------|--------|
| React | LCP, FID, error rate, API latency |
| Node.js API | Request latency (p50, p95, p99), error rate, active connections |
| Queue | Queue depth, processing time, failed jobs count |
| Workers | Jobs/sec, success rate, Databricks call latency |
| Databricks | Cluster utilization, job duration, failed runs |

### Health Check Endpoint

```ts
app.get("/health", async (req, res) => {
  const redisOk = await redisDao.ping();
  const dbOk = await jobRepository.ping();

  const healthy = redisOk && dbOk;
  res.status(healthy ? 200 : 503).json({
    status: healthy ? "healthy" : "degraded",
    redis: redisOk,
    database: dbOk,
    uptime: process.uptime(),
  });
});
```

---

## Error Handling in the Architecture

### API Layer

```ts
// Centralized error middleware
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  const requestId = req.headers["x-request-id"];
  logger.error({ requestId, error: err.message, stack: err.stack });

  if (err instanceof AppError) {
    return res.status(err.statusCode).json({ error: err.message, requestId });
  }

  res.status(500).json({ error: "Internal server error", requestId });
});
```

### Worker Layer

```ts
worker.on("failed", async (job, err) => {
  logger.error({ jobId: job?.data.jobId, error: err.message });
  if (job) {
    await jobRepository.updateStatus(job.data.jobId, "FAILED");
    await redisDao.setJobStatus(job.data.jobId, {
      status: "FAILED",
      error: err.message,
    });
  }
});
```

### Databricks Layer

Failed Databricks jobs are detected by polling `getRunStatus()`. The worker catches `FAILED` result state and marks the job accordingly.

---

## 90-Second Spoken Version

> I would use a three-layer architecture where React is the thin UI on a CDN, Node.js acts as a BFF and orchestration layer, and Databricks handles asynchronous data processing. The cardinal rule is React never talks directly to Databricks.
>
> The core pattern is async jobs. When a user triggers a report, React calls a Node endpoint. Node validates, creates a job record, enqueues it to BullMQ, and returns a job ID immediately. A background worker picks the job, calls the Databricks Jobs API to trigger a Spark job, polls for completion, then fetches results via the SQL API or Delta Lake. React polls a status endpoint or uses SSE for real-time updates.
>
> Inside Node, I keep strict separation of concerns: middleware for auth and logging, controllers for HTTP, services for business logic, and a DAO layer that encapsulates all Databricks SDK calls. This makes each layer testable and replaceable.
>
> For scalability: React on CDN, Node stateless behind a load balancer, BullMQ queue to decouple API from processing, Databricks with auto-scaling clusters and Delta storage. All three layers scale independently.
>
> Key bottlenecks: cluster cold starts — mitigated by warm pools; polling storms — mitigated by Redis caching and exponential backoff; large payloads — mitigated by pre-aggregation and pagination. I also propagate correlation IDs from React through Node to Databricks for full traceability.

---

## Quick Revision Formula

```
Architecture = React CDN + Node BFF + BullMQ Queue + Workers + Databricks Async
Communication = REST trigger → jobId → poll/SSE → results from Delta Lake
Node.js      = Middleware → Controller → Service → DAO (Databricks/Redis/DB)
Scalability  = Stateless Node + Auto-scaling Databricks + Queue decoupling
Bottlenecks  = Cold starts + Polling storms + Large payloads + Tight coupling
```

### Three Key Lines to Remember

```
1. React never talks directly to Databricks — Node.js owns all orchestration.
2. Long-running jobs use async pattern: trigger → jobId → background worker → results.
3. Databricks Jobs API for heavy compute, SQL API for querying results, Delta Lake for storage.
```
