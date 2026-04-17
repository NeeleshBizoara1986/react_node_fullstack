# Q6 — Node.js Async Patterns and Concurrency for Long-Running Jobs (10/10 Preparation Guide)

## The Interview Question

> **When working with asynchronous processes in Node.js for data processing jobs, what patterns do you find most effective? How would you structure your backend to handle multiple concurrent long-running jobs while maintaining responsiveness for API requests?**

---

## Assessment Feedback (3/10) — Why You Lost Points

Your original answer:
- Touched on async and Redis caching but **lacked depth**
- **Didn't address concurrent job handling** architecture
- **Missing job queues, worker pools, microservice approaches**
- Response was **fragmented** and disorganized

To score **10/10**, you must:
1. Distinguish **async vs parallel** (most candidates confuse these)
2. Name concrete patterns: Promises, async/await, Event Emitters, Streams, Queues
3. Show the **API server + Queue + Worker** architecture with code
4. Explain **event loop** and why CPU-bound work blocks it
5. Show **Worker Threads** for CPU-heavy tasks
6. Mention **PM2 Cluster Mode** for multi-process scaling
7. Cover **retries, backpressure, graceful shutdown**
8. Include Databricks context

---

## 10/10 Interview Answer (Memorize This)

> In Node.js, I separate concerns into **API servers** that respond quickly and **background workers** that process long-running jobs. The API layer should validate, create a job record, enqueue work to **BullMQ** (backed by Redis), and return a `202 Accepted` with a `jobId` immediately — never block waiting for processing.
>
> For async patterns, I use **async/await** for readable sequential flows, **Promise.all** for concurrent I/O operations (like fetching from multiple APIs simultaneously), **Promise.allSettled** when I need partial results even if some calls fail, and **Streams** for processing large files or datasets chunk-by-chunk without loading everything into memory.
>
> For the worker layer, BullMQ handles job orchestration with built-in **retries with exponential backoff**, **concurrency control**, and **dead-letter queues** for jobs that permanently fail. Workers run as **separate processes** from the API server, so they can be scaled independently — more API instances for traffic spikes, more workers for job backlogs.
>
> If a job involves CPU-heavy computation inside Node itself (large JSON transforms, CSV parsing), I use **Worker Threads** to offload that work so the main event loop stays responsive. For process-level scaling, **PM2 in cluster mode** runs multiple Node instances across CPU cores.
>
> I also implement **backpressure** by monitoring queue depth and pausing ingestion when workers are overwhelmed, **graceful shutdown** so in-progress jobs complete cleanly before the process exits, and **job state tracking** in Redis/DB so the React frontend can poll status or receive SSE/WebSocket updates.
>
> The key principle is: **the event loop is for I/O coordination, not computation**. Any blocking work must be moved out — to queues, workers, Worker Threads, or external services like Databricks.

---

## Why This Is a 10/10 Answer

| Criteria | Covered? |
|----------|----------|
| Async vs parallel distinction | ✅ |
| Specific async patterns (Promises, async/await, Event Emitters, Streams) | ✅ |
| Job queue architecture (BullMQ) | ✅ |
| Worker processes (separate from API) | ✅ |
| Worker Threads (CPU-heavy tasks) | ✅ |
| PM2 Cluster Mode (multi-process) | ✅ |
| Retries + dead-letter queue | ✅ |
| Backpressure + graceful shutdown | ✅ |
| Event loop awareness | ✅ |
| Databricks integration | ✅ |
| API responsiveness maintained | ✅ |

---

## Async vs Parallel — Say This First

This is the **most important distinction** in the interview.

### Asynchronous (Single Thread, Non-Blocking I/O)

The program does not wait idly for I/O operations.

```
Call Databricks API → don't wait → handle other requests → callback when done
```

Node.js uses a **single thread with an event loop** that efficiently handles thousands of concurrent I/O operations without creating new threads for each one.

### Parallel (Multiple Threads/Processes, True Simultaneity)

Multiple tasks actually **run at the same time** on different CPU cores.

```
Worker Thread 1: parsing CSV file A
Worker Thread 2: parsing CSV file B
Worker Thread 3: parsing CSV file C
(all running simultaneously on different cores)
```

**Interview line**: "Async keeps Node from waiting for I/O; parallelism requires Worker Threads or separate processes for true simultaneous execution."

---

## The Event Loop — Know This Cold

