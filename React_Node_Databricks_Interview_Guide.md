# React + Node.js + Databricks Architecture — Interview Deep Dive

> **Goal**: Score 10/10 by understanding each concept deeply, with examples and diagrams.

---

## TABLE OF CONTENTS

1. [The Big Picture — Why This Architecture?](#1-the-big-picture)
2. [Layer 1: React Frontend (Thin UI)](#2-react-frontend)
3. [Layer 2: Node.js Backend (BFF / Orchestrator)](#3-nodejs-backend)
4. [Layer 3: Databricks (Async Data Processing)](#4-databricks)
5. [The Full Communication Flow (Step-by-Step)](#5-communication-flow)
6. [Scalability Considerations](#6-scalability)
7. [Maintainability Strategies](#7-maintainability)
8. [Potential Bottlenecks & Mitigations](#8-bottlenecks)
9. [Where Security Fits (Without Over-Dominating)](#9-security)
10. [90-Second Elevator Answer](#10-elevator-answer)

---

## 1. THE BIG PICTURE

### The Core Rule (Memorize This):

```
React NEVER talks directly to Databricks.
Node.js is the middleman (orchestrator).
Databricks handles heavy async data crunching.
```

### Why?

Think of it like ordering food at a restaurant:

```
You (React)  →  Waiter (Node.js)  →  Kitchen (Databricks)

- You don't walk into the kitchen yourself.
- The waiter takes your order, gives it to the kitchen.
- The kitchen cooks (processes data) and signals when ready.
- The waiter brings the food (results) to you.
```

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    INTERNET / USERS                      │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTPS
                       ▼
┌──────────────────────────────────────────────────────────┐
│              LAYER 1: REACT FRONTEND                     │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────────┐  │
│  │ Dashboard │  │  Forms   │  │  Job Status Tracker   │  │
│  └──────────┘  └──────────┘  └───────────────────────┘  │
│                                                          │
│  Redux / RTK Query / React Query (state management)      │
└──────────────────────┬───────────────────────────────────┘
                       │ REST API / WebSocket
                       ▼
┌──────────────────────────────────────────────────────────┐
│              LAYER 2: NODE.JS BACKEND (BFF)              │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────────┐  │
│  │ API Routes│  │  Auth    │  │  Job Orchestrator     │  │
│  └──────────┘  └──────────┘  └───────────────────────┘  │
│                                                          │
│  ┌──────────────┐  ┌────────────────────────────────┐   │
│  │  Redis Cache  │  │  Queue (Bull / BullMQ)         │   │
│  └──────────────┘  └────────────────────────────────┘   │
└──────────────────────┬───────────────────────────────────┘
                       │ REST API / SDK / JDBC
                       ▼
┌──────────────────────────────────────────────────────────┐
│              LAYER 3: DATABRICKS                         │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────────┐  │
│  │  Jobs API │  │ SQL API  │  │  Delta Lake Storage   │  │
│  └──────────┘  └──────────┘  └───────────────────────┘  │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Spark Clusters (auto-scaling compute)            │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

---

## 2. REACT FRONTEND (Thin UI)

### What "Thin UI" Means

React should ONLY handle:
- ✅ Displaying data
- ✅ Capturing user input
- ✅ Showing loading/error states
- ✅ Triggering API calls

React should NOT handle:
- ❌ Business logic
- ❌ Data processing
- ❌ Direct database/Databricks queries
- ❌ Authentication token management (beyond storage)

### Easy Example: Job Trigger Component

```tsx
// AnalyticsPage.tsx — React component (THIN)
import { useState } from "react";
import api from "../store/axiosInstance";

function AnalyticsPage() {
  const [jobId, setJobId] = useState<string | null>(null);
  const [status, setStatus] = useState<string>("idle");
  const [result, setResult] = useState<any>(null);

  // Step 1: User clicks → React calls Node API (NOT Databricks!)
  const triggerReport = async () => {
    setStatus("submitting");
    const res = await api.post("/api/analytics/report", {
      dateRange: "2024-01-01/2024-12-31",
      metric: "revenue",
    });
    setJobId(res.data.jobId);    // Node returns a job ID
    setStatus("processing");
    pollForResult(res.data.jobId); // Start polling
  };

  // Step 2: React polls Node for status (NOT Databricks!)
  const pollForResult = async (id: string) => {
    const interval = setInterval(async () => {
      const res = await api.get(`/api/analytics/status/${id}`);
      if (res.data.status === "COMPLETED") {
        clearInterval(interval);
        setResult(res.data.result);
        setStatus("done");
      } else if (res.data.status === "FAILED") {
        clearInterval(interval);
        setStatus("error");
      }
    }, 3000); // Poll every 3 seconds
  };

  return (
    <div>
      <button onClick={triggerReport} disabled={status === "processing"}>
        Generate Revenue Report
      </button>

      {status === "processing" && <p>⏳ Processing... Job ID: {jobId}</p>}
      {status === "done" && <pre>{JSON.stringify(result, null, 2)}</pre>}
      {status === "error" && <p>❌ Job failed. Try again.</p>}
    </div>
  );
}
```

### Diagram: What React Does

```
User clicks "Generate Report"
         │
         ▼
┌─────────────────────┐
│     React (UI)       │
│                      │
│  1. Call Node API    │──────► POST /api/analytics/report
│  2. Get jobId back   │◄────── { jobId: "abc123" }
│  3. Show "Processing"│
│  4. Poll /status/id  │──────► GET /api/analytics/status/abc123
│  5. Get result       │◄────── { status: "COMPLETED", result: {...} }
│  6. Render result    │
└─────────────────────┘
```

### Key Interview Point: BFF Pattern

**BFF = Backend For Frontend**

```
Instead of this (BAD):
  React ──► Databricks    (direct connection, no control)
  React ──► Database
  React ──► Third-party API

Do this (GOOD):
  React ──► Node.js (BFF) ──► Databricks
                           ──► Database
                           ──► Third-party API
```

**Why BFF?**
- Node.js shapes the response for what React needs (no over-fetching)
- Single point of control for auth, caching, rate limiting
- React stays thin and fast

---

## 3. NODE.JS BACKEND (BFF / Orchestrator)

### This is the MOST IMPORTANT layer to explain well in the interview.

Node.js has 5 key responsibilities:

```
┌────────────────────────────────────────────────────┐
│                NODE.JS RESPONSIBILITIES              │
├────────────────────────────────────────────────────┤
│  1. API Gateway        → Routes, validation         │
│  2. Job Orchestrator   → Trigger Databricks jobs    │
│  3. Status Tracker     → Track job progress         │
│  4. Cache Manager      → Redis for results/status   │
│  5. Response Shaper    → Format data for React      │
└────────────────────────────────────────────────────┘
```

### How Node.js Talks to Databricks (3 Methods)

#### Method 1: Databricks REST API (Most Common)

```javascript
// databricksService.js — Node.js service
const axios = require("axios");

const DATABRICKS_HOST = process.env.DATABRICKS_HOST; // e.g., https://adb-123.azuredatabricks.net
const DATABRICKS_TOKEN = process.env.DATABRICKS_TOKEN;

const databricksApi = axios.create({
  baseURL: `${DATABRICKS_HOST}/api/2.1`,
  headers: {
    Authorization: `Bearer ${DATABRICKS_TOKEN}`,
    "Content-Type": "application/json",
  },
});

// Trigger a Databricks job
async function triggerJob(jobId, parameters) {
  const response = await databricksApi.post("/jobs/run-now", {
    job_id: jobId,
    notebook_params: parameters,  // Pass parameters to the notebook
  });
  return response.data.run_id;  // Returns a run_id to track
}

// Check job status
async function getJobStatus(runId) {
  const response = await databricksApi.get(`/jobs/runs/get?run_id=${runId}`);
  return {
    status: response.data.state.life_cycle_state,  // PENDING, RUNNING, TERMINATED
    result: response.data.state.result_state,       // SUCCESS, FAILED
  };
}

// Get job output
async function getJobOutput(runId) {
  const response = await databricksApi.get(`/jobs/runs/get-output?run_id=${runId}`);
  return response.data;
}

module.exports = { triggerJob, getJobStatus, getJobOutput };
```

#### Method 2: Databricks SQL API (For Quick Queries)

```javascript
// For smaller, interactive queries (not long-running)
async function runSqlQuery(query) {
  const response = await databricksApi.post("/sql/statements", {
    warehouse_id: process.env.DATABRICKS_WAREHOUSE_ID,
    statement: query,
    wait_timeout: "30s",  // Wait up to 30s for result
  });
  return response.data;
}

// Example usage:
// const result = await runSqlQuery("SELECT SUM(revenue) FROM sales WHERE year = 2024");
```

#### Method 3: Databricks SDK (TypeScript/JavaScript)

```javascript
// Using @databricks/sdk (official SDK)
const { WorkspaceClient } = require("@databricks/sdk");

const client = new WorkspaceClient({
  host: process.env.DATABRICKS_HOST,
  token: process.env.DATABRICKS_TOKEN,
});

async function triggerJobViaSDK(jobId) {
  const run = await client.jobs.runNow({ job_id: jobId });
  return run.run_id;
}
```

### Complete Node.js Route Example

```javascript
// routes/analytics.js — Express routes
const express = require("express");
const router = express.Router();
const redis = require("../config/redis");
const { triggerJob, getJobStatus, getJobOutput } = require("../services/databricksService");

// POST /api/analytics/report — Trigger a Databricks job
router.post("/report", async (req, res) => {
  try {
    const { dateRange, metric } = req.body;

    // 1. Validate input
    if (!dateRange || !metric) {
      return res.status(400).json({ error: "dateRange and metric are required" });
    }

    // 2. Check cache first (maybe this exact report was already generated)
    const cacheKey = `report:${metric}:${dateRange}`;
    const cached = await redis.get(cacheKey);
    if (cached) {
      return res.json({ status: "COMPLETED", result: JSON.parse(cached) });
    }

    // 3. Trigger Databricks job
    const runId = await triggerJob(process.env.DATABRICKS_REPORT_JOB_ID, {
      date_range: dateRange,
      metric: metric,
    });

    // 4. Store job info in Redis for tracking
    await redis.setex(`job:${runId}`, 3600, JSON.stringify({
      status: "PROCESSING",
      createdAt: Date.now(),
      params: { dateRange, metric },
    }));

    // 5. Return immediately (DON'T WAIT for Databricks!)
    res.json({
      jobId: runId,
      status: "PROCESSING",
      message: "Report generation started. Poll /status/:jobId for updates.",
    });

  } catch (error) {
    res.status(500).json({ error: "Failed to trigger report" });
  }
});

// GET /api/analytics/status/:jobId — Check job status
router.get("/status/:jobId", async (req, res) => {
  try {
    const { jobId } = req.params;

    // 1. Check Redis first (avoid hitting Databricks API too often)
    const cachedStatus = await redis.get(`job:${jobId}`);
    if (cachedStatus) {
      const parsed = JSON.parse(cachedStatus);
      if (parsed.status === "COMPLETED") {
        return res.json(parsed);
      }
    }

    // 2. Check Databricks for real status
    const status = await getJobStatus(jobId);

    if (status.status === "TERMINATED" && status.result === "SUCCESS") {
      // 3. Fetch the output
      const output = await getJobOutput(jobId);

      // 4. Cache the result
      const result = { status: "COMPLETED", result: output };
      await redis.setex(`job:${jobId}`, 3600, JSON.stringify(result));

      // 5. Also cache by params for future identical requests
      if (cachedStatus) {
        const { params } = JSON.parse(cachedStatus);
        const cacheKey = `report:${params.metric}:${params.dateRange}`;
        await redis.setex(cacheKey, 3600, JSON.stringify(output));
      }

      return res.json(result);
    }

    if (status.result === "FAILED") {
      return res.json({ status: "FAILED", error: "Databricks job failed" });
    }

    // Still running
    res.json({ status: "PROCESSING" });

  } catch (error) {
    res.status(500).json({ error: "Failed to check status" });
  }
});

module.exports = router;
```

### Diagram: Node.js Orchestration Flow

```
┌──────────────────────────────────────────────────────────────┐
│                    NODE.JS ORCHESTRATION                       │
│                                                               │
│  POST /api/analytics/report                                   │
│         │                                                     │
│         ▼                                                     │
│  ┌─────────────┐    Hit?    ┌───────────────┐                │
│  │  Validate    │──────────►│  Redis Cache   │──► Return      │
│  │  Input       │    Miss   └───────────────┘    cached       │
│  └─────┬───────┘                                  result      │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────┐                                        │
│  │  Trigger          │                                        │
│  │  Databricks Job   │──────► POST /api/2.1/jobs/run-now     │
│  │                   │◄────── { run_id: 12345 }              │
│  └──────┬───────────┘                                        │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────┐                                        │
│  │  Store runId      │                                        │
│  │  in Redis         │                                        │
│  └──────┬───────────┘                                        │
│         │                                                     │
│         ▼                                                     │
│  Return { jobId: 12345, status: "PROCESSING" }               │
│  (IMMEDIATELY — don't wait for Databricks!)                   │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. DATABRICKS (Async Data Processing Layer)

### What is Databricks? (Simple Explanation)

Think of Databricks as a **super-powered calculator** for big data:

```
Regular Database:  "Give me row #5"        → Instant (milliseconds)
Databricks:        "Analyze 10 billion     → Takes time (seconds to hours)
                    rows and find patterns"
```

### Why Databricks and Not Just a Regular Database?

```
┌───────────────────┬────────────────────────┬──────────────────────┐
│  Feature          │  Regular DB (Postgres) │  Databricks          │
├───────────────────┼────────────────────────┼──────────────────────┤
│  Data size        │  GBs                   │  TBs to PBs          │
│  Query type       │  Simple CRUD           │  Complex analytics   │
│  Processing       │  Single server         │  Distributed cluster │
│  Speed (small)    │  ⚡ Very fast           │  ⚡ Slower (overhead) │
│  Speed (big data) │  🐌 Very slow          │  🚀 Very fast        │
│  ML/AI            │  ❌ Not designed for   │  ✅ Built-in support │
│  Cost model       │  Always running        │  Pay per use          │
└───────────────────┴────────────────────────┴──────────────────────┘
```

### Databricks Key Concepts for the Interview

#### 1. Delta Lake (Storage)

```
Delta Lake = Smart file storage format

Regular files:     data.csv  (no versioning, no transactions)
Delta Lake:        delta/    (versioned, ACID transactions, time travel)

Think of it like:
  CSV     = Google Docs without version history
  Delta   = Google Docs WITH version history (can undo, see changes)
```

#### 2. Databricks Jobs (Processing)

```
A "Job" in Databricks = A scheduled or triggered task

Example Job: "Calculate monthly revenue for all customers"
  - Input:  10TB of transaction data in Delta Lake
  - Process: Spark distributes work across 20 machines
  - Output:  Summary table written back to Delta Lake
  - Time:    ~5 minutes (vs. hours on a single server)
```

#### 3. Auto-Scaling Clusters

```
Low demand:      ┌─────┐
                 │Node 1│   (1 machine)
                 └─────┘

High demand:     ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
                 │Node 1│ │Node 2│ │Node 3│ │Node 4│ │Node 5│
                 └─────┘ └─────┘ └─────┘ └─────┘ └─────┘
                 (auto-scales to 5 machines)

This is WHY Databricks is used:
  - You pay for 1 machine when idle
  - It auto-scales to 50 machines when processing a huge job
  - Scales back down automatically
```

### Databricks Notebook Example (What Runs Inside)

```python
# This runs INSIDE Databricks (not in Node.js!)
# Databricks notebook: /Reports/revenue_report

# Receive parameters from Node.js
date_range = dbutils.widgets.get("date_range")  # "2024-01-01/2024-12-31"
metric = dbutils.widgets.get("metric")           # "revenue"

# Read MASSIVE data from Delta Lake
df = spark.read.format("delta").table("sales_transactions")
# This table might have BILLIONS of rows!

# Process (distributed across the cluster)
from pyspark.sql import functions as F

start_date, end_date = date_range.split("/")

result = (
    df.filter(F.col("date").between(start_date, end_date))
      .groupBy("region", "product_category")
      .agg(
          F.sum("amount").alias("total_revenue"),
          F.count("*").alias("transaction_count"),
          F.avg("amount").alias("avg_transaction")
      )
      .orderBy(F.desc("total_revenue"))
)

# Write results to a Delta table (Node.js reads from here)
result.write.format("delta").mode("overwrite").saveAsTable("report_results")

# This processes 10 billion rows in ~5 minutes
# A regular database would take HOURS
```

---

## 5. THE FULL COMMUNICATION FLOW (Star of the Interview)

### Complete Flow Diagram

```
TIME ──────────────────────────────────────────────────────────►

REACT (Browser)          NODE.JS (Server)           DATABRICKS (Cloud)
     │                        │                          │
  1. │ ── POST /report ──►    │                          │
     │    {dateRange, metric} │                          │
     │                        │                          │
     │                     2. │ ── Validate input         │
     │                        │                          │
     │                     3. │ ── Check Redis cache      │
     │                        │    (cache miss)           │
     │                        │                          │
     │                     4. │ ── POST /jobs/run-now ──►│
     │                        │    {job_id, params}       │
     │                        │                       5. │ ← Receives job
     │                        │ ◄── {run_id: 12345} ─── │    Returns run_id
     │                        │                          │
     │                     6. │ ── Store in Redis         │
     │                        │    job:12345 → PROCESSING │
     │                        │                          │
     │ ◄── {jobId, status} ──│                       7. │ ── Starts processing
     │     "PROCESSING"       │                          │    (Spark cluster)
     │                        │                          │
  8. │ ── GET /status/12345 ►│                          │    ⚙️ Crunching
     │    (polling)           │                          │    billions of rows
     │                     9. │ ── GET /jobs/runs/get ──►│
     │                        │ ◄── RUNNING ────────────│
     │ ◄── "PROCESSING" ─────│                          │
     │                        │                          │
 10. │ ── GET /status/12345 ►│                          │
     │                    11. │ ── GET /jobs/runs/get ──►│
     │                        │ ◄── TERMINATED/SUCCESS ─│ 12. Done!
     │                        │                          │     Results in
     │                    13. │ ── GET /runs/get-output ►│     Delta Lake
     │                        │ ◄── {result data} ──────│
     │                        │                          │
     │                    14. │ ── Cache in Redis         │
     │                        │                          │
     │ ◄── {status: DONE,    │                          │
 15. │      result: {...}} ──│                          │
     │                        │                          │
 16. │ ── Render result       │                          │
     │    to user             │                          │
```

### Synchronous vs Asynchronous — WHY This Matters

```
❌ SYNCHRONOUS (BAD — NEVER DO THIS):

React ──► Node ──► Databricks ──────────────────► Node ──► React
                   (waits 5 minutes)
                   HTTP request times out! ☠️

✅ ASYNCHRONOUS (GOOD — ALWAYS DO THIS):

React ──► Node ──► Trigger job ──► Return immediately ──► React
                        │                                    │
                        └── Databricks processes ──┐         │
                                                    │        │
React ──► Node ──► Check if done? ◄───────────────┘         │
     ◄── "Still processing" ─────────────────────────────────┘
     ...
React ──► Node ──► Check if done? ──► Yes! Get results ──► React
     ◄── Here are your results ──────────────────────────────┘
```

### Why Not Just Use WebSockets Instead of Polling?

Both are valid! Here's when to use each:

```
┌────────────────────┬─────────────────┬───────────────────────┐
│  Method            │  Polling         │  WebSocket/SSE        │
├────────────────────┼─────────────────┼───────────────────────┤
│  Complexity        │  Simple          │  More complex         │
│  Real-time         │  Near real-time  │  True real-time       │
│  Server load       │  Higher          │  Lower                │
│  Best for          │  < 100 users     │  > 100 concurrent     │
│  Infrastructure    │  Standard REST   │  Needs WS support     │
└────────────────────┴─────────────────┴───────────────────────┘
```

WebSocket approach:

```javascript
// Node.js — WebSocket for job updates
const WebSocket = require("ws");
const wss = new WebSocket.Server({ port: 8080 });

// After Databricks job completes:
function notifyClient(userId, jobId, result) {
  const client = activeConnections.get(userId);
  if (client && client.readyState === WebSocket.OPEN) {
    client.send(JSON.stringify({ jobId, status: "COMPLETED", result }));
  }
}
```

---

## 6. SCALABILITY CONSIDERATIONS

### 6.1 Node.js Layer Scaling

```
                    ┌──────────────┐
                    │ Load Balancer│
                    │   (NGINX)    │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Node 1  │ │  Node 2  │ │  Node 3  │
        │ (API)    │ │ (API)    │ │ (API)    │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │             │            │
             └─────────────┼────────────┘
                           │
                    ┌──────┴───────┐
                    │    Redis     │
                    │  (Shared     │
                    │   State)     │
                    └──────────────┘
```

**Key Points:**

- **Stateless APIs**: No data stored in Node.js memory → any Node can handle any request
- **Redis**: Shared state (job status, cache, sessions) across all Node instances
- **Horizontal scaling**: Add more Node instances behind the load balancer

```javascript
// Why stateless matters:
// ❌ BAD: Storing state in Node.js memory
const jobStatuses = {};  // Dies when this Node instance restarts!

// ✅ GOOD: Storing state in Redis
await redis.set(`job:${runId}`, JSON.stringify({ status: "PROCESSING" }));
// Any Node instance can read this!
```

### 6.2 Databricks Layer Scaling

```
SEPARATE COMPUTE FROM STORAGE (Lakehouse Architecture)

  ┌────────────────────┐         ┌────────────────────┐
  │   COMPUTE          │         │   STORAGE           │
  │   (Spark Clusters) │         │   (Delta Lake)      │
  │                    │◄───────►│                     │
  │   Scale UP/DOWN    │  reads  │   Scale             │
  │   independently    │  writes │   independently     │
  │                    │         │                     │
  │   Pay per use      │         │   Pay per GB        │
  └────────────────────┘         └────────────────────┘

This means:
  - You can process MORE data without bigger clusters
  - You can add MORE clusters without more storage
  - Each scales on its own cost curve
```

### 6.3 React Layer Scaling

```
React = Static files (HTML, JS, CSS)
  ↓
Deploy to CDN (CloudFront, Azure CDN, Vercel)
  ↓
CDN automatically scales globally

No scaling concerns for React itself.
The concern is API call patterns:
  - Debounce user actions
  - Cache responses client-side
  - Use optimistic UI updates
```

---

## 7. MAINTAINABILITY STRATEGIES

### 7.1 Clear Separation of Concerns

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│    REACT     │    │   NODE.JS    │    │  DATABRICKS  │
│              │    │              │    │              │
│  UI only     │    │  Orchestrate │    │  Data only   │
│  No business │    │  Validate    │    │  No UI logic │
│  logic       │    │  Shape data  │    │  No API logic│
│              │    │  Cache       │    │              │
└──────────────┘    └──────────────┘    └──────────────┘

Each layer can be:
  ✅ Developed by different teams
  ✅ Deployed independently
  ✅ Tested independently
  ✅ Scaled independently
  ✅ Replaced without affecting others
```

### 7.2 Config-Driven Databricks Jobs

```javascript
// ❌ BAD: Hard-coded job IDs
const runId = await triggerJob(12345, params);

// ✅ GOOD: Config-driven
// config/databricks-jobs.json
{
  "reports": {
    "revenue":  { "jobId": 12345, "timeout": 600 },
    "users":    { "jobId": 12346, "timeout": 300 },
    "inventory": { "jobId": 12347, "timeout": 900 }
  }
}

// Usage:
const jobConfig = config.reports[metric];
const runId = await triggerJob(jobConfig.jobId, params);
```

### 7.3 API Versioning

```
/api/v1/analytics/report   ← Current version
/api/v2/analytics/report   ← New version with breaking changes

Both work simultaneously → no big-bang migrations
```

### 7.4 Error Handling Strategy

```javascript
// Centralized error handling across all layers:

// Layer 1 (React): Show user-friendly messages
// Layer 2 (Node):  Log full errors, return clean responses
// Layer 3 (Databricks): Retry failed jobs, alert on repeated failures

// Node.js error middleware:
app.use((err, req, res, next) => {
  logger.error({ err, requestId: req.id });  // Full details for debugging

  res.status(err.status || 500).json({
    error: err.userMessage || "Something went wrong",  // Clean for React
    requestId: req.id,  // For support tickets
  });
});
```

---

## 8. POTENTIAL BOTTLENECKS & MITIGATIONS

### Bottleneck Map

```
┌──────────────────────────────────────────────────────────────┐
│                    BOTTLENECK MAP                              │
│                                                               │
│  React ──────► Node.js ──────► Databricks                    │
│    │              │                │                          │
│    │              │                │                          │
│  ┌┴─────────┐ ┌──┴────────┐ ┌───┴──────────────────────┐   │
│  │Too many   │ │Blocking   │ │Cluster cold start        │   │
│  │polls      │ │HTTP calls │ │(2-5 min to spin up)      │   │
│  │           │ │           │ │                           │   │
│  │Large      │ │Memory     │ │Over-querying              │   │
│  │payloads   │ │pressure   │ │interactively              │   │
│  │           │ │from many  │ │                           │   │
│  │No         │ │concurrent │ │Large data transfers       │   │
│  │debouncing │ │requests   │ │between layers             │   │
│  └───────────┘ └───────────┘ └───────────────────────────┘   │
│                                                               │
│  MITIGATIONS:                                                 │
│  ┌───────────┐ ┌───────────┐ ┌───────────────────────────┐  │
│  │Debounce   │ │Async job  │ │Pre-warmed clusters        │  │
│  │requests   │ │pattern    │ │(always-on small cluster)  │  │
│  │           │ │           │ │                           │  │
│  │Pagination │ │Redis      │ │Pre-aggregated tables      │  │
│  │           │ │caching    │ │(materialized views)       │  │
│  │           │ │           │ │                           │  │
│  │Client-side│ │Queue      │ │Write results to fast      │  │
│  │caching    │ │(BullMQ)   │ │storage (not query live)   │  │
│  └───────────┘ └───────────┘ └───────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### Detailed Bottleneck Examples

```
BOTTLENECK 1: Cluster Cold Start
─────────────────────────────────
Problem:  Databricks cluster takes 2-5 minutes to start
          User submits a job → waits 2 min just for cluster startup!

Fix:      Use "warm pools" or always-on SQL warehouse for quick queries
          Use serverless compute (auto-provisioned)

BOTTLENECK 2: Blocking HTTP Calls
──────────────────────────────────
Problem:  Node waits synchronously for Databricks → thread blocked
          10 users = 10 blocked threads = Node.js crashes

Fix:      ALWAYS use async pattern (trigger → poll → retrieve)
          Never await a Databricks job in an HTTP handler

BOTTLENECK 3: Polling Storms
─────────────────────────────
Problem:  1000 users polling /status every 2 seconds
          = 500 requests/second hitting Node AND Databricks

Fix:      Cache status in Redis (5-10 sec TTL)
          Node checks Redis first, only hits Databricks on cache miss
          Use exponential backoff (2s → 4s → 8s → 16s)
```

---

## 9. WHERE SECURITY FITS

### Say this in ONE sentence, then move on:

> "Security is handled orthogonally: OAuth2/JWT for auth, HTTPS everywhere, API key rotation for Databricks, and network-level controls — without coupling it to data-processing logic."

### Quick Security Diagram

```
React (Browser)
  │
  │  JWT Token in header
  │  HTTPS only
  ▼
Node.js (API)
  │
  │  Validates JWT
  │  Rate limiting
  │  Input sanitization
  │  Service account token (NOT user token)
  ▼
Databricks
  │
  │  API token (server-to-server)
  │  IP allowlisting
  │  Delta Lake access controls
  ▼
Data
```

**Key Security Point**: Users NEVER get direct Databricks credentials. Node.js uses a service account.

---

## 10. THE 90-SECOND ELEVATOR ANSWER

Memorize this and say it calmly:

---

*"I'd use a **layered architecture** where React is a **thin UI**, Node.js acts as a **BFF and orchestration layer**, and Databricks handles **asynchronous data processing**.*

*React never talks directly to Databricks. Node triggers Databricks jobs via **REST APIs**, immediately returns a **job identifier**, and tracks execution **asynchronously**.*

*Databricks processes data at scale using **Spark clusters** and writes results to **Delta Lake storage**, which Node later retrieves and exposes to the UI.*

*This separation enables **independent scaling** — Node scales horizontally behind a load balancer, Databricks auto-scales compute independently from storage.*

*For **maintainability**, I'd use **config-driven job definitions**, **API versioning**, and **clear separation of concerns** so each layer can be developed and deployed independently.*

*Potential **bottlenecks** include cluster cold starts, blocking HTTP calls, and polling storms — mitigated with **warm pools**, **async job patterns**, and **Redis caching**.*

*Security is handled orthogonally with **JWT auth**, **service accounts** for Databricks, and **HTTPS** throughout."*

---

## BONUS: How Your `counterSlice.ts` Relates to This

Your existing Redux code already uses the async pattern!

```typescript
// YOUR CODE (counterSlice.ts):
export const fetchRandomNumber = createAsyncThunk(
  "counter/fetchRandomNumber",
  async (_, { signal, rejectWithValue }) => {
    const response = await api.get("/posts", { signal: controller.signal });
    return response.data[0].id;
  }
);

// INTERVIEW VERSION (Databricks integration):
export const triggerAnalyticsReport = createAsyncThunk(
  "analytics/triggerReport",
  async (params, { rejectWithValue }) => {
    try {
      // Step 1: Trigger (returns immediately with jobId)
      const res = await api.post("/api/analytics/report", params);
      return res.data;  // { jobId: "12345", status: "PROCESSING" }
    } catch (error) {
      return rejectWithValue(error.response?.data || "Failed to trigger report");
    }
  }
);

export const pollJobStatus = createAsyncThunk(
  "analytics/pollStatus",
  async (jobId, { rejectWithValue }) => {
    try {
      const res = await api.get(`/api/analytics/status/${jobId}`);
      return res.data;  // { status: "COMPLETED", result: {...} }
    } catch (error) {
      return rejectWithValue(error.response?.data || "Failed to check status");
    }
  }
);
```

Same pattern. Same `createAsyncThunk`. Same `pending/fulfilled/rejected` handling.
The only difference is: the backend talks to Databricks instead of JSONPlaceholder.

---

## INTERVIEW CHEAT SHEET (Quick Reference)

```
Question                          → Answer Keywords
─────────────────────────────────────────────────────────────────
"How does React talk to          → "Through Node.js BFF, never
 Databricks?"                       directly to Databricks"

"How do you handle long-running  → "Async job pattern: trigger,
 Databricks jobs?"                  return jobId, poll for status"

"Where's the bottleneck?"        → "Cluster cold starts, blocking
                                    calls, polling storms"

"How do you scale?"              → "Node: stateless + horizontal
                                    Databricks: auto-scaling clusters
                                    Storage: Delta Lake (independent)"

"How do you keep it              → "Separation of concerns,
 maintainable?"                     config-driven jobs, API versioning"

"Security?"                       → "JWT, service accounts, HTTPS,
                                    orthogonal to data processing"
```