```
┌───────────────────────────┐
│       Incoming Requests    │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│     Event Loop (1 thread) │  ← All I/O callbacks, async/await, timers
│                            │
│  Good: non-blocking I/O    │  ← File reads, HTTP calls, DB queries
│  Bad:  CPU-heavy sync code │  ← Blocks ALL other requests
└─────────────┬─────────────┘
              │
              ▼ offload heavy work
┌───────────────────────────┐
│  Worker Threads / Workers  │  ← CPU-heavy tasks run here
└───────────────────────────┘
```

### What Blocks the Event Loop (BAD)

```ts
// ❌ This blocks ALL users for seconds
function processData(data: any[]) {
  for (let i = 0; i < 100_000_000; i++) {
    // heavy synchronous computation
  }
}
```

### What Does NOT Block (GOOD)

```ts
// ✅ This is non-blocking — event loop stays free
const data = await fetch("https://databricks.example.com/api/2.1/jobs/runs/get?run_id=123");
const dbResult = await db.query("SELECT * FROM reports WHERE id = $1", [id]);
const cached = await redis.get("job:123");
```

**Rule**: I/O operations (HTTP, DB, file, Redis) are non-blocking. CPU computation (loops, JSON.parse of huge files, crypto, compression) blocks the event loop.

---

## Core Async Patterns — With Code

### 1. async/await (Primary Pattern)

Most readable for sequential async flows:

```ts
async function processJob(jobId: string) {
  const job = await jobRepository.findById(jobId);     // DB read (non-blocking)
  const runId = await databricks.submitJob(job.params); // HTTP call (non-blocking)
  const result = await databricks.waitForResult(runId); // Polling (non-blocking)
  await jobRepository.updateResults(jobId, result);     // DB write (non-blocking)
}
```

### 2. Promise.all (Concurrent I/O)

Run multiple independent I/O operations **at the same time**:

```ts
// ❌ Sequential — total time = 3s + 2s + 1s = 6s
const users = await fetchUsers();       // 3s
const reports = await fetchReports();   // 2s
const settings = await fetchSettings(); // 1s

// ✅ Concurrent — total time = max(3s, 2s, 1s) = 3s
const [users, reports, settings] = await Promise.all([
  fetchUsers(),     // starts immediately
  fetchReports(),   // starts immediately
  fetchSettings(),  // starts immediately
]);
```

### 3. Promise.allSettled (Partial Results)

When you need results from all calls, even if some fail:

```ts
const results = await Promise.allSettled([
  databricks.getJobStatus(runId1),
  databricks.getJobStatus(runId2),
  databricks.getJobStatus(runId3),
]);

results.forEach((result, index) => {
  if (result.status === "fulfilled") {
    console.log(`Job ${index}: ${result.value.state}`);
  } else {
    console.error(`Job ${index} failed: ${result.reason}`);
  }
});
```

### 4. Promise.race (First to Respond / Timeout)

Implement timeout or fastest-response-wins:

```ts
// Timeout pattern
async function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error(`Timeout after ${ms}ms`)), ms)
  );
  return Promise.race([promise, timeout]);
}

// Usage: fail if Databricks doesn't respond in 30s
const status = await withTimeout(databricks.getRunStatus(runId), 30000);
```

### 5. Event Emitters (Pub/Sub Within a Process)

Decouple producers from consumers:

```ts
import { EventEmitter } from "events";

const jobEvents = new EventEmitter();

// Producer: Worker emits events
jobEvents.emit("job.completed", { jobId: "j-123", result: data });
jobEvents.emit("job.failed", { jobId: "j-456", error: "timeout" });

// Consumer 1: Update cache
jobEvents.on("job.completed", async ({ jobId, result }) => {
  await redis.set(`job:${jobId}`, JSON.stringify({ status: "COMPLETED", result }));
});

// Consumer 2: Send notification
jobEvents.on("job.completed", async ({ jobId }) => {
  await notificationService.notifyUser(jobId);
});

// Consumer 3: Push to WebSocket
jobEvents.on("job.completed", async ({ jobId }) => {
  wsServer.pushToClient(jobId, { status: "COMPLETED" });
});
```

### 6. Streams (Large Data, Chunk-by-Chunk)

Process data without loading everything into memory:

```ts
import { createReadStream } from "fs";
import { Transform, pipeline } from "stream";
import { promisify } from "util";

const pipelineAsync = promisify(pipeline);

// Transform stream: process CSV rows one by one
const transformer = new Transform({
  objectMode: true,
  transform(chunk, encoding, callback) {
    const row = chunk.toString().trim();
    const processed = transformRow(row); // your business logic
    callback(null, processed);
  },
});

// Process a 10GB file without loading it all into memory
await pipelineAsync(
  createReadStream("large-dataset.csv"),
  transformer,
  createWriteStream("output.csv")
);
```

### Pattern Comparison Table

| Pattern | Use When | Example |
|---------|----------|---------|
| `async/await` | Sequential steps that depend on each other | Process job step by step |
| `Promise.all` | Multiple independent I/O operations | Fetch status of 5 jobs at once |
| `Promise.allSettled` | Need all results even if some fail | Check health of multiple services |
| `Promise.race` | Need first result or timeout | Timeout for slow external APIs |
| Event Emitters | Decoupled in-process pub/sub | Job lifecycle notifications |
| Streams | Large data, memory-efficient | Process 10GB CSV file |

---

## Core Architecture: API Server + Queue + Workers

This is the **most important architecture** for this question.

```
┌───────────────────────────────────────────────────┐
│                   React / Client                    │
└───────────────────────┬───────────────────────────┘
                        │ HTTPS
                        ▼
┌───────────────────────────────────────────────────┐
│              Load Balancer                          │
└───────────────────────┬───────────────────────────┘
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ API Srv 1│  │ API Srv 2│  │ API Srv 3│  ← respond fast, never block
   └────┬─────┘  └────┬─────┘  └────┬─────┘
        │              │              │
        └──────────────┼──────────────┘
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │  Redis   │  │  BullMQ  │  │ Postgres │
   │  Cache   │  │  Queue   │  │ Job DB   │
   └──────────┘  └────┬─────┘  └──────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ Worker 1 │ │ Worker 2 │ │ Worker 3 │   ← separate processes, scale independently
   └────┬─────┘ └────┬─────┘ └────┬─────┘
        │            │            │
        └────────────┼────────────┘
                     │ REST API / SDK
                     ▼
              ┌──────────────┐
              │  Databricks   │
              └──────────────┘
```

### Why This Design Wins

| Concern | How It's Solved |
|---------|----------------|
| API responsiveness | Controllers enqueue + return immediately (202) |
| Long-running work | Workers process in background, not in HTTP lifecycle |
| Traffic spikes | Queue absorbs burst, workers drain at steady pace |
| Independent scaling | More users? Add API instances. More jobs? Add workers. |
| Retries | BullMQ retries failed jobs automatically |
| Fault tolerance | Worker crash = BullMQ re-queues the job |

---

## BullMQ Implementation — Full Code

### Queue Setup

```ts
// queues/reportQueue.ts
import { Queue } from "bullmq";

export const reportQueue = new Queue("report-queue", {
  connection: { host: "localhost", port: 6379 },
  defaultJobOptions: {
    attempts: 3,                    // Retry up to 3 times
    backoff: {
      type: "exponential",          // 2s → 4s → 8s
      delay: 2000,
    },
    removeOnComplete: { count: 1000 }, // Keep last 1000 completed jobs
    removeOnFail: { count: 5000 },     // Keep last 5000 failed jobs
  },
});
```

### Worker Setup

```ts
// workers/reportWorker.ts
import { Worker, Job } from "bullmq";
import { databricksDao } from "../dao/databricksDao";
import { redisDao } from "../dao/redisDao";
import { jobRepository } from "../dao/jobRepository";
import { logger } from "../utils/logger";

const worker = new Worker(
  "report-queue",
  async (job: Job) => {
    const { jobId, metric, dateRange } = job.data;
    logger.info({ jobId, event: "job_started" });

    // 1. Mark as RUNNING
    await Promise.all([
      redisDao.setJobStatus(jobId, { status: "RUNNING" }),
      jobRepository.updateStatus(jobId, "RUNNING"),
    ]);

    // 2. Trigger Databricks job
    const runId = await databricksDao.submitJobRun(1, { metric, date_range: dateRange });
    logger.info({ jobId, runId, event: "databricks_triggered" });

    // 3. Poll Databricks until complete (with max attempts)
    let attempts = 0;
    const maxAttempts = 120; // 120 * 5s = 10 min max
    let status = await databricksDao.getRunStatus(runId);

    while ((status.state === "PENDING" || status.state === "RUNNING") && attempts < maxAttempts) {
      await new Promise((r) => setTimeout(r, 5000));
      status = await databricksDao.getRunStatus(runId);
      attempts++;

      // Report progress to BullMQ (visible in dashboard)
      await job.updateProgress(Math.min((attempts / maxAttempts) * 100, 99));
    }

    if (status.resultState !== "SUCCESS") {
      throw new Error(`Databricks job failed: ${status.stateMessage}`);
    }

    // 4. Fetch results
    const results = await databricksDao.queryResults(
      `SELECT * FROM reports.output WHERE job_id = '${jobId}' LIMIT 1000`
    );

    // 5. Mark COMPLETED
    await Promise.all([
      redisDao.setJobStatus(jobId, { status: "COMPLETED" }),
      jobRepository.updateWithResults(jobId, "COMPLETED", results),
    ]);

    logger.info({ jobId, event: "job_completed" });
    return results;
  },
  {
    connection: { host: "localhost", port: 6379 },
    concurrency: 5,  // Process up to 5 jobs at once per worker instance
  }
);

// Error handling
worker.on("failed", async (job, err) => {
  if (job) {
    logger.error({ jobId: job.data.jobId, error: err.message, event: "job_failed" });
    await Promise.all([
      redisDao.setJobStatus(job.data.jobId, { status: "FAILED", error: err.message }),
      jobRepository.updateStatus(job.data.jobId, "FAILED"),
    ]);
  }
});

// Graceful shutdown
process.on("SIGTERM", async () => {
  logger.info("Received SIGTERM, closing worker gracefully...");
  await worker.close(); // Waits for current jobs to finish
  process.exit(0);
});
```

### Dead-Letter Queue for Permanently Failed Jobs

```ts
// When a job exhausts all retries, move to dead-letter queue
worker.on("failed", async (job, err) => {
  if (job && job.attemptsMade >= (job.opts.attempts ?? 3)) {
    logger.error({
      jobId: job.data.jobId,
      error: err.message,
      event: "moved_to_dead_letter",
    });
    // Optionally: alert ops team for manual investigation
    await alertService.notifyOps({
      type: "DEAD_LETTER_JOB",
      jobId: job.data.jobId,
      error: err.message,
    });
  }
});
```

---

## Worker Threads — For CPU-Heavy Tasks

### When to Use

| Task Type | Solution |
|-----------|---------|
| HTTP calls, DB queries, file I/O | Regular async/await (event loop handles it) |
| Large JSON parsing, CSV parsing, compression | **Worker Threads** |
| Long-running Databricks orchestration | **Queue Workers** (BullMQ) |
| Multi-process scaling | **PM2 Cluster Mode** |

### Worker Thread Example

```ts
// utils/cpuWorker.ts
import { Worker, isMainThread, parentPort, workerData } from "worker_threads";
import { resolve } from "path";

// Function to offload CPU work to a worker thread
export function runInWorkerThread<T>(taskFile: string, data: any): Promise<T> {
  return new Promise((resolve, reject) => {
    const worker = new Worker(taskFile, { workerData: data });

    worker.on("message", (result: T) => resolve(result));
    worker.on("error", (err) => reject(err));
    worker.on("exit", (code) => {
      if (code !== 0) reject(new Error(`Worker exited with code ${code}`));
    });
  });
}

// ─── The worker thread file ───
// tasks/parseCSV.worker.ts
import { parentPort, workerData } from "worker_threads";

// This runs in a separate thread, doesn't block the event loop
function parseAndTransformCSV(csvData: string) {
  const rows = csvData.split("\n");
  const results = [];

  for (const row of rows) {
    const cols = row.split(",");
    // Heavy transformation logic here
    results.push({
      id: cols[0],
      value: parseFloat(cols[1]) * 1.15, // some computation
      category: cols[2]?.toUpperCase(),
    });
  }

  return results;
}

const result = parseAndTransformCSV(workerData.csvContent);
parentPort?.postMessage(result);
```

### Usage in API

```ts
// In a controller or service — offload CPU work
import { runInWorkerThread } from "../utils/cpuWorker";

async function processBigCSV(req: Request, res: Response) {
  const csvContent = await fs.readFile(req.body.filePath, "utf-8");

  // ✅ CPU work runs in separate thread, event loop stays free
  const parsed = await runInWorkerThread(
    "./tasks/parseCSV.worker.js",
    { csvContent }
  );

  res.json({ rowCount: parsed.length });
}
```

---

## PM2 Cluster Mode

Scale Node.js across all CPU cores:

```bash
# pm2.config.js
module.exports = {
  apps: [
    {
      name: "api-server",
      script: "./dist/server.js",
      instances: "max",        # One instance per CPU core
      exec_mode: "cluster",    # Cluster mode
      max_memory_restart: "500M",
    },
    {
      name: "job-worker",
      script: "./dist/worker.js",
      instances: 4,            # 4 worker processes
      exec_mode: "fork",       # Fork mode for workers
    },
  ],
};
```

```bash
pm2 start pm2.config.js
pm2 monit         # Monitor all processes
pm2 reload all    # Zero-downtime restart
```

### Cluster vs Worker Threads vs Queue Workers

| Approach | Use Case | How It Works |
|----------|----------|-------------|
| **PM2 Cluster** | Scale API across CPU cores | Multiple Node processes, load balanced |
| **Worker Threads** | CPU-heavy computation | Separate thread within same process |
| **Queue Workers** | Long-running background jobs | Separate process consuming from BullMQ |

**Interview line**: "For HTTP request scaling, I use PM2 cluster mode. For CPU-heavy tasks, Worker Threads. For long-running job orchestration, BullMQ queue workers."

---

## Backpressure Handling

### What Is Backpressure?

When producers (API servers) add jobs faster than consumers (workers) can process them, the queue grows unbounded.

### How to Handle It

```ts
// Monitor queue depth and slow down if needed
async function checkBackpressure() {
  const waiting = await reportQueue.getWaitingCount();
  const active = await reportQueue.getActiveCount();

  if (waiting > 1000) {
    logger.warn({ waiting, active, event: "backpressure_high" });
    // Option 1: Reject new jobs with 429
    // Option 2: Auto-scale workers
    // Option 3: Send alert to ops
  }
}

// In API controller, check before enqueuing
export async function triggerReport(req: Request, res: Response) {
  const waiting = await reportQueue.getWaitingCount();

  if (waiting > 500) {
    return res.status(429).json({
      error: "System busy, please try again later",
      retryAfter: 60,
    });
  }

  // Normal flow: enqueue job
  const job = await jobService.createAndEnqueue(req.body);
  res.status(202).json({ jobId: job.id, status: "QUEUED" });
}
```

---

## Graceful Shutdown

When a worker process is about to exit (deploy, scale-down, crash recovery), it should:

```ts
// Finish current jobs, don't accept new ones, then exit
async function gracefulShutdown(signal: string) {
  logger.info({ signal, event: "shutdown_started" });

  // 1. Stop accepting new jobs
  await worker.close();

  // 2. Close database connections
  await db.end();

  // 3. Close Redis connection
  await redis.quit();

  logger.info({ event: "shutdown_complete" });
  process.exit(0);
}

process.on("SIGTERM", () => gracefulShutdown("SIGTERM"));
process.on("SIGINT", () => gracefulShutdown("SIGINT"));
```

**Why it matters**: Without graceful shutdown, in-progress jobs are abandoned mid-execution, leading to:
- Corrupted job states
- Lost results
- Orphaned Databricks jobs still running

---

## Job State Tracking

### State Machine

```
QUEUED → RUNNING → COMPLETED
             │
             ├──→ FAILED
             │       │
             │       ├──→ RETRYING → RUNNING (retry loop)
             │       │
             │       └──→ DEAD_LETTER (exhausted retries)
             │
             └──→ CANCELLED (user cancelled)
```

### Status Endpoint for React

```ts
// GET /api/jobs/:jobId
export async function getJobStatus(req: Request, res: Response) {
  const { jobId } = req.params;

  // Read from Redis (fast) — status updated by workers
  const cached = await redis.get(`job:${jobId}`);
  if (cached) {
    return res.json(JSON.parse(cached));
  }

  // Fallback to DB
  const job = await jobRepository.findById(jobId);
  if (!job) return res.status(404).json({ error: "Job not found" });

  res.json({
    jobId: job.id,
    status: job.status,
    progress: job.progress,
    createdAt: job.createdAt,
    completedAt: job.completedAt,
  });
}
```

---

## SSE for Real-Time Status Updates (Alternative to Polling)

```ts
// SSE endpoint — push updates to React instead of polling
app.get("/api/jobs/:jobId/stream", authMiddleware, (req, res) => {
  const { jobId } = req.params;

  res.writeHead(200, {
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache",
    Connection: "keep-alive",
  });

  const interval = setInterval(async () => {
    const status = await redis.get(`job:${jobId}`);
    if (status) {
      res.write(`data: ${status}\n\n`);

      const parsed = JSON.parse(status);
      if (parsed.status === "COMPLETED" || parsed.status === "FAILED") {
        clearInterval(interval);
        res.end();
      }
    }
  }, 2000);

  req.on("close", () => {
    clearInterval(interval);
  });
});
```

---

## Monitoring and Observability

### Key Metrics to Track

| Metric | Where to Check | Why It Matters |
|--------|---------------|----------------|
| Queue depth | BullMQ dashboard | High depth = workers overwhelmed |
| Processing time (p50, p95) | Worker logs | Slow = investigate bottleneck |
| Failed job rate | Worker error logs | High = fix root cause |
| Event loop lag | `process` metrics | High lag = CPU blocking |
| Active connections | API server | Spike = check for connection leaks |

### Event Loop Lag Monitor

```ts
// Detect if something is blocking the event loop
let lastCheck = Date.now();

setInterval(() => {
  const now = Date.now();
  const lag = now - lastCheck - 1000; // Expected 1000ms interval
  if (lag > 100) {
    logger.warn({ lag, event: "event_loop_lag" });
  }
  lastCheck = now;
}, 1000);
```

---

## Complete Project Structure

```
src/
  server.ts              # Express app + middleware setup
  controllers/
    jobController.ts     # HTTP handlers (thin)
  services/
    jobService.ts        # Business logic + orchestration
  queues/
    reportQueue.ts       # BullMQ queue definition
  workers/
    reportWorker.ts      # Background job processor
  tasks/
    parseCSV.worker.ts   # Worker Thread for CPU tasks
  dao/
    databricksDao.ts     # Databricks API calls
    redisDao.ts          # Redis operations
    jobRepository.ts     # PostgreSQL job records
  middleware/
    auth.ts              # JWT verification
    rateLimiter.ts       # Rate limiting
  utils/
    logger.ts            # Structured logging
    cpuWorker.ts         # Worker Thread helper

# Separate deployment
pm2.config.js            # Process management
```

---

## 90-Second Spoken Version

> In Node.js, I separate the system into API servers that respond fast and background workers that process long-running jobs. The API layer validates, creates a job record, enqueues to BullMQ backed by Redis, and returns a 202 with job ID immediately. I never block the HTTP handler waiting for Databricks to finish.
>
> For async patterns, I use async/await for sequential flows, Promise.all for concurrent I/O, Promise.allSettled when I need partial results, and Streams for large file processing without loading everything into memory.
>
> For the worker layer, BullMQ gives me retries with exponential backoff, concurrency control, and dead-letter queues. Workers run as separate processes from the API, so I can scale them independently — more API instances for traffic, more workers for job backlogs.
>
> If I need CPU-heavy work inside Node, like large JSON transforms or CSV parsing, I use Worker Threads so the event loop stays free. For process-level scaling, PM2 in cluster mode runs one Node instance per CPU core.
>
> I also handle backpressure by monitoring queue depth and returning 429 when overwhelmed, implement graceful shutdown so in-progress jobs finish cleanly, and track job state in Redis for the frontend to poll or receive SSE updates. The key rule: the event loop is for I/O coordination, never for computation.

---

## Quick Revision Formula

```
Node Async = Fast API (202) + BullMQ Queue + Worker Processes + Redis Status
Patterns   = async/await + Promise.all + Streams + Event Emitters
CPU-Heavy  = Worker Threads (not event loop)
Scaling    = PM2 Cluster (API) + Independent Workers (jobs)
Reliability = Retries + Backoff + Dead Letter + Graceful Shutdown
```

### Three Key Lines to Remember

```
1. Async keeps Node from waiting for I/O; parallelism requires Worker Threads or separate processes.
2. Long-running jobs belong in a queue (BullMQ) processed by workers, never inside HTTP request handlers.
3. The event loop is for I/O coordination — CPU-heavy work must be offloaded to Worker Threads or external services.
```
