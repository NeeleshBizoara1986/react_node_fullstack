# React + Node.js + Databricks — Complete Interview Guide V2

> **Additions**: Separation of Concerns (Middleware, Logger, Services, Controllers, Error Handler),
> Deep Security, Microservices Communication, Node-to-Databricks Secure Calls

---

## TABLE OF CONTENTS

1. [Architecture Overview](#1-architecture-overview)
2. [Node.js — Separation of Concerns (Full Project Structure)](#2-separation-of-concerns)
3. [Middleware Layer (Logger, Auth, Validation, Rate Limiter)](#3-middleware-layer)
4. [Controller Layer](#4-controller-layer)
5. [Service Layer](#5-service-layer)
6. [DAO / Data Access Layer](#6-dao-layer)
7. [Centralized Error Handler](#7-error-handler)
8. [Security — Deep Dive](#8-security-deep-dive)
9. [Microservices Communication (Dashboard + Jobs Example)](#9-microservices-communication)
10. [Secure Node.js ↔ Databricks Communication](#10-node-to-databricks-secure)
11. [Full Flow with All Layers (End-to-End Diagram)](#11-full-flow)
12. [90-Second Interview Answer (Updated)](#12-elevator-answer)

---

## 1. ARCHITECTURE OVERVIEW

### The Golden Rule

```
React NEVER talks directly to Databricks.
Node.js is the middleman (orchestrator).
Databricks handles heavy async data crunching.
```

### Restaurant Analogy

```
You (React)  →  Waiter (Node.js)  →  Kitchen (Databricks)

- You don't walk into the kitchen yourself
- The waiter takes your order, gives it to the kitchen
- The kitchen cooks (processes data) and signals when ready
- The waiter brings the food (results) to you
```

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                INTERNET / USERS (Browser)                      │
└──────────────────────┬───────────────────────────────────────┘
                       │ HTTPS + JWT
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                 REACT FRONTEND (Thin UI)                       │
│  Dashboard │ Forms │ Job Status Tracker │ Charts               │
└──────────────────────┬───────────────────────────────────────┘
                       │ REST API / WebSocket
                       ▼
┌──────────────────────────────────────────────────────────────┐
│               NODE.JS BACKEND (BFF / Orchestrator)            │
│                                                               │
│  Middleware → Controllers → Services → DAOs                   │
│  (Logger, Auth, Validator, RateLimiter, ErrorHandler)         │
│                                                               │
│  Redis (Cache + Job Tracking) │ Queue (BullMQ)                │
└──────────────────────┬───────────────────────────────────────┘
                       │ REST API + Service Account Token
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                 DATABRICKS (Data Processing)                   │
│  Jobs API │ SQL API │ Delta Lake │ Spark Clusters              │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. SEPARATION OF CONCERNS — Full Project Structure

### Why Separation of Concerns Matters

```
❌ BAD (Everything in one file):
  routes/analytics.js  →  Validates input + Calls Databricks + Handles errors
                           (God file — untestable, unmaintainable)

✅ GOOD (Separated responsibilities):
  middleware/     →  Cross-cutting concerns (auth, logging, validation)
  controllers/    →  Handle HTTP request/response
  services/       →  Business logic + orchestration
  dao/            →  Data access (Databricks, Redis, DB)
  errors/         →  Custom error classes + centralized handler
  config/         →  Environment config
  utils/          →  Shared helpers
```

### Full Folder Structure

```
node-backend/
├── src/
│   ├── app.js                     ← Express app setup
│   ├── server.js                  ← Server entry point
│   │
│   ├── config/
│   │   ├── index.js               ← All config from env vars
│   │   ├── redis.js               ← Redis connection
│   │   └── databricks.js          ← Databricks connection config
│   │
│   ├── middleware/
│   │   ├── logger.js              ← Request/response logging
│   │   ├── auth.js                ← JWT verification
│   │   ├── validate.js            ← Input validation
│   │   ├── rateLimiter.js         ← Rate limiting
│   │   └── errorHandler.js        ← Centralized error handler
│   │
│   ├── controllers/
│   │   ├── analyticsController.js ← HTTP layer only
│   │   └── jobController.js       ← Job status endpoints
│   │
│   ├── services/
│   │   ├── analyticsService.js    ← Business logic
│   │   ├── jobService.js          ← Job orchestration logic
│   │   └── cacheService.js        ← Cache logic
│   │
│   ├── dao/
│   │   ├── databricksDao.js       ← Databricks API calls
│   │   └── redisDao.js            ← Redis operations
│   │
│   ├── errors/
│   │   └── AppError.js            ← Custom error classes
│   │
│   ├── routes/
│   │   ├── analyticsRoutes.js     ← Route definitions
│   │   └── jobRoutes.js
│   │
│   └── utils/
│       ├── constants.js
│       └── helpers.js
│
├── .env
├── package.json
└── tests/
```

### Flow Through the Layers

```
HTTP Request arrives
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│  MIDDLEWARE LAYER (runs in order, like a pipeline)        │
│                                                          │
│  1. Logger        → Logs request method, URL, timestamp  │
│  2. Auth          → Validates JWT token                  │
│  3. Rate Limiter  → Blocks if too many requests          │
│  4. Validator     → Checks request body/params           │
└──────────────────────┬──────────────────────────────────┘
                       │ (request passes all checks)
                       ▼
┌──────────────────────────────────────────────────────────┐
│  CONTROLLER         → Extracts data from req              │
│                     → Calls service                       │
│                     → Sends HTTP response                 │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  SERVICE            → Business logic                      │
│                     → Orchestrates (cache check, job      │
│                       trigger, status tracking)            │
│                     → Calls DAO layer                     │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  DAO (Data Access)  → Talks to Databricks REST API        │
│                     → Talks to Redis                      │
│                     → Talks to Database                   │
└──────────────────────────────────────────────────────────┘

If any layer throws → ERROR HANDLER catches it → sends clean response
```

---

## 3. MIDDLEWARE LAYER

### What is Middleware?

```
Middleware = Functions that run BEFORE your actual route handler.
They sit in the middle between the request and the controller.

Think of it like airport security:
  Gate 1: Check passport  (Auth middleware)
  Gate 2: Scan luggage     (Validation middleware)
  Gate 3: Log entry        (Logger middleware)
  Gate 4: Check capacity   (Rate limiter middleware)
  
  Only after ALL gates → you board the plane (Controller)
```

### 3.1 Logger Middleware

```javascript
// src/middleware/logger.js
const { v4: uuidv4 } = require("uuid");

/**
 * WHY: Every request gets a unique ID so we can trace it
 * through all layers (controller → service → dao → Databricks).
 * This is critical for debugging in production.
 */
function loggerMiddleware(req, res, next) {
  // Generate unique request ID
  req.requestId = req.headers["x-request-id"] || uuidv4();
  
  const startTime = Date.now();

  // Log when request arrives
  console.log(JSON.stringify({
    type: "REQUEST",
    requestId: req.requestId,
    method: req.method,
    url: req.originalUrl,
    ip: req.ip,
    userAgent: req.get("User-Agent"),
    timestamp: new Date().toISOString(),
  }));

  // Log when response is sent (using 'finish' event)
  res.on("finish", () => {
    const duration = Date.now() - startTime;
    console.log(JSON.stringify({
      type: "RESPONSE",
      requestId: req.requestId,
      method: req.method,
      url: req.originalUrl,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      timestamp: new Date().toISOString(),
    }));
  });

  // Pass request ID in response header (for frontend debugging)
  res.setHeader("X-Request-Id", req.requestId);

  next(); // Move to next middleware
}

module.exports = loggerMiddleware;
```

**What this logs:**

```
→ REQUEST  { requestId: "abc-123", method: "POST", url: "/api/analytics/report", timestamp: "..." }
← RESPONSE { requestId: "abc-123", statusCode: 200, duration: "45ms" }
```

### 3.2 Auth Middleware (JWT Verification)

```javascript
// src/middleware/auth.js
const jwt = require("jsonwebtoken");
const AppError = require("../errors/AppError");

/**
 * WHY: Every request must prove "who is calling?"
 * 
 * Flow:
 *   React sends: Authorization: Bearer eyJhbGciOiJSUz...
 *   This middleware:
 *     1. Extracts the token
 *     2. Verifies it's valid and not expired
 *     3. Attaches user info to req.user
 *     4. If invalid → rejects with 401
 */
function authMiddleware(req, res, next) {
  // Step 1: Extract token from header
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    return next(new AppError("No token provided", 401));
  }

  const token = authHeader.split(" ")[1];

  try {
    // Step 2: Verify token
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    // Step 3: Attach user info to request (available in controller/service)
    req.user = {
      id: decoded.userId,
      email: decoded.email,
      role: decoded.role,      // e.g., "admin", "analyst", "viewer"
    };

    next(); // Token is valid → proceed
  } catch (error) {
    if (error.name === "TokenExpiredError") {
      return next(new AppError("Token expired, please login again", 401));
    }
    return next(new AppError("Invalid token", 401));
  }
}

/**
 * Role-based access control middleware.
 * Usage: router.post("/report", auth, authorize("admin", "analyst"), controller.create)
 */
function authorize(...allowedRoles) {
  return (req, res, next) => {
    if (!allowedRoles.includes(req.user.role)) {
      return next(new AppError("You don't have permission for this action", 403));
    }
    next();
  };
}

module.exports = { authMiddleware, authorize };
```

**Diagram: How JWT Auth Works**

```
┌─────────────────────────────────────────────────────────────┐
│                     JWT AUTH FLOW                             │
│                                                              │
│  1. User logs in:                                            │
│     React ──POST /login──► Node                              │
│     { email, password }                                      │
│                                                              │
│  2. Node verifies credentials, creates JWT:                  │
│     jwt.sign({ userId: 5, role: "analyst" }, SECRET)         │
│     Returns: { token: "eyJhbG..." }                          │
│                                                              │
│  3. React stores token (memory or httpOnly cookie)           │
│                                                              │
│  4. Every subsequent request includes token:                 │
│     React ──GET /api/report──► Node                          │
│     Header: Authorization: Bearer eyJhbG...                  │
│                                                              │
│  5. Auth middleware verifies:                                │
│     ┌──────────┐                                            │
│     │ Valid?    │──Yes──► req.user = { id: 5, role: "analyst" } │
│     │ Expired?  │──No───► 401 Unauthorized                   │
│     │ Tampered? │──No───► 401 Invalid token                  │
│     └──────────┘                                            │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 Validation Middleware

```javascript
// src/middleware/validate.js
const { validationResult, body, param } = require("express-validator");
const AppError = require("../errors/AppError");

/**
 * WHY: Never trust data from the frontend.
 * Validate BEFORE it reaches the controller.
 * 
 * Think of it like a bouncer:
 *   "Your name's not on the list" → 400 Bad Request
 */

// Reusable validation rules
const reportValidation = [
  body("dateRange")
    .notEmpty().withMessage("dateRange is required")
    .matches(/^\d{4}-\d{2}-\d{2}\/\d{4}-\d{2}-\d{2}$/)
    .withMessage("dateRange must be YYYY-MM-DD/YYYY-MM-DD"),
  body("metric")
    .notEmpty().withMessage("metric is required")
    .isIn(["revenue", "users", "inventory", "transactions"])
    .withMessage("metric must be one of: revenue, users, inventory, transactions"),
];

const jobIdValidation = [
  param("jobId")
    .notEmpty().withMessage("jobId is required")
    .isNumeric().withMessage("jobId must be a number"),
];

// Middleware that checks validation results
function handleValidationErrors(req, res, next) {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    const messages = errors.array().map(e => e.msg).join(", ");
    return next(new AppError(messages, 400));
  }
  next();
}

module.exports = { reportValidation, jobIdValidation, handleValidationErrors };
```

### 3.4 Rate Limiter Middleware

```javascript
// src/middleware/rateLimiter.js
const rateLimit = require("express-rate-limit");
const RedisStore = require("rate-limit-redis");
const redis = require("../config/redis");

/**
 * WHY: Prevent abuse.
 * Without this:
 *   - One user could trigger 1000 Databricks jobs per minute
 *   - Databricks costs $$$, so uncontrolled access = huge bills
 *   - DDoS attacks could bring down the system
 */

// General API rate limiter: 100 requests per 15 minutes per IP
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,   // 15 minutes
  max: 100,                    // 100 requests per window
  message: { error: "Too many requests, try again later" },
  standardHeaders: true,
  store: new RedisStore({ sendCommand: (...args) => redis.call(...args) }),
});

// Strict limiter for expensive Databricks jobs: 5 per hour per user
const jobTriggerLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,   // 1 hour
  max: 5,                     // Only 5 job triggers per hour
  keyGenerator: (req) => req.user?.id || req.ip, // Per user, not per IP
  message: { error: "Job trigger limit reached. Max 5 per hour." },
  store: new RedisStore({ sendCommand: (...args) => redis.call(...args) }),
});

module.exports = { apiLimiter, jobTriggerLimiter };
```

### How All Middleware Connects in app.js

```javascript
// src/app.js — Express app setup
const express = require("express");
const helmet = require("helmet");           // Security headers
const cors = require("cors");
const loggerMiddleware = require("./middleware/logger");
const { apiLimiter } = require("./middleware/rateLimiter");
const errorHandler = require("./middleware/errorHandler");
const analyticsRoutes = require("./routes/analyticsRoutes");
const jobRoutes = require("./routes/jobRoutes");

const app = express();

// ── GLOBAL MIDDLEWARE (runs on EVERY request) ──────────────
app.use(helmet());                      // Security headers
app.use(cors({ origin: process.env.FRONTEND_URL })); // CORS
app.use(express.json({ limit: "10kb" })); // Parse JSON (limit size)
app.use(loggerMiddleware);              // Log every request
app.use(apiLimiter);                    // Rate limit all APIs

// ── ROUTES ─────────────────────────────────────────────────
app.use("/api/analytics", analyticsRoutes);
app.use("/api/jobs", jobRoutes);

// ── 404 HANDLER ────────────────────────────────────────────
app.use((req, res) => {
  res.status(404).json({ error: "Route not found" });
});

// ── CENTRALIZED ERROR HANDLER (must be LAST) ──────────────
app.use(errorHandler);

module.exports = app;
```

**Diagram: Middleware Pipeline**

```
Request arrives at Express
         │
         ▼
┌─────────────────┐
│  helmet()        │ → Adds security headers (X-Frame-Options, etc.)
└────────┬────────┘
         ▼
┌─────────────────┐
│  cors()          │ → Only allows requests from your React domain
└────────┬────────┘
         ▼
┌─────────────────┐
│  express.json()  │ → Parses request body, limits to 10KB
└────────┬────────┘
         ▼
┌─────────────────┐
│  loggerMiddleware│ → Logs request, generates requestId
└────────┬────────┘
         ▼
┌─────────────────┐
│  apiLimiter      │ → Rejects if too many requests
└────────┬────────┘
         ▼
┌─────────────────┐
│  authMiddleware  │ → Verifies JWT (applied per route)
└────────┬────────┘
         ▼
┌─────────────────┐
│  validate        │ → Checks input format (applied per route)
└────────┬────────┘
         ▼
    CONTROLLER → SERVICE → DAO
         │
         ▼ (if any error thrown anywhere)
┌─────────────────┐
│  errorHandler    │ → Catches ALL errors, sends clean response
└─────────────────┘
```

---

## 4. CONTROLLER LAYER

### What Does a Controller Do?

```
Controller = The RECEPTIONIST
  1. Takes the incoming request
  2. Extracts what's needed (body, params, user info)
  3. Passes it to the right SERVICE
  4. Sends back the response

Controller does NOT:
  ❌ Contain business logic
  ❌ Call Databricks directly
  ❌ Handle caching
  ❌ Know about databases
```

```javascript
// src/controllers/analyticsController.js
const analyticsService = require("../services/analyticsService");

/**
 * Controller is THIN — it only handles HTTP concerns.
 * All logic lives in the service layer.
 */
class AnalyticsController {

  // POST /api/analytics/report
  async triggerReport(req, res, next) {
    try {
      const { dateRange, metric } = req.body; // Extracted from request
      const userId = req.user.id;             // From auth middleware

      // Pass to service — controller doesn't know HOW it works
      const result = await analyticsService.triggerReport({ dateRange, metric, userId });

      res.status(202).json(result); // 202 = Accepted (async)
    } catch (error) {
      next(error); // Pass to error handler
    }
  }

  // GET /api/analytics/status/:jobId
  async getJobStatus(req, res, next) {
    try {
      const { jobId } = req.params;
      const result = await analyticsService.getJobStatus(jobId);
      res.json(result);
    } catch (error) {
      next(error);
    }
  }

  // GET /api/analytics/result/:jobId
  async getJobResult(req, res, next) {
    try {
      const { jobId } = req.params;
      const userId = req.user.id;

      const result = await analyticsService.getJobResult(jobId, userId);
      res.json(result);
    } catch (error) {
      next(error);
    }
  }
}

module.exports = new AnalyticsController();
```

### Routes File (Connects Middleware → Controller)

```javascript
// src/routes/analyticsRoutes.js
const express = require("express");
const router = express.Router();
const { authMiddleware, authorize } = require("../middleware/auth");
const { reportValidation, jobIdValidation, handleValidationErrors } = require("../middleware/validate");
const { jobTriggerLimiter } = require("../middleware/rateLimiter");
const analyticsController = require("../controllers/analyticsController");

// Every route in this file requires authentication
router.use(authMiddleware);

// POST /api/analytics/report
// Pipeline: auth → rate limit → validate → controller
router.post(
  "/report",
  authorize("admin", "analyst"),    // Only admin/analyst can trigger
  jobTriggerLimiter,                // Max 5 per hour
  reportValidation,                 // Validate body
  handleValidationErrors,           // Check validation result
  analyticsController.triggerReport  // Finally → controller
);

// GET /api/analytics/status/:jobId
router.get(
  "/status/:jobId",
  jobIdValidation,
  handleValidationErrors,
  analyticsController.getJobStatus
);

// GET /api/analytics/result/:jobId
router.get(
  "/result/:jobId",
  authorize("admin", "analyst", "viewer"), // Viewers can see results
  jobIdValidation,
  handleValidationErrors,
  analyticsController.getJobResult
);

module.exports = router;
```

---

## 5. SERVICE LAYER

### What Does a Service Do?

```
Service = The BRAIN / MANAGER
  1. Contains all BUSINESS LOGIC
  2. Orchestrates between different DAOs
  3. Handles caching decisions
  4. DOES NOT know about HTTP (no req, res objects)

Think of it like a project manager:
  "Check the cache. If miss, trigger the job. Track the status. Return the result."
  The PM doesn't do the actual coding — they coordinate the developers (DAOs).
```

```javascript
// src/services/analyticsService.js
const databricksDao = require("../dao/databricksDao");
const redisDao = require("../dao/redisDao");
const config = require("../config");
const AppError = require("../errors/AppError");

class AnalyticsService {

  /**
   * Trigger a report generation job.
   * Business logic:
   *   1. Check if this exact report was already generated (cache)
   *   2. If not, trigger a Databricks job
   *   3. Track the job in Redis
   *   4. Return job ID to caller
   */
  async triggerReport({ dateRange, metric, userId }) {
    // Step 1: Check cache — maybe this report already exists
    const cacheKey = `report:${metric}:${dateRange}`;
    const cached = await redisDao.get(cacheKey);

    if (cached) {
      return {
        status: "COMPLETED",
        result: JSON.parse(cached),
        source: "cache",
      };
    }

    // Step 2: Get job config (which Databricks job to run)
    const jobConfig = config.databricksJobs[metric];
    if (!jobConfig) {
      throw new AppError(`Unknown metric: ${metric}`, 400);
    }

    // Step 3: Trigger Databricks job
    const runId = await databricksDao.triggerJob(jobConfig.jobId, {
      date_range: dateRange,
      metric: metric,
      requested_by: userId,
    });

    // Step 4: Track in Redis
    await redisDao.setWithExpiry(`job:${runId}`, 3600, {
      status: "PROCESSING",
      createdAt: Date.now(),
      userId,
      params: { dateRange, metric },
    });

    // Step 5: Return job ID (async pattern — don't wait for Databricks!)
    return {
      jobId: runId,
      status: "PROCESSING",
      message: "Report generation started. Poll /status/:jobId for updates.",
    };
  }

  /**
   * Check job status.
   * Business logic:
   *   1. Check Redis first (avoid hitting Databricks too often)
   *   2. If not in cache or still processing, check Databricks
   *   3. If done, cache the result
   */
  async getJobStatus(jobId) {
    // Step 1: Check Redis
    const cached = await redisDao.get(`job:${jobId}`);
    if (cached) {
      const parsed = JSON.parse(cached);
      if (parsed.status === "COMPLETED" || parsed.status === "FAILED") {
        return parsed;
      }
    }

    // Step 2: Check Databricks
    const status = await databricksDao.getJobStatus(jobId);

    if (status.lifeCycleState === "TERMINATED") {
      if (status.resultState === "SUCCESS") {
        const output = await databricksDao.getJobOutput(jobId);

        // Step 3: Cache result
        const result = { status: "COMPLETED", result: output };
        await redisDao.setWithExpiry(`job:${jobId}`, 3600, result);

        // Also cache by params for future identical requests
        if (cached) {
          const { params } = JSON.parse(cached);
          await redisDao.setWithExpiry(`report:${params.metric}:${params.dateRange}`, 3600, output);
        }

        return result;
      }

      return { status: "FAILED", error: "Databricks job failed" };
    }

    return { status: "PROCESSING", databricksState: status.lifeCycleState };
  }

  /**
   * Get final results for a completed job.
   */
  async getJobResult(jobId, userId) {
    const job = await redisDao.get(`job:${jobId}`);
    if (!job) throw new AppError("Job not found", 404);

    const parsed = JSON.parse(job);

    // Security: Only the user who triggered the job can see results
    if (parsed.userId && parsed.userId !== userId) {
      throw new AppError("You don't have access to this job", 403);
    }

    if (parsed.status !== "COMPLETED") {
      throw new AppError("Job is not completed yet", 400);
    }

    return parsed;
  }
}

module.exports = new AnalyticsService();
```

---

## 6. DAO (Data Access) LAYER

### What Does a DAO Do?

```
DAO = The WORKER who actually talks to external systems
  1. Talks to Databricks API
  2. Talks to Redis
  3. Talks to Database
  4. DOES NOT contain business logic
  5. Can be swapped out without changing service layer

Think of it like a delivery driver:
  "Go to Databricks, pick up the data, bring it back."
  The driver doesn't decide WHAT to fetch — the service tells them.
```

### 6.1 Databricks DAO

```javascript
// src/dao/databricksDao.js
const axios = require("axios");
const config = require("../config");
const AppError = require("../errors/AppError");

/**
 * This DAO handles ALL communication with Databricks.
 * If Databricks API changes, ONLY this file changes.
 * Service layer doesn't know or care about API details.
 */
class DatabricksDao {
  constructor() {
    this.client = axios.create({
      baseURL: `${config.databricks.host}/api/2.1`,
      headers: {
        Authorization: `Bearer ${config.databricks.token}`,
        "Content-Type": "application/json",
      },
      timeout: 30000, // 30 second timeout for API calls
    });
  }

  async triggerJob(jobId, parameters) {
    try {
      const response = await this.client.post("/jobs/run-now", {
        job_id: jobId,
        notebook_params: parameters,
      });
      return response.data.run_id;
    } catch (error) {
      throw new AppError(
        `Failed to trigger Databricks job: ${error.message}`,
        502  // 502 = Bad Gateway (upstream service failed)
      );
    }
  }

  async getJobStatus(runId) {
    try {
      const response = await this.client.get(`/jobs/runs/get?run_id=${runId}`);
      return {
        lifeCycleState: response.data.state.life_cycle_state,
        resultState: response.data.state.result_state,
      };
    } catch (error) {
      throw new AppError(`Failed to get job status: ${error.message}`, 502);
    }
  }

  async getJobOutput(runId) {
    try {
      const response = await this.client.get(`/jobs/runs/get-output?run_id=${runId}`);
      return response.data;
    } catch (error) {
      throw new AppError(`Failed to get job output: ${error.message}`, 502);
    }
  }

  // For quick interactive queries (not long-running jobs)
  async runSqlQuery(query) {
    try {
      const response = await this.client.post("/sql/statements", {
        warehouse_id: config.databricks.warehouseId,
        statement: query,
        wait_timeout: "30s",
      });
      return response.data;
    } catch (error) {
      throw new AppError(`SQL query failed: ${error.message}`, 502);
    }
  }
}

module.exports = new DatabricksDao();
```

### 6.2 Redis DAO

```javascript
// src/dao/redisDao.js
const redis = require("../config/redis");

class RedisDao {
  async get(key) {
    return await redis.get(key);
  }

  async setWithExpiry(key, ttlSeconds, value) {
    await redis.setex(key, ttlSeconds, JSON.stringify(value));
  }

  async delete(key) {
    await redis.del(key);
  }

  async exists(key) {
    return await redis.exists(key);
  }
}

module.exports = new RedisDao();
```

---

## 7. CENTRALIZED ERROR HANDLER

### Why Centralized Error Handling?

```
❌ BAD: Error handling scattered everywhere
  Every controller has its own try/catch with different formats
  Some errors return { error: "..." }, others return { message: "..." }
  Frontend doesn't know what to expect

✅ GOOD: ONE error handler catches EVERYTHING
  Consistent error format
  Logs full details for debugging
  Sends clean response to frontend
  Handles all error types (validation, auth, Databricks, unknown)
```

### Custom Error Class

```javascript
// src/errors/AppError.js

/**
 * Custom error class that carries HTTP status code.
 * 
 * Usage:
 *   throw new AppError("Job not found", 404);
 *   throw new AppError("Databricks unavailable", 502);
 */
class AppError extends Error {
  constructor(message, statusCode = 500, details = null) {
    super(message);
    this.statusCode = statusCode;
    this.details = details;
    this.isOperational = true; // Distinguishes expected errors from bugs
  }
}

module.exports = AppError;
```

### Centralized Error Handler Middleware

```javascript
// src/middleware/errorHandler.js

/**
 * This is the LAST middleware in Express.
 * It catches ALL errors from any layer.
 * 
 * Flow:
 *   Controller throws → Express calls next(error) → THIS catches it
 *   Service throws → Controller's catch calls next(error) → THIS catches it
 *   DAO throws → Service throws → Controller catches → THIS catches it
 */
function errorHandler(err, req, res, next) {

  // ── 1. LOG FULL ERROR (for debugging) ──────────────────
  console.error(JSON.stringify({
    type: "ERROR",
    requestId: req.requestId,   // From logger middleware
    message: err.message,
    statusCode: err.statusCode || 500,
    stack: err.stack,            // Full stack trace (NEVER send to frontend)
    url: req.originalUrl,
    method: req.method,
    userId: req.user?.id,
    timestamp: new Date().toISOString(),
  }));

  // ── 2. DETERMINE ERROR TYPE ──────────────────────────────

  // Known operational error (we threw it intentionally)
  if (err.isOperational) {
    return res.status(err.statusCode).json({
      error: err.message,
      requestId: req.requestId,   // User can share this for support
      ...(err.details && { details: err.details }),
    });
  }

  // JWT errors
  if (err.name === "JsonWebTokenError") {
    return res.status(401).json({
      error: "Invalid token",
      requestId: req.requestId,
    });
  }

  // Validation errors (express-validator)
  if (err.name === "ValidationError") {
    return res.status(400).json({
      error: "Invalid input",
      details: err.details,
      requestId: req.requestId,
    });
  }

  // ── 3. UNKNOWN ERROR (bug in our code) ──────────────────
  // NEVER expose internal details to the frontend!
  res.status(500).json({
    error: "Something went wrong. Please try again.",
    requestId: req.requestId,
  });
}

module.exports = errorHandler;
```

**Diagram: Error Propagation**

```
DAO throws "Databricks timeout"
      │
      ▼
Service catches, wraps: throw new AppError("Job trigger failed", 502)
      │
      ▼
Controller catches: next(error)
      │
      ▼
┌─────────────────────────────────────────────────────┐
│           CENTRALIZED ERROR HANDLER                  │
│                                                      │
│  1. Logs FULL error (stack trace, user, URL)         │
│     → Goes to logging system (ELK, CloudWatch)       │
│                                                      │
│  2. Sends CLEAN response to React:                   │
│     {                                                │
│       "error": "Job trigger failed",                 │
│       "requestId": "abc-123"                         │
│     }                                                │
│                                                      │
│  React shows: "Something went wrong. Ref: abc-123"  │
│  Support team searches logs by requestId: "abc-123"  │
└─────────────────────────────────────────────────────┘
```

---

## 8. SECURITY — DEEP DIVE

### Security at Every Layer

```
┌──────────────────────────────────────────────────────────────────┐
│                    SECURITY BY LAYER                               │
│                                                                   │
│  REACT (Browser)                                                  │
│  ├── HTTPS only (encrypted transport)                            │
│  ├── HttpOnly cookies (prevents XSS token theft)                 │
│  ├── CSP headers (Content Security Policy)                       │
│  ├── No secrets in frontend code                                 │
│  └── Sanitize rendered content (prevent XSS)                     │
│                                                                   │
│  NODE.JS (Server)                                                │
│  ├── JWT verification on every request                           │
│  ├── RBAC (Role-Based Access Control)                            │
│  ├── Rate limiting (prevent abuse + cost control)                │
│  ├── Input validation (prevent injection)                        │
│  ├── Helmet.js (security headers)                                │
│  ├── CORS (only allow your React domain)                         │
│  ├── Request size limits (prevent payload attacks)               │
│  └── Service account for Databricks (not user tokens)            │
│                                                                   │
│  DATABRICKS                                                      │
│  ├── Personal Access Token (PAT) or OAuth (M2M)                 │
│  ├── IP allowlisting (only Node server IPs can connect)          │
│  ├── Table-level access controls (ACLs)                          │
│  ├── Encrypted at rest (Delta Lake)                              │
│  └── Audit logging (who ran what job)                            │
│                                                                   │
│  NETWORK                                                         │
│  ├── VPC / Private network between Node and Databricks           │
│  ├── No public exposure of Databricks                            │
│  └── TLS 1.2+ everywhere                                        │
└──────────────────────────────────────────────────────────────────┘
```

### 8.1 What is JWT? (Easy Explanation)

```
JWT = JSON Web Token = A signed ID card

Think of it like a hotel key card:
  ┌─────────────────────────────────────┐
  │  HOTEL KEY CARD (JWT)               │
  │                                     │
  │  Name: Neelesh                      │
  │  Room: 305                          │  ← This is the "payload"
  │  Role: VIP Guest                    │
  │  Check-out: April 20               │  ← Expiry
  │                                     │
  │  Signature: ██████████████████      │  ← Can't be faked
  └─────────────────────────────────────┘

  - The hotel (Node.js) creates this card when you check in (login)
  - You show it to access any room (API endpoint)
  - Staff can verify it's real by checking the signature
  - It expires automatically
  - If someone changes the Room number, the signature breaks → REJECTED
```

### JWT Structure (3 Parts)

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjV9.abc123signature
─────── HEADER ───────.──── PAYLOAD ────.──── SIGNATURE ──

HEADER:   { "alg": "HS256", "typ": "JWT" }
PAYLOAD:  { "userId": 5, "role": "analyst", "exp": 1713398400 }
SIGNATURE: HMAC-SHA256(header + payload, SECRET_KEY)

The SECRET_KEY lives ONLY on the server.
Anyone can READ the payload (it's just base64).
But NO ONE can MODIFY it without the secret key.
```

### 8.2 What is RBAC? (Role-Based Access Control)

```
RBAC = Different users have different permissions

ROLES AND PERMISSIONS TABLE:
┌────────────┬─────────────────┬──────────────┬──────────────┐
│  Action    │  Admin          │  Analyst     │  Viewer      │
├────────────┼─────────────────┼──────────────┼──────────────┤
│  Trigger   │  ✅ Yes         │  ✅ Yes      │  ❌ No       │
│  job       │                 │              │              │
├────────────┼─────────────────┼──────────────┼──────────────┤
│  View      │  ✅ All jobs    │  ✅ Own jobs │  ✅ Own jobs │
│  status    │                 │              │              │
├────────────┼─────────────────┼──────────────┼──────────────┤
│  Delete    │  ✅ Yes         │  ❌ No       │  ❌ No       │
│  job       │                 │              │              │
├────────────┼─────────────────┼──────────────┼──────────────┤
│  Manage    │  ✅ Yes         │  ❌ No       │  ❌ No       │
│  users     │                 │              │              │
└────────────┴─────────────────┴──────────────┴──────────────┘

In code:
  router.post("/report", auth, authorize("admin", "analyst"), controller);
  //                            ↑ Only these roles can trigger jobs
```

### 8.3 OAuth2 — What Is It?

```
OAuth2 = "Login with Google/GitHub" but also "Service-to-Service auth"

TWO main uses:

1. USER LOGIN (Authorization Code Flow):
   ┌──────┐     ┌──────────┐     ┌──────────┐
   │ User │────►│  React    │────►│  Google   │
   │      │     │  "Login   │     │  "Allow   │
   │      │     │   with    │     │   this    │
   │      │     │   Google" │     │   app?"   │
   │      │     └──────────┘     └────┬─────┘
   │      │                           │ auth code
   │      │     ┌──────────┐          │
   │      │     │  Node.js  │◄────────┘
   │      │◄────│  exchanges│────► Gets user info from Google
   │      │     │  code for │      Creates JWT for your app
   │      │     │  token    │
   └──────┘     └──────────┘

2. SERVICE-TO-SERVICE (Client Credentials Flow):
   Used for Node.js → Databricks communication (see Section 10)
```

### 8.4 What is CORS? (Easy Explanation)

```
CORS = Cross-Origin Resource Sharing

Problem:
  Your React runs on:  https://myapp.com
  Your Node API is on: https://api.myapp.com
  
  Browsers BLOCK requests between different domains by default!
  This is a SECURITY feature (prevents evil sites from calling your API).

Solution: CORS headers tell the browser "it's OK, I trust this origin"

Node.js code:
  app.use(cors({
    origin: "https://myapp.com",   // ONLY allow your React app
    methods: ["GET", "POST"],       // ONLY these methods
    credentials: true,              // Allow cookies
  }));

What happens:
  Browser → "Hey API, can myapp.com talk to you?"
  API → "Yes, myapp.com is allowed" (CORS header)
  Browser → "OK, I'll allow the request"

  Browser → "Hey API, can evil-site.com talk to you?"
  API → (no CORS header for evil-site.com)
  Browser → "BLOCKED! ❌"
```

### 8.5 What Are Security Headers (Helmet)?

```javascript
app.use(helmet());  // One line, adds ALL these headers:

// What Helmet adds:
{
  "X-Content-Type-Options": "nosniff",      // Don't guess file types
  "X-Frame-Options": "DENY",                 // Prevent clickjacking
  "X-XSS-Protection": "1; mode=block",       // XSS protection
  "Strict-Transport-Security": "max-age=...", // Force HTTPS
  "Content-Security-Policy": "...",           // Control resource loading
}

Easy analogy:
  Helmet = Wearing a seatbelt, airbag, and crash helmet at once
  One line of code, multiple protections.
```

---

## 9. MICROSERVICES COMMUNICATION — Dashboard + Jobs Example

### Scenario

```
You have TWO separate microservices:

┌──────────────────┐        ┌──────────────────┐
│  DASHBOARD        │        │  JOBS             │
│  SERVICE          │        │  SERVICE          │
│                   │        │                   │
│  - User UI        │        │  - Trigger jobs   │
│  - Show charts    │        │  - Track status   │
│  - Show status    │        │  - Talk to        │
│  - Authentication │        │    Databricks     │
│                   │        │                   │
│  Port: 3001       │        │  Port: 3002       │
└──────────────────┘        └──────────────────┘

Question: How do they talk to each other SECURELY?
```

### 3 Communication Patterns

#### Pattern 1: Synchronous (HTTP/REST) — Simple, Direct

```
Dashboard Service ──HTTP──► Jobs Service
                   ◄─────── Response

When to use: Simple request/response, low latency needed
```

```javascript
// Dashboard Service calling Jobs Service

// ❌ BAD: Dashboard calls Databricks directly
const result = await axios.get("https://databricks.com/api/jobs/...");

// ✅ GOOD: Dashboard calls Jobs Service (which handles Databricks)
const result = await axios.get("http://jobs-service:3002/internal/jobs/status/123", {
  headers: {
    Authorization: `Bearer ${internalServiceToken}`,  // Service-to-service token
  },
});
```

#### Pattern 2: Asynchronous (Message Queue) — Decoupled, Reliable

```
Dashboard ──► Message Queue (RabbitMQ/Redis) ──► Jobs Service
                                                    │
              (Dashboard doesn't wait)              │
                                                    ▼
              Message Queue ◄── "Job done!" ◄── Jobs Service
                    │
                    ▼
              Dashboard reads the result
```

```javascript
// Dashboard Service — SENDS a message to the queue
const { Queue } = require("bullmq");
const jobQueue = new Queue("analytics-jobs", { connection: redisConfig });

// When user requests a report:
await jobQueue.add("generate-report", {
  dateRange: "2024-01-01/2024-12-31",
  metric: "revenue",
  userId: 5,
});
// Returns immediately! Dashboard doesn't wait.

// ──────────────────────────────────────────────────

// Jobs Service — PROCESSES messages from the queue
const { Worker } = require("bullmq");

const worker = new Worker("analytics-jobs", async (job) => {
  const { dateRange, metric, userId } = job.data;

  // Trigger Databricks
  const runId = await databricksDao.triggerJob(config.jobs[metric].jobId, {
    date_range: dateRange,
    metric: metric,
  });

  // Wait for completion (worker can wait, HTTP can't)
  const result = await pollUntilComplete(runId);

  // Store result — Dashboard polls for this
  await redisDao.setWithExpiry(`job:${job.id}`, 3600, {
    status: "COMPLETED",
    result,
    userId,
  });
}, { connection: redisConfig });
```

#### Pattern 3: Event-Driven (Pub/Sub) — Real-Time Updates

```
┌──────────────┐    "job.completed"    ┌──────────────┐
│ Jobs Service  │────── publishes ─────►│ Event Bus    │
│               │      event           │ (Redis PubSub│
└──────────────┘                       │  / Kafka)    │
                                       └──────┬───────┘
                                              │ subscribers
                                    ┌─────────┴─────────┐
                                    ▼                   ▼
                             ┌──────────┐        ┌──────────┐
                             │Dashboard │        │Notification│
                             │Service   │        │Service     │
                             │(updates  │        │(sends      │
                             │ UI)      │        │ email)     │
                             └──────────┘        └──────────┘
```

```javascript
// Jobs Service — PUBLISHES event when job is done
const redis = require("../config/redis");

async function onJobComplete(jobId, result) {
  await redis.publish("job-events", JSON.stringify({
    type: "JOB_COMPLETED",
    jobId,
    result,
    timestamp: Date.now(),
  }));
}

// ──────────────────────────────────────────────────

// Dashboard Service — SUBSCRIBES to events
const subscriber = redis.duplicate();
await subscriber.subscribe("job-events");

subscriber.on("message", (channel, message) => {
  const event = JSON.parse(message);

  if (event.type === "JOB_COMPLETED") {
    // Notify connected React clients via WebSocket
    websocketServer.notifyUser(event.userId, {
      type: "JOB_DONE",
      jobId: event.jobId,
      result: event.result,
    });
  }
});
```

### How Microservices Authenticate with Each Other

```
PROBLEM: Dashboard calls Jobs Service. How does Jobs Service
         know it's really Dashboard (not an attacker)?

3 SOLUTIONS:
```

#### Solution 1: Internal JWT (Service-to-Service Token)

```javascript
// Each service has its own service account credentials
// A central auth service issues tokens

// Dashboard Service requesting Jobs Service:
const serviceToken = jwt.sign(
  { service: "dashboard", permissions: ["read:jobs", "create:jobs"] },
  process.env.INTERNAL_JWT_SECRET,
  { expiresIn: "5m" }     // Short-lived!
);

const response = await axios.get("http://jobs-service:3002/internal/jobs/123", {
  headers: { Authorization: `Bearer ${serviceToken}` },
});

// Jobs Service validates:
function internalAuthMiddleware(req, res, next) {
  const token = req.headers.authorization?.split(" ")[1];
  const decoded = jwt.verify(token, process.env.INTERNAL_JWT_SECRET);

  if (decoded.service !== "dashboard") {
    return res.status(403).json({ error: "Unknown service" });
  }

  if (!decoded.permissions.includes("read:jobs")) {
    return res.status(403).json({ error: "Insufficient permissions" });
  }

  req.callingService = decoded.service;
  next();
}
```

#### Solution 2: mTLS (Mutual TLS) — Both Sides Verify Certificates

```
Normal TLS:  Client verifies server  (browser checks website cert)
mTLS:        Client AND server verify EACH OTHER

┌─────────────────┐                ┌─────────────────┐
│ Dashboard Service│                │  Jobs Service    │
│                  │                │                  │
│  Has:            │                │  Has:            │
│  - Its own cert  │───── mTLS ────│  - Its own cert  │
│  - CA cert       │    (mutual)   │  - CA cert       │
│                  │    (verify)   │                  │
└─────────────────┘                └─────────────────┘

Both sides present certificates → both verify → connection established
No tokens needed! The certificate IS the identity.
```

```javascript
// mTLS in Node.js
const https = require("https");
const fs = require("fs");

// Jobs Service (SERVER side)
const server = https.createServer({
  key: fs.readFileSync("./certs/jobs-service.key"),
  cert: fs.readFileSync("./certs/jobs-service.cert"),
  ca: fs.readFileSync("./certs/ca.cert"),      // Certificate Authority
  requestCert: true,                             // Require client cert!
  rejectUnauthorized: true,                      // Reject invalid certs
});

// Dashboard Service (CLIENT side)
const agent = new https.Agent({
  key: fs.readFileSync("./certs/dashboard-service.key"),
  cert: fs.readFileSync("./certs/dashboard-service.cert"),
  ca: fs.readFileSync("./certs/ca.cert"),
});

const response = await axios.get("https://jobs-service:3002/internal/jobs/123", {
  httpsAgent: agent,
});
```

#### Solution 3: API Gateway (Centralized)

```
All microservices sit BEHIND a gateway:

React ──► API Gateway ──► Dashboard Service
                      ──► Jobs Service
                      ──► Notification Service

The gateway handles:
  ✅ Authentication (verifies JWT/API key)
  ✅ Routing (which service handles which URL)
  ✅ Rate limiting
  ✅ Load balancing
  ✅ Service discovery

Internal services trust the gateway.
They ONLY accept traffic from the gateway (network rules).
```

```
┌──────────────────────────────────────────────────────────────┐
│                   MICROSERVICES AUTH DIAGRAM                   │
│                                                               │
│  ┌────────┐                                                  │
│  │ React  │──── User JWT ────►┌────────────┐                │
│  │(Browser)│                   │ API Gateway │                │
│  └────────┘                   │            │                │
│                                │ Validates  │                │
│                                │ user JWT   │                │
│                                └──────┬─────┘                │
│                              ┌────────┼────────┐             │
│                              │        │        │             │
│                              ▼        ▼        ▼             │
│                         ┌────────┐ ┌────────┐ ┌────────┐    │
│                         │Dashboard│ │  Jobs  │ │Notif.  │    │
│                         │Service │ │Service │ │Service │    │
│                         └───┬────┘ └───┬────┘ └────────┘    │
│                             │          │                     │
│   Internal calls:           │  mTLS or │                     │
│   (service-to-service JWT   │ internal │                     │
│    or mTLS)                 └──────────┘                     │
│                                                               │
│   Databricks calls:                                          │
│   (Service Account Token or OAuth M2M)                       │
│                              │                               │
│                              ▼                               │
│                         ┌──────────┐                         │
│                         │Databricks│                         │
│                         └──────────┘                         │
└──────────────────────────────────────────────────────────────┘
```

### Complete Microservice Communication Table

```
┌──────────────────────┬─────────────────┬─────────────────────────┐
│  WHO → WHO           │  Auth Method    │  Why This Choice        │
├──────────────────────┼─────────────────┼─────────────────────────┤
│  React → Node API    │  User JWT       │  Identifies the USER    │
│  Dashboard → Jobs    │  Service JWT    │  Identifies the SERVICE │
│  Node → Databricks   │  OAuth M2M /    │  Machine-to-machine     │
│                      │  Service Token  │  (no user involved)     │
│  Node → Redis        │  Password +     │  In private network     │
│                      │  Private network│                         │
│  Node → Database     │  Connection     │  In private network     │
│                      │  string + TLS   │                         │
└──────────────────────┴─────────────────┴─────────────────────────┘
```

---

## 10. SECURE NODE.JS ↔ DATABRICKS COMMUNICATION

### The Problem

```
Node.js needs to call Databricks APIs.
But Databricks has your company's most valuable data!

If an attacker gets the Databricks credentials:
  - They can read ALL your data
  - They can run expensive jobs ($$$)
  - They can delete tables

So we need MULTIPLE layers of protection.
```

### Layer 1: Authentication (WHO is calling?)

#### Option A: Personal Access Token (PAT) — Simpler

```javascript
// .env (NEVER commit this file!)
DATABRICKS_TOKEN=dapi1234567890abcdef

// databricksDao.js
const client = axios.create({
  baseURL: `${process.env.DATABRICKS_HOST}/api/2.1`,
  headers: {
    Authorization: `Bearer ${process.env.DATABRICKS_TOKEN}`,
  },
});
```

```
PROS: Simple to set up
CONS: Token doesn't expire automatically
      One token = full access
      If leaked, attacker has access until you manually revoke
```

#### Option B: OAuth Machine-to-Machine (M2M) — More Secure

```javascript
// OAuth M2M flow: get a SHORT-LIVED token using client credentials

class DatabricksAuth {
  constructor() {
    this.clientId = process.env.DATABRICKS_CLIENT_ID;
    this.clientSecret = process.env.DATABRICKS_CLIENT_SECRET;
    this.tokenUrl = `${process.env.DATABRICKS_HOST}/oidc/v1/token`;
    this.cachedToken = null;
    this.tokenExpiry = 0;
  }

  async getToken() {
    // Return cached token if still valid (with 5 min buffer)
    if (this.cachedToken && Date.now() < this.tokenExpiry - 300000) {
      return this.cachedToken;
    }

    // Request new token
    const response = await axios.post(this.tokenUrl, 
      new URLSearchParams({
        grant_type: "client_credentials",
        client_id: this.clientId,
        client_secret: this.clientSecret,
        scope: "all-apis",
      }),
      { headers: { "Content-Type": "application/x-www-form-urlencoded" } }
    );

    this.cachedToken = response.data.access_token;
    this.tokenExpiry = Date.now() + (response.data.expires_in * 1000);
    return this.cachedToken;
  }
}

// Usage in DAO:
class DatabricksDao {
  constructor() {
    this.auth = new DatabricksAuth();
  }

  async triggerJob(jobId, params) {
    const token = await this.auth.getToken();  // Always fresh token
    
    const response = await axios.post(
      `${process.env.DATABRICKS_HOST}/api/2.1/jobs/run-now`,
      { job_id: jobId, notebook_params: params },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    return response.data.run_id;
  }
}
```

```
OAuth M2M Flow Diagram:

┌──────────┐    1. ClientID + Secret     ┌──────────────────┐
│ Node.js   │──────────────────────────►│ Databricks Auth   │
│ (Jobs     │                           │ Server            │
│  Service) │    2. Short-lived token    │ (OAuth endpoint)  │
│           │◄──────────────────────────│                   │
│           │    (expires in 1 hour)     └──────────────────┘
│           │
│           │    3. Use token for API calls
│           │──────────────────────────►┌──────────────────┐
│           │    Authorization: Bearer   │ Databricks API    │
│           │    <short-lived-token>     │ (Jobs, SQL, etc.) │
│           │◄──────────────────────────│                   │
└──────────┘    4. Response              └──────────────────┘

WHY BETTER:
  - Token expires automatically (1 hour)
  - If leaked, limited damage window
  - Can be rotated without service restart
  - Audit trail of token issuance
```

### Layer 2: Network Security (WHERE can calls come from?)

```
┌─────────────────────────────────────────────────────────────┐
│                    NETWORK SECURITY                          │
│                                                              │
│  ┌──────────────────────────────────────────────────┐       │
│  │          YOUR VPC / Virtual Network               │       │
│  │                                                   │       │
│  │   ┌──────────┐        ┌──────────────────┐       │       │
│  │   │ Node.js  │───────►│ Databricks       │       │       │
│  │   │ Servers  │ Private│ (Private Link /  │       │       │
│  │   │          │ Network│  VPC Endpoint)    │       │       │
│  │   └──────────┘        └──────────────────┘       │       │
│  │                                                   │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
│  INTERNET ──── ❌ BLOCKED ────► Databricks                   │
│  (Hackers can't reach Databricks even WITH a valid token)    │
│                                                              │
│  IP Allowlisting:                                            │
│    Databricks config: Only accept from 10.0.1.0/24           │
│    (Your Node.js server subnet)                              │
└─────────────────────────────────────────────────────────────┘

Think of it like:
  Token = Door key
  Network = The building is in a gated community
  
  Even if someone steals the key, they can't reach the building!
```

### Layer 3: Least Privilege (WHAT can the caller do?)

```
The Node.js service account should have MINIMUM permissions:

❌ BAD: Admin token that can do anything
  - Read all tables ✅
  - Delete all tables ✅  ← DANGEROUS!
  - Create users ✅       ← DANGEROUS!
  - Drop databases ✅     ← DANGEROUS!

✅ GOOD: Scoped token with only what's needed
  - Run specific jobs ✅
  - Read report_results table ✅
  - Read job status ✅
  - Delete tables ❌       ← Not allowed
  - Manage clusters ❌     ← Not allowed
  - Access raw data ❌     ← Not allowed
```

### Layer 4: Secret Management (HOW are credentials stored?)

```
❌ BAD: Secrets in code or .env file
  const TOKEN = "dapi1234567890";  // In source code! 😱
  
  .env file committed to git!      // Everyone can see! 😱

✅ GOOD: Use a secret manager
  - AWS Secrets Manager
  - Azure Key Vault
  - HashiCorp Vault
  - GCP Secret Manager

  Code reads secrets at startup, never stored in files:
```

```javascript
// Using Azure Key Vault
const { SecretClient } = require("@azure/keyvault-secrets");
const { DefaultAzureCredential } = require("@azure/identity");

const client = new SecretClient(
  "https://my-keyvault.vault.azure.net",
  new DefaultAzureCredential()
);

async function loadSecrets() {
  const databricksToken = await client.getSecret("databricks-token");
  const jwtSecret = await client.getSecret("jwt-secret");
  
  return {
    databricksToken: databricksToken.value,
    jwtSecret: jwtSecret.value,
  };
}

// Secrets are:
// ✅ Encrypted at rest
// ✅ Access-controlled (only your app can read them)
// ✅ Audited (who accessed what, when)
// ✅ Rotatable without redeploying
```

### Complete Node.js → Databricks Security Summary

```
┌─────────────────────────────────────────────────────────────┐
│         SECURE NODE → DATABRICKS COMMUNICATION               │
│                                                              │
│  Layer 1: AUTHENTICATION                                     │
│  ├── OAuth M2M (client_credentials) for short-lived tokens  │
│  └── Fallback: PAT with regular rotation                    │
│                                                              │
│  Layer 2: NETWORK                                            │
│  ├── Private Link / VPC peering (no public internet)        │
│  ├── IP allowlisting                                        │
│  └── TLS 1.2+ encryption in transit                         │
│                                                              │
│  Layer 3: AUTHORIZATION                                      │
│  ├── Least privilege (only needed permissions)              │
│  ├── Scoped to specific jobs and tables                     │
│  └── RBAC within Databricks workspace                       │
│                                                              │
│  Layer 4: SECRETS                                            │
│  ├── Secrets in Key Vault (never in code or env files)      │
│  ├── Auto-rotation                                          │
│  └── Audit logging                                          │
│                                                              │
│  Layer 5: MONITORING                                         │
│  ├── Log every Databricks API call                          │
│  ├── Alert on unusual patterns (too many jobs, etc.)        │
│  └── Databricks audit logs                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 11. FULL END-TO-END FLOW (Everything Together)

```
USER clicks "Generate Report" in React
         │
         ▼
┌─ REACT ──────────────────────────────────────────────────────┐
│  POST /api/analytics/report                                   │
│  Headers: { Authorization: "Bearer <user-jwt>" }             │
│  Body: { dateRange: "2024-01-01/2024-12-31", metric: "revenue" } │
└──────────────────────┬───────────────────────────────────────┘
                       │ HTTPS
                       ▼
┌─ NODE.JS ── MIDDLEWARE PIPELINE ─────────────────────────────┐
│                                                               │
│  1. Logger       → requestId: "req-abc-123"                  │
│  2. Auth         → Verify JWT → req.user = { id: 5 }        │
│  3. Authorize    → user.role = "analyst" → ALLOWED           │
│  4. Rate Limiter → 3rd request this hour → ALLOWED (max 5)  │
│  5. Validator    → dateRange ✅, metric ✅                   │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌─ CONTROLLER ─────────────────────────────────────────────────┐
│  Extract: { dateRange, metric } from req.body                │
│  Extract: userId from req.user                               │
│  Call: analyticsService.triggerReport(...)                    │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌─ SERVICE ────────────────────────────────────────────────────┐
│  1. Check Redis cache → MISS                                 │
│  2. Look up Databricks job config for "revenue"              │
│  3. Call databricksDao.triggerJob(12345, params)              │
│  4. Store job:run-789 → PROCESSING in Redis                  │
│  5. Return { jobId: "run-789", status: "PROCESSING" }        │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌─ DAO (Databricks) ──────────────────────────────────────────┐
│  1. Get OAuth M2M token (cached or refresh)                  │
│  2. POST https://databricks.company.com/api/2.1/jobs/run-now │
│     Headers: { Authorization: Bearer <service-token> }       │
│     Body: { job_id: 12345, notebook_params: {...} }          │
│  3. Receive: { run_id: "run-789" }                           │
│  4. Connection: Private network (VPC), TLS 1.2+             │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌─ DATABRICKS ─────────────────────────────────────────────────┐
│  Spark cluster auto-scales → processes 10B rows              │
│  Writes result to Delta Lake table: report_results           │
│  Job status: TERMINATED / SUCCESS                            │
└──────────────────────────────────────────────────────────────┘
                       │
         (React polls /status/run-789 every 3s)
                       │
         Eventually → result returned → React renders chart
```

---

## 12. UPDATED 90-SECOND INTERVIEW ANSWER

> "I'd design a **layered architecture** with React as a thin UI, Node.js as a BFF and orchestrator, and Databricks for async data processing. React never talks to Databricks directly.
>
> On the **Node.js side**, I follow strict separation of concerns: **middleware** handles cross-cutting concerns like JWT auth, request logging with trace IDs, input validation, and rate limiting. **Controllers** handle HTTP request/response only. **Services** contain business logic and orchestration. **DAOs** encapsulate all external API calls to Databricks and Redis. A **centralized error handler** ensures consistent error responses.
>
> For **Databricks communication**, Node triggers jobs via REST API using **OAuth M2M tokens** — short-lived, auto-refreshable, and scoped to minimum permissions. The connection goes through a **private network** with IP allowlisting, never over public internet. Secrets are stored in **Key Vault**, not in code.
>
> For **microservices**, if Dashboard and Jobs are separate services, they communicate via **REST with service-to-service JWTs** for synchronous calls, or **message queues** for async job processing, with **mTLS** for network-level trust.
>
> For **scalability**, Node is stateless behind a load balancer with Redis for shared state, and Databricks auto-scales compute independently from storage.
>
> **Bottlenecks** like cluster cold starts, blocking HTTP calls, and polling storms are mitigated with warm pools, async patterns, and Redis caching with exponential backoff."

---

## 13. POLLING vs WEBSOCKET vs SSE vs WEB WORKER — Complete Deep Dive

### The Core Question

```
When a user triggers a Databricks job that takes 5 minutes,
HOW does React know when it's done?

4 approaches:
  1. Polling         → React keeps asking "done yet?" every few seconds
  2. WebSocket       → Node pushes "it's done!" to React instantly
  3. SSE             → Node streams updates to React (one-way push)
  4. Web Worker      → React offloads the polling to a background thread
```

---

### 13.1 WHAT IS EACH? (Easy Analogies)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    REAL-WORLD ANALOGIES                               │
│                                                                      │
│  POLLING = Calling a restaurant every 5 min:                        │
│            "Is my order ready?" "No." ... "Ready?" "No." ... "Yes!" │
│            You waste effort asking, but it's simple.                 │
│                                                                      │
│  WEBSOCKET = Restaurant gives you a walkie-talkie:                  │
│              They call YOU when food is ready.                       │
│              Two-way: you can also talk back anytime.                │
│              Always connected (walkie-talkie stays on).              │
│                                                                      │
│  SSE = Restaurant has a speaker system (PA):                        │
│        "Order 42 ready!" ... "Order 43 ready!"                      │
│        One-way: restaurant → you (you can't talk back).             │
│        You just listen.                                              │
│                                                                      │
│  WEB WORKER = You send your friend to wait at the restaurant:       │
│               "Hey friend, keep calling them. Tell me when ready."  │
│               YOU continue shopping (main thread stays free).        │
│               Friend does the tedious work in background.            │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 13.2 COMPARISON TABLE

```
┌────────────────┬──────────────┬──────────────┬──────────────┬───────────────┐
│                │  POLLING     │  WEBSOCKET   │  SSE         │  WEB WORKER   │
├────────────────┼──────────────┼──────────────┼──────────────┼───────────────┤
│ Direction      │ Client → Srv │ Client ↔ Srv │ Server → Cli │ Main ↔ Worker │
│                │ (pull)       │ (bi-direct.) │ (push only)  │ (in browser)  │
├────────────────┼──────────────┼──────────────┼──────────────┼───────────────┤
│ Connection     │ New HTTP     │ Persistent   │ Persistent   │ No network    │
│ type           │ each time    │ TCP socket   │ HTTP stream  │ (thread only) │
├────────────────┼──────────────┼──────────────┼──────────────┼───────────────┤
│ Protocol       │ HTTP/HTTPS   │ ws:// wss:// │ HTTP/HTTPS   │ Browser API   │
├────────────────┼──────────────┼──────────────┼──────────────┼───────────────┤
│ Latency        │ Up to poll   │ Near instant │ Near instant │ Same as what  │
│                │ interval     │ (~50ms)      │ (~50ms)      │ it wraps      │
├────────────────┼──────────────┼──────────────┼──────────────┼───────────────┤
│ Server load    │ HIGH         │ MEDIUM       │ LOW          │ No server     │
│                │ (constant    │ (persistent  │ (persistent  │ impact        │
│                │  requests)   │  connections)│  connections)│               │
├────────────────┼──────────────┼──────────────┼──────────────┼───────────────┤
│ Complexity     │ ⭐ Easy      │ ⭐⭐⭐ Hard │ ⭐⭐ Medium  │ ⭐⭐ Medium   │
├────────────────┼──────────────┼──────────────┼──────────────┼───────────────┤
│ Auto-reconnect │ Built-in     │ Manual       │ Built-in     │ N/A           │
│                │ (new request)│ (code it)    │ (EventSource)│               │
├────────────────┼──────────────┼──────────────┼──────────────┼───────────────┤
│ Scale to 10K   │ Hard (10K    │ Hard (10K    │ Easier (less │ No server     │
│ users          │ req/sec!)    │ connections) │ overhead per │ concern       │
│                │              │              │ connection)  │               │
├────────────────┼──────────────┼──────────────┼──────────────┼───────────────┤
│ Works through  │ ✅ Yes       │ ❌ Sometimes │ ✅ Yes       │ ✅ Yes        │
│ proxies/LB     │              │ needs config │              │               │
├────────────────┼──────────────┼──────────────┼──────────────┼───────────────┤
│ Browser        │ ✅ All       │ ✅ All modern│ ✅ All modern│ ✅ All modern │
│ support        │              │              │ (not IE)     │               │
├────────────────┼──────────────┼──────────────┼──────────────┼───────────────┤
│ UI blocking?   │ No (async)   │ No (async)   │ No (async)   │ ✅ Guaranteed │
│                │ but callbacks│ but callbacks│ but callbacks│ non-blocking  │
│                │ run on main  │ run on main  │ run on main  │ (diff thread) │
└────────────────┴──────────────┴──────────────┴──────────────┴───────────────┘
```

---

### 13.3 VISUAL DIAGRAM: How Each Works

```
═══════════════════════════════════════════════════════════════════
APPROACH 1: POLLING
═══════════════════════════════════════════════════════════════════

TIME ───────────────────────────────────────────────────────────►

React                          Node.js                  Databricks
  │                              │                          │
  │──GET /status/123──►          │                          │
  │◄── { status: "PROCESSING" } │                          │
  │                              │                          │
  │   (wait 3 seconds)          │                       ⚙️ working
  │                              │                          │
  │──GET /status/123──►          │                          │
  │◄── { status: "PROCESSING" } │                          │
  │                              │                          │
  │   (wait 3 seconds)          │                       ⚙️ working
  │                              │                          │
  │──GET /status/123──►          │──check status──►         │
  │◄── { status: "DONE" }       │◄── TERMINATED ──         │ ✅ done
  │                              │                          │

  Total HTTP requests: ~20 (for a 1-minute job, polling every 3s)
  Wasted requests: ~19 (only the last one was useful)

═══════════════════════════════════════════════════════════════════
APPROACH 2: WEBSOCKET
═══════════════════════════════════════════════════════════════════

React                          Node.js                  Databricks
  │                              │                          │
  │══ WebSocket CONNECT ═══════► │                          │
  │   (persistent connection)    │                          │
  │                              │                          │
  │──{ action: "start", ... }──► │──trigger job──►          │
  │◄──{ jobId: 123 }────────    │                          │
  │                              │                       ⚙️ working
  │   (connection stays open)    │                          │
  │   (React does other stuff)   │                          │
  │                              │                          │
  │                              │     (Node polls          │
  │                              │      Databricks          │
  │                              │      internally)         │
  │                              │                       ✅ done
  │                              │◄── TERMINATED ──         │
  │◄──{ status: "DONE",  }─────│                          │
  │    { result: {...}  }        │                          │
  │                              │                          │
  │══ WebSocket still open ═════►│  (reuse for more)       │

  Total HTTP requests: 0 (after initial WS handshake)
  React gets result INSTANTLY when done (no delay)

═══════════════════════════════════════════════════════════════════
APPROACH 3: SSE (Server-Sent Events)
═══════════════════════════════════════════════════════════════════

React                          Node.js                  Databricks
  │                              │                          │
  │──GET /events/123 ──────────► │                          │
  │   (HTTP stream stays open)   │                          │
  │                              │                       ⚙️ working
  │◄── event: progress           │                          │
  │    data: { percent: 25 }     │                          │
  │                              │                       ⚙️ working
  │◄── event: progress           │                          │
  │    data: { percent: 75 }     │                          │
  │                              │                       ✅ done
  │◄── event: complete           │◄── TERMINATED ──         │
  │    data: { result: {...} }   │                          │
  │                              │                          │
  │   (connection closes)        │                          │

  Total HTTP requests: 1 (stays open as a stream)
  Server pushes updates as they happen
  React CAN'T send data back through this connection

═══════════════════════════════════════════════════════════════════
APPROACH 4: WEB WORKER (+ any of the above)
═══════════════════════════════════════════════════════════════════

Main Thread (UI)              Web Worker (Background)    Node.js
  │                              │                          │
  │──"start polling job 123"──► │                          │
  │                              │──GET /status/123──►      │
  │  (UI stays 100% responsive) │◄── PROCESSING ──         │
  │  (user scrolls, clicks,     │                          │
  │   types — zero lag)          │──GET /status/123──►      │
  │                              │◄── PROCESSING ──         │
  │                              │                          │
  │                              │──GET /status/123──►      │
  │                              │◄── DONE ──               │
  │◄──"job done! result={..}"── │                          │
  │                              │                          │
  │  (Update UI with result)     │                          │

  Web Worker doesn't replace polling/WS/SSE.
  It WRAPS them so the main thread stays free.
```

---

### 13.4 IMPLEMENTATION: POLLING (All 3 Layers)

#### React (Polling with Exponential Backoff)

```tsx
// src/hooks/useJobPolling.ts
import { useState, useEffect, useRef, useCallback } from "react";
import api from "../store/axiosInstance";

type JobStatus = "idle" | "submitting" | "processing" | "completed" | "failed";

interface UseJobPollingOptions {
  initialInterval?: number; // Starting poll interval (ms)
  maxInterval?: number;     // Max poll interval (ms)
  backoffFactor?: number;   // Multiply interval each time
}

export function useJobPolling(options: UseJobPollingOptions = {}) {
  const {
    initialInterval = 2000,  // Start: poll every 2s
    maxInterval = 30000,     // Max: poll every 30s
    backoffFactor = 1.5,     // Each poll waits 1.5x longer
  } = options;

  const [jobId, setJobId] = useState<string | null>(null);
  const [status, setStatus] = useState<JobStatus>("idle");
  const [result, setResult] = useState<any>(null);
  const [error, setError] = useState<string | null>(null);
  const timerRef = useRef<ReturnType<typeof setTimeout>>();
  const intervalRef = useRef(initialInterval);

  // Cleanup on unmount (prevents memory leaks!)
  useEffect(() => {
    return () => {
      if (timerRef.current) clearTimeout(timerRef.current);
    };
  }, []);

  const pollStatus = useCallback(async (id: string) => {
    try {
      const res = await api.get(`/api/analytics/status/${id}`);

      if (res.data.status === "COMPLETED") {
        setStatus("completed");
        setResult(res.data.result);
        intervalRef.current = initialInterval; // Reset for next job
        return; // Stop polling
      }

      if (res.data.status === "FAILED") {
        setStatus("failed");
        setError(res.data.error || "Job failed");
        intervalRef.current = initialInterval;
        return; // Stop polling
      }

      // Still processing → poll again with backoff
      //
      // WHY EXPONENTIAL BACKOFF?
      // Poll 1: wait 2s    (job just started, check quick)
      // Poll 2: wait 3s    (probably still processing)
      // Poll 3: wait 4.5s  (getting patient)
      // Poll 4: wait 6.7s  (definitely still going)
      // ...
      // Poll N: wait 30s   (max — don't wait longer than this)
      //
      // This reduces server load by 60-70% vs fixed interval!
      intervalRef.current = Math.min(
        intervalRef.current * backoffFactor,
        maxInterval
      );

      timerRef.current = setTimeout(() => pollStatus(id), intervalRef.current);
    } catch (err) {
      setStatus("failed");
      setError("Network error while checking status");
    }
  }, [initialInterval, maxInterval, backoffFactor]);

  const triggerJob = useCallback(async (params: any) => {
    setStatus("submitting");
    setError(null);
    setResult(null);
    intervalRef.current = initialInterval;

    try {
      const res = await api.post("/api/analytics/report", params);
      setJobId(res.data.jobId);
      setStatus("processing");

      // Start polling after a short delay (give Databricks time to start)
      timerRef.current = setTimeout(
        () => pollStatus(res.data.jobId),
        initialInterval
      );
    } catch (err: any) {
      setStatus("failed");
      setError(err.response?.data?.error || "Failed to trigger job");
    }
  }, [pollStatus, initialInterval]);

  // Allow manual cancellation
  const cancelPolling = useCallback(() => {
    if (timerRef.current) clearTimeout(timerRef.current);
    setStatus("idle");
  }, []);

  return { triggerJob, cancelPolling, jobId, status, result, error };
}
```

```tsx
// Usage in component:
function ReportPage() {
  const { triggerJob, status, result, error } = useJobPolling({
    initialInterval: 2000,
    maxInterval: 15000,
    backoffFactor: 1.5,
  });

  return (
    <div>
      <button
        onClick={() => triggerJob({ dateRange: "2024-01-01/2024-12-31", metric: "revenue" })}
        disabled={status === "processing"}
      >
        Generate Report
      </button>

      {status === "processing" && <p>⏳ Processing...</p>}
      {status === "completed" && <pre>{JSON.stringify(result, null, 2)}</pre>}
      {status === "failed" && <p>❌ {error}</p>}
    </div>
  );
}
```

#### Node.js (Polling Status Endpoint with Redis Cache)

```javascript
// src/controllers/analyticsController.js
// This endpoint is called repeatedly by React's polling

async getJobStatus(req, res, next) {
  try {
    const { jobId } = req.params;

    // 1. Check Redis FIRST (avoids hitting Databricks every poll)
    const cached = await redisDao.get(`job:${jobId}`);
    if (cached) {
      const parsed = JSON.parse(cached);
      if (parsed.status === "COMPLETED" || parsed.status === "FAILED") {
        return res.json(parsed); // Return cached final status
      }
    }

    // 2. Only call Databricks if status is still "PROCESSING"
    const dbStatus = await databricksDao.getJobStatus(jobId);

    if (dbStatus.lifeCycleState === "TERMINATED") {
      if (dbStatus.resultState === "SUCCESS") {
        const output = await databricksDao.getJobOutput(jobId);
        const result = { status: "COMPLETED", result: output };
        await redisDao.setWithExpiry(`job:${jobId}`, 3600, result);
        return res.json(result);
      }
      const failed = { status: "FAILED", error: "Databricks job failed" };
      await redisDao.setWithExpiry(`job:${jobId}`, 3600, failed);
      return res.json(failed);
    }

    // 3. Still running — cache this briefly to reduce Databricks API calls
    //    Next poll within 5 seconds gets this cached status
    await redisDao.setWithExpiry(`job:${jobId}`, 5, { status: "PROCESSING" });
    res.json({ status: "PROCESSING" });

  } catch (error) {
    next(error);
  }
}
```

#### Databricks (No Change Needed)

```
Databricks doesn't know or care about polling.
It just processes the job. Node.js checks status via REST API.

POST /api/2.1/jobs/run-now       → Trigger
GET  /api/2.1/jobs/runs/get      → Check status (Node calls this)
GET  /api/2.1/jobs/runs/get-output → Get results
```

#### Impact of Polling

```
✅ PROS:
  - Simplest to implement
  - Works everywhere (all browsers, all proxies, all load balancers)
  - Stateless — server doesn't track connections
  - Easy to debug (regular HTTP requests)

❌ CONS:
  - Wastes bandwidth (95% of polls return "still processing")
  - Adds latency (up to full poll interval delay)
  - High server load at scale (1000 users × polling every 3s = 333 req/s!)
  - Databricks API rate limits can be hit

COST IMPACT:
  100 users × 3s polling × 5-min job = 10,000 wasted HTTP requests
  With exponential backoff: ~3,000 requests (70% reduction!)
```

---

### 13.5 IMPLEMENTATION: WEBSOCKET (All 3 Layers)

#### What is WebSocket? (Deeper)

```
Normal HTTP:
  React: "Open door → ask question → get answer → close door"
  (New connection every time)

WebSocket:
  React: "Open door → keep it open → talk whenever → listen whenever"
  (ONE connection, stays alive, two-way communication)

Technical flow:
  1. React sends HTTP request with "Upgrade: websocket" header
  2. Node responds "101 Switching Protocols"
  3. Connection upgrades from HTTP → WebSocket (TCP stays open)
  4. Both sides can send messages anytime
```

#### React (WebSocket with Reconnection)

```tsx
// src/hooks/useJobWebSocket.ts
import { useState, useEffect, useRef, useCallback } from "react";

type JobStatus = "idle" | "connecting" | "processing" | "completed" | "failed";

export function useJobWebSocket() {
  const [status, setStatus] = useState<JobStatus>("idle");
  const [result, setResult] = useState<any>(null);
  const [error, setError] = useState<string | null>(null);
  const [jobId, setJobId] = useState<string | null>(null);
  const wsRef = useRef<WebSocket | null>(null);
  const reconnectAttempts = useRef(0);
  const maxReconnectAttempts = 5;

  // Connect to WebSocket
  const connect = useCallback(() => {

    // Get JWT token for authentication
    const token = localStorage.getItem("authToken");

    // wss:// = WebSocket Secure (like https://)
    // Pass token as query param (WebSocket doesn't support headers easily)
    const ws = new WebSocket(
      `wss://api.yourapp.com/ws/jobs?token=${token}`
    );

    ws.onopen = () => {
      console.log("WebSocket connected");
      reconnectAttempts.current = 0; // Reset on successful connection
    };

    // THIS is the magic — server pushes updates to us!
    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);

      switch (message.type) {
        case "JOB_ACCEPTED":
          setJobId(message.jobId);
          setStatus("processing");
          break;

        case "JOB_PROGRESS":
          // Optional: show progress percentage
          console.log(`Job ${message.jobId}: ${message.percent}%`);
          break;

        case "JOB_COMPLETED":
          setStatus("completed");
          setResult(message.result);
          break;

        case "JOB_FAILED":
          setStatus("failed");
          setError(message.error);
          break;
      }
    };

    ws.onclose = (event) => {
      console.log("WebSocket closed:", event.code);

      // Auto-reconnect with exponential backoff
      if (reconnectAttempts.current < maxReconnectAttempts) {
        const delay = Math.pow(2, reconnectAttempts.current) * 1000;
        // 1s → 2s → 4s → 8s → 16s
        console.log(`Reconnecting in ${delay}ms...`);
        setTimeout(() => {
          reconnectAttempts.current++;
          connect();
        }, delay);
      } else {
        setError("Connection lost. Please refresh the page.");
      }
    };

    ws.onerror = (err) => {
      console.error("WebSocket error:", err);
    };

    wsRef.current = ws;
  }, []);

  // Trigger a job through the EXISTING WebSocket connection
  const triggerJob = useCallback((params: any) => {
    if (!wsRef.current || wsRef.current.readyState !== WebSocket.OPEN) {
      setError("Not connected to server");
      return;
    }

    setStatus("processing");
    setError(null);
    setResult(null);

    // Send job request through WebSocket (not HTTP!)
    wsRef.current.send(JSON.stringify({
      action: "TRIGGER_JOB",
      params,
    }));
  }, []);

  // Connect on mount, disconnect on unmount
  useEffect(() => {
    connect();
    return () => {
      wsRef.current?.close(1000, "Component unmounted");
    };
  }, [connect]);

  return { triggerJob, jobId, status, result, error };
}
```

```tsx
// Usage in component (almost identical to polling version!)
function ReportPage() {
  const { triggerJob, status, result, error } = useJobWebSocket();

  return (
    <div>
      <button
        onClick={() => triggerJob({ dateRange: "2024-01-01/2024-12-31", metric: "revenue" })}
        disabled={status === "processing"}
      >
        Generate Report
      </button>

      {status === "processing" && <p>⏳ Processing...</p>}
      {status === "completed" && <pre>{JSON.stringify(result, null, 2)}</pre>}
      {status === "failed" && <p>❌ {error}</p>}
    </div>
  );
}
```

#### Node.js (WebSocket Server with Auth)

```javascript
// src/websocket/wsServer.js
const { WebSocketServer } = require("ws");
const jwt = require("jsonwebtoken");
const url = require("url");
const analyticsService = require("../services/analyticsService");
const databricksDao = require("../dao/databricksDao");
const redisDao = require("../dao/redisDao");

function setupWebSocket(httpServer) {

  const wss = new WebSocketServer({ server: httpServer });

  // Track connected users: userId → WebSocket
  const clients = new Map();

  wss.on("connection", (ws, req) => {

    // ── STEP 1: AUTHENTICATE ─────────────────────────────
    // Extract token from query string: /ws/jobs?token=ey...
    const params = url.parse(req.url, true).query;
    let user;

    try {
      user = jwt.verify(params.token, process.env.JWT_SECRET);
    } catch (err) {
      ws.close(4001, "Invalid token");
      return;
    }

    // Track this connection
    clients.set(user.userId, ws);
    console.log(`WS: User ${user.userId} connected`);

    // ── STEP 2: HANDLE INCOMING MESSAGES ─────────────────
    ws.on("message", async (data) => {
      try {
        const message = JSON.parse(data);

        if (message.action === "TRIGGER_JOB") {
          // Trigger job through service layer (same as REST!)
          const result = await analyticsService.triggerReport({
            dateRange: message.params.dateRange,
            metric: message.params.metric,
            userId: user.userId,
          });

          // Confirm job accepted
          ws.send(JSON.stringify({
            type: "JOB_ACCEPTED",
            jobId: result.jobId,
          }));

          // Start internal polling to Databricks
          // (Node polls Databricks, pushes updates to React via WS)
          pollAndNotify(result.jobId, user.userId);
        }
      } catch (error) {
        ws.send(JSON.stringify({
          type: "ERROR",
          error: error.message,
        }));
      }
    });

    // ── STEP 3: CLEANUP ON DISCONNECT ────────────────────
    ws.on("close", () => {
      clients.delete(user.userId);
      console.log(`WS: User ${user.userId} disconnected`);
    });

    // Heartbeat to keep connection alive (prevent timeout)
    ws.isAlive = true;
    ws.on("pong", () => { ws.isAlive = true; });
  });

  // ── HEARTBEAT: Kill dead connections every 30s ───────────
  const heartbeat = setInterval(() => {
    wss.clients.forEach((ws) => {
      if (!ws.isAlive) return ws.terminate();
      ws.isAlive = false;
      ws.ping();
    });
  }, 30000);

  wss.on("close", () => clearInterval(heartbeat));

  // ── INTERNAL POLLING → PUSH TO CLIENT ────────────────────
  // Node polls Databricks (server-side), then pushes to React via WS
  async function pollAndNotify(jobId, userId) {
    const check = async () => {
      try {
        const status = await databricksDao.getJobStatus(jobId);

        if (status.lifeCycleState === "TERMINATED") {
          const client = clients.get(userId);
          if (!client) return; // User disconnected

          if (status.resultState === "SUCCESS") {
            const output = await databricksDao.getJobOutput(jobId);
            client.send(JSON.stringify({
              type: "JOB_COMPLETED",
              jobId,
              result: output,
            }));
          } else {
            client.send(JSON.stringify({
              type: "JOB_FAILED",
              jobId,
              error: "Databricks job failed",
            }));
          }
          return; // Stop polling
        }

        // Still running → check again in 5 seconds
        setTimeout(check, 5000);
      } catch (err) {
        console.error(`Poll error for job ${jobId}:`, err);
        setTimeout(check, 10000); // Retry slower on error
      }
    };

    // Start checking after 3 seconds
    setTimeout(check, 3000);
  }

  // ── PUBLIC METHOD: Push to any user from anywhere ────────
  function notifyUser(userId, message) {
    const client = clients.get(userId);
    if (client && client.readyState === 1) {
      client.send(JSON.stringify(message));
    }
  }

  return { wss, notifyUser };
}

module.exports = { setupWebSocket };
```

```javascript
// src/server.js — Integrate WebSocket with Express
const http = require("http");
const app = require("./app");
const { setupWebSocket } = require("./websocket/wsServer");

const server = http.createServer(app);
const { notifyUser } = setupWebSocket(server);

// Export notifyUser so other services can push to users
module.exports.notifyUser = notifyUser;

server.listen(3001, () => {
  console.log("Server + WebSocket running on port 3001");
});
```

#### Databricks (Same as Polling — No Change)

```
Databricks Still Doesn't Know About WebSockets!

The pattern is:
  React ←──WS──── Node.js ────REST API────► Databricks

  Databricks only understands REST.
  Node.js translates between WebSocket (frontend) and REST (Databricks).
  Node does the polling to Databricks, then pushes via WebSocket to React.
```

#### Impact of WebSocket

```
✅ PROS:
  - Real-time updates (no polling delay)
  - Lower network traffic (no repeated HTTP requests from React)
  - Bi-directional (React can send AND receive)
  - Better UX (instant feedback)

❌ CONS:
  - Complex to implement (connection management, heartbeats, reconnection)
  - Stateful — server must track each connection (harder to scale)
  - Load balancer config needed (sticky sessions or Redis pub/sub)
  - Memory usage: each connection holds a TCP socket
  - Harder to debug (not regular HTTP in DevTools)
  - Node STILL polls Databricks (WS only helps React ↔ Node)

SCALING CONCERN:
  10,000 users = 10,000 persistent connections
  Each connection ≈ 2-5KB RAM
  Node.js can handle ~50K connections per instance
  BUT: need sticky sessions or Redis pub/sub for multi-node
```

---

### 13.6 IMPLEMENTATION: SSE (All 3 Layers)

#### What is SSE? (Server-Sent Events)

```
SSE = HTTP connection that stays open, server pushes data through it

Like a radio station:
  - You tune in (open connection)
  - The station broadcasts (server sends events)
  - You listen (React receives events)
  - You CAN'T talk back through the radio (one-way)

VS WebSocket:
  WebSocket = Phone call (two-way)
  SSE = Radio (one-way: server → client)

FOR OUR USE CASE: SSE is PERFECT because:
  - React only needs to RECEIVE updates (job status)
  - React doesn't need to SEND data through this connection
  - Trigger job via regular POST, then listen for updates via SSE
```

#### React (SSE with EventSource API)

```tsx
// src/hooks/useJobSSE.ts
import { useState, useEffect, useRef, useCallback } from "react";
import api from "../store/axiosInstance";

type JobStatus = "idle" | "submitting" | "processing" | "completed" | "failed";

export function useJobSSE() {
  const [status, setStatus] = useState<JobStatus>("idle");
  const [result, setResult] = useState<any>(null);
  const [error, setError] = useState<string | null>(null);
  const [progress, setProgress] = useState<number>(0);
  const [jobId, setJobId] = useState<string | null>(null);
  const eventSourceRef = useRef<EventSource | null>(null);

  // Cleanup EventSource on unmount
  useEffect(() => {
    return () => {
      eventSourceRef.current?.close();
    };
  }, []);

  const triggerJob = useCallback(async (params: any) => {
    // Close any existing SSE connection
    eventSourceRef.current?.close();

    setStatus("submitting");
    setError(null);
    setResult(null);
    setProgress(0);

    try {
      // Step 1: Trigger job via regular HTTP POST
      const res = await api.post("/api/analytics/report", params);
      const newJobId = res.data.jobId;
      setJobId(newJobId);
      setStatus("processing");

      // Step 2: Open SSE connection to listen for updates
      // EventSource is a browser built-in API!
      const token = localStorage.getItem("authToken");
      const eventSource = new EventSource(
        `/api/analytics/events/${newJobId}?token=${token}`
      );

      // Listen for different event types
      eventSource.addEventListener("progress", (event) => {
        const data = JSON.parse(event.data);
        setProgress(data.percent);
      });

      eventSource.addEventListener("completed", (event) => {
        const data = JSON.parse(event.data);
        setStatus("completed");
        setResult(data.result);
        eventSource.close(); // Done! Close the connection
      });

      eventSource.addEventListener("failed", (event) => {
        const data = JSON.parse(event.data);
        setStatus("failed");
        setError(data.error);
        eventSource.close();
      });

      // Built-in auto-reconnect! (browser handles this automatically)
      eventSource.onerror = (err) => {
        console.error("SSE error:", err);
        // EventSource auto-reconnects unless we close it
        // After 3 retries with no success, give up:
        if (eventSource.readyState === EventSource.CLOSED) {
          setStatus("failed");
          setError("Lost connection to server");
        }
      };

      eventSourceRef.current = eventSource;

    } catch (err: any) {
      setStatus("failed");
      setError(err.response?.data?.error || "Failed to trigger job");
    }
  }, []);

  return { triggerJob, jobId, status, result, error, progress };
}
```

```tsx
// Usage with progress bar!
function ReportPage() {
  const { triggerJob, status, result, error, progress } = useJobSSE();

  return (
    <div>
      <button
        onClick={() => triggerJob({ dateRange: "2024-01-01/2024-12-31", metric: "revenue" })}
        disabled={status === "processing"}
      >
        Generate Report
      </button>

      {status === "processing" && (
        <div>
          <p>⏳ Processing... {progress}%</p>
          <progress value={progress} max={100} />
        </div>
      )}
      {status === "completed" && <pre>{JSON.stringify(result, null, 2)}</pre>}
      {status === "failed" && <p>❌ {error}</p>}
    </div>
  );
}
```

#### Node.js (SSE Endpoint)

```javascript
// src/controllers/sseController.js
const databricksDao = require("../dao/databricksDao");
const redisDao = require("../dao/redisDao");
const jwt = require("jsonwebtoken");

/**
 * SSE endpoint: GET /api/analytics/events/:jobId
 *
 * HOW IT WORKS:
 * 1. Client opens connection (stays open)
 * 2. Server sends events as text through the connection
 * 3. Each event has a type and data
 * 4. Connection closes when job completes
 *
 * SSE FORMAT (plain text over HTTP):
 * event: progress
 * data: {"percent": 50}
 * 
 * event: completed
 * data: {"result": {...}}
 */
async function streamJobEvents(req, res, next) {
  try {
    const { jobId } = req.params;

    // Verify auth (SSE uses query param since EventSource doesn't support headers)
    const token = req.query.token;
    if (!token) return res.status(401).end();

    try {
      jwt.verify(token, process.env.JWT_SECRET);
    } catch {
      return res.status(401).end();
    }

    // ── SET SSE HEADERS ──────────────────────────────────
    res.writeHead(200, {
      "Content-Type": "text/event-stream",   // THIS makes it SSE!
      "Cache-Control": "no-cache",            // Don't cache the stream
      "Connection": "keep-alive",             // Keep connection open
      "X-Accel-Buffering": "no",             // Prevent NGINX from buffering
    });

    // Send initial event
    sendEvent(res, "connected", { message: "Listening for job updates" });

    // ── POLL DATABRICKS & PUSH EVENTS ────────────────────
    let pollCount = 0;
    const maxPolls = 360; // Max 30 minutes (5s × 360)

    const pollInterval = setInterval(async () => {
      pollCount++;

      try {
        // Check if client disconnected
        if (res.writableEnded) {
          clearInterval(pollInterval);
          return;
        }

        const status = await databricksDao.getJobStatus(jobId);

        if (status.lifeCycleState === "RUNNING") {
          // Send progress update (estimate based on poll count)
          sendEvent(res, "progress", {
            percent: Math.min(Math.round((pollCount / 60) * 100), 95),
            state: "RUNNING",
          });
        }

        if (status.lifeCycleState === "TERMINATED") {
          clearInterval(pollInterval);

          if (status.resultState === "SUCCESS") {
            const output = await databricksDao.getJobOutput(jobId);
            await redisDao.setWithExpiry(`job:${jobId}`, 3600, {
              status: "COMPLETED",
              result: output,
            });
            sendEvent(res, "completed", { result: output });
          } else {
            sendEvent(res, "failed", { error: "Databricks job failed" });
          }

          res.end(); // Close the stream
        }

        if (pollCount >= maxPolls) {
          clearInterval(pollInterval);
          sendEvent(res, "failed", { error: "Job timed out" });
          res.end();
        }

      } catch (err) {
        sendEvent(res, "error", { error: "Failed to check status" });
      }
    }, 5000); // Poll Databricks every 5 seconds

    // ── CLEANUP IF CLIENT DISCONNECTS ───────────────────
    req.on("close", () => {
      clearInterval(pollInterval);
      console.log(`SSE: Client disconnected from job ${jobId}`);
    });

  } catch (error) {
    next(error);
  }
}

// Helper: Send an SSE event
function sendEvent(res, eventType, data) {
  res.write(`event: ${eventType}\n`);
  res.write(`data: ${JSON.stringify(data)}\n\n`);  // Double newline = end of event
}

module.exports = { streamJobEvents };
```

```javascript
// src/routes/analyticsRoutes.js — Add SSE route
const { streamJobEvents } = require("../controllers/sseController");

router.get("/events/:jobId", streamJobEvents);
// Note: SSE auth is handled inside the controller (query param)
```

#### SSE Data Format (What Goes Over the Wire)

```
HTTP Response (stays open, keeps sending):

event: connected
data: {"message":"Listening for job updates"}

event: progress
data: {"percent":10,"state":"RUNNING"}

event: progress
data: {"percent":35,"state":"RUNNING"}

event: progress
data: {"percent":72,"state":"RUNNING"}

event: completed
data: {"result":{"totalRevenue":1500000,"regions":[...]}}

(connection closes)
```

#### Impact of SSE

```
✅ PROS:
  - Simpler than WebSocket (just HTTP with keep-alive)
  - Auto-reconnection built into browser (EventSource API)
  - Works through proxies and load balancers easily (it's just HTTP!)
  - Lower overhead than WebSocket (no upgrade handshake)
  - Can send named events (progress, completed, failed)
  - Perfect for our use case (server → client updates)

❌ CONS:
  - ONE-WAY ONLY: server → client (can't send data back)
  - Max 6 connections per domain in HTTP/1.1 (use HTTP/2!)
  - Text only (no binary data)
  - Node still polls Databricks internally (same as WebSocket)
  - Each SSE connection holds an HTTP connection open on the server

WHY SSE IS OFTEN THE BEST CHOICE FOR DATABRICKS:
  We only need server → client updates (job status).
  The job trigger goes via regular POST (doesn't need the stream).
  SSE is simpler, auto-reconnects, works through proxies.
```

---

### 13.7 IMPLEMENTATION: WEB WORKER (All 3 Layers)

#### What is a Web Worker?

```
PROBLEM:
  JavaScript in the browser runs on a SINGLE THREAD (the "main thread").
  The main thread handles:
    - UI rendering
    - User interactions (clicks, scrolls, typing)
    - API calls
    - State updates

  If polling logic runs on the main thread:
    - Timer callbacks compete with UI rendering
    - Heavy JSON parsing can cause jank (stuttering)
    - If tab is in background, browser throttles timers

WEB WORKER:
  A separate JavaScript thread that runs IN the browser.

  ┌─────────────────────┐     ┌─────────────────────┐
  │   MAIN THREAD        │     │   WEB WORKER         │
  │                      │     │   (separate thread)   │
  │   React rendering    │     │                      │
  │   User interactions  │◄───►│   Polling logic      │
  │   State updates      │ msg │   JSON parsing       │
  │   Smooth 60fps UI    │     │   Data transformation│
  │                      │     │                      │
  └─────────────────────┘     └─────────────────────┘
  
  They communicate via messages (like texting).
  Worker CAN'T access DOM, React state, or window.
  Worker CAN make HTTP requests (fetch/XMLHttpRequest).
```

#### React (Web Worker for Polling)

```typescript
// src/workers/jobPoller.worker.ts
// THIS FILE RUNS IN A SEPARATE THREAD!

// Web Worker has its own scope — no React, no DOM, no window
// But it CAN use fetch()!

let pollTimer: ReturnType<typeof setTimeout> | null = null;
let currentInterval = 2000;
const MAX_INTERVAL = 30000;
const BACKOFF_FACTOR = 1.5;

// Listen for messages from main thread
self.onmessage = async (event: MessageEvent) => {
  const { type, jobId, apiUrl, token } = event.data;

  if (type === "START_POLLING") {
    currentInterval = 2000;
    pollForStatus(jobId, apiUrl, token);
  }

  if (type === "STOP_POLLING") {
    if (pollTimer) clearTimeout(pollTimer);
    pollTimer = null;
  }
};

async function pollForStatus(jobId: string, apiUrl: string, token: string) {
  try {
    const response = await fetch(`${apiUrl}/api/analytics/status/${jobId}`, {
      headers: { Authorization: `Bearer ${token}` },
    });
    const data = await response.json();

    // Send result back to main thread
    self.postMessage({ type: "STATUS_UPDATE", data });

    if (data.status === "COMPLETED" || data.status === "FAILED") {
      return; // Stop polling
    }

    // Continue polling with backoff
    currentInterval = Math.min(currentInterval * BACKOFF_FACTOR, MAX_INTERVAL);
    pollTimer = setTimeout(
      () => pollForStatus(jobId, apiUrl, token),
      currentInterval
    );

  } catch (error) {
    self.postMessage({
      type: "STATUS_UPDATE",
      data: { status: "ERROR", error: "Network error in worker" },
    });
    // Retry slower
    pollTimer = setTimeout(
      () => pollForStatus(jobId, apiUrl, token),
      10000
    );
  }
}
```

```tsx
// src/hooks/useJobWorkerPolling.ts
import { useState, useEffect, useRef, useCallback } from "react";
import api from "../store/axiosInstance";

type JobStatus = "idle" | "submitting" | "processing" | "completed" | "failed";

export function useJobWorkerPolling() {
  const [status, setStatus] = useState<JobStatus>("idle");
  const [result, setResult] = useState<any>(null);
  const [error, setError] = useState<string | null>(null);
  const [jobId, setJobId] = useState<string | null>(null);
  const workerRef = useRef<Worker | null>(null);

  useEffect(() => {
    // Create the Web Worker
    const worker = new Worker(
      new URL("../workers/jobPoller.worker.ts", import.meta.url),
      { type: "module" }
    );

    // Listen for messages FROM the worker
    worker.onmessage = (event: MessageEvent) => {
      const { type, data } = event.data;

      if (type === "STATUS_UPDATE") {
        if (data.status === "COMPLETED") {
          setStatus("completed");
          setResult(data.result);
        } else if (data.status === "FAILED") {
          setStatus("failed");
          setError(data.error);
        }
        // "PROCESSING" — just keep waiting
      }
    };

    workerRef.current = worker;

    // Cleanup: terminate worker on unmount
    return () => {
      worker.postMessage({ type: "STOP_POLLING" });
      worker.terminate();
    };
  }, []);

  const triggerJob = useCallback(async (params: any) => {
    setStatus("submitting");
    setError(null);
    setResult(null);

    try {
      // Step 1: Trigger job via normal HTTP
      const res = await api.post("/api/analytics/report", params);
      setJobId(res.data.jobId);
      setStatus("processing");

      // Step 2: Tell worker to start polling (OFF the main thread!)
      workerRef.current?.postMessage({
        type: "START_POLLING",
        jobId: res.data.jobId,
        apiUrl: import.meta.env.VITE_API_URL,
        token: localStorage.getItem("authToken"),
      });

    } catch (err: any) {
      setStatus("failed");
      setError(err.response?.data?.error || "Failed to trigger job");
    }
  }, []);

  return { triggerJob, jobId, status, result, error };
}
```

#### Vite Config (Enable Web Workers)

```javascript
// vite.config.js — Workers work out of the box with Vite!
// No special config needed. Vite handles worker bundling.
// The `new URL(..., import.meta.url)` pattern is the key.
```

#### Node.js and Databricks (No Change!)

```
Web Workers are purely a FRONTEND concept.
Node.js and Databricks don't know or care.
The HTTP requests from the worker look identical to requests from the main thread.
```

#### Impact of Web Worker

```
✅ PROS:
  - Main thread stays 100% free (UI never stutters)
  - Heavy JSON parsing happens off-thread
  - Browser doesn't throttle worker timers in background tabs
  - Can combine with ANY approach (polling, WebSocket, SSE in worker)

❌ CONS:
  - Can't access DOM, React state, or window
  - Communication is async only (postMessage)
  - Adds complexity (separate file, message passing)
  - Debugging is harder (separate DevTools thread)
  - Slight overhead for small tasks (not worth it for simple polling)

WHEN ITS WORTH IT:
  - Heavy data processing on the frontend
  - JSON parsing of large payloads (>1MB)
  - Multiple jobs being polled simultaneously
  - Need polling to continue in background tabs
  - Dashboard with many real-time widgets
```

---

### 13.8 COMBINED APPROACH (Best Practice)

#### The Real-World Best: SSE + Web Worker

```
For our Databricks use case, the IDEAL combo is:

  ┌─────────────────────────────────────────────────────────┐
  │  WEB WORKER (background thread)                         │
  │  ├── Opens SSE connection to Node                       │
  │  ├── Receives events (progress, completed, failed)      │
  │  ├── Parses JSON off the main thread                    │
  │  └── Posts results to main thread                       │
  └─────────────────────┬───────────────────────────────────┘
                        │ postMessage
                        ▼
  ┌─────────────────────────────────────────────────────────┐
  │  MAIN THREAD (React)                                    │
  │  ├── Triggers job via POST                              │
  │  ├── Receives updates from worker                       │
  │  ├── Updates UI (progress bar, result)                  │
  │  └── Stays smooth at 60fps                              │
  └─────────────────────────────────────────────────────────┘
```

```typescript
// src/workers/sseWorker.worker.ts
// SSE INSIDE a Web Worker — best of both worlds!

self.onmessage = (event: MessageEvent) => {
  const { type, jobId, apiUrl, token } = event.data;

  if (type === "START_LISTENING") {
    // Open SSE connection FROM the worker (not main thread)
    const eventSource = new EventSource(
      `${apiUrl}/api/analytics/events/${jobId}?token=${token}`
    );

    eventSource.addEventListener("progress", (e) => {
      self.postMessage({ type: "PROGRESS", data: JSON.parse(e.data) });
    });

    eventSource.addEventListener("completed", (e) => {
      const parsed = JSON.parse(e.data); // Heavy parse in worker!
      self.postMessage({ type: "COMPLETED", data: parsed });
      eventSource.close();
    });

    eventSource.addEventListener("failed", (e) => {
      self.postMessage({ type: "FAILED", data: JSON.parse(e.data) });
      eventSource.close();
    });

    eventSource.onerror = () => {
      // EventSource auto-reconnects
      self.postMessage({ type: "RECONNECTING" });
    };
  }
};
```

---

### 13.9 DECISION MATRIX — When to Use What

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DECISION FLOWCHART                                    │
│                                                                         │
│  "I need to show Databricks job status to the user"                    │
│         │                                                               │
│         ▼                                                               │
│  How many concurrent users?                                            │
│         │                                                               │
│    ┌────┴────┐                                                         │
│    │         │                                                         │
│  < 100     > 100                                                       │
│    │         │                                                         │
│    ▼         ▼                                                         │
│  POLLING   Need real-time?                                             │
│  (simple,     │                                                        │
│   works      ┌┴──────┐                                                 │
│   fine)      │       │                                                 │
│            Yes      No → POLLING + Redis cache                         │
│              │                                                         │
│              ▼                                                         │
│         Need bi-directional?                                           │
│              │                                                         │
│         ┌────┴────┐                                                    │
│         │         │                                                    │
│        Yes       No                                                    │
│         │         │                                                    │
│         ▼         ▼                                                    │
│     WEBSOCKET    SSE                                                   │
│     (chat, live  (job status,                                          │
│      collab)     notifications)                                        │
│                                                                         │
│  THEN: Add WEB WORKER if:                                              │
│    - Large JSON payloads (>1MB)                                        │
│    - Multiple simultaneous jobs                                        │
│    - Need background tab support                                       │
│    - Dashboard with many real-time widgets                             │
└─────────────────────────────────────────────────────────────────────────┘
```

### Use Case Table

```
┌────────────────────────────┬─────────────┬────────────────────────────────┐
│  USE CASE                  │  BEST PICK  │  WHY                           │
├────────────────────────────┼─────────────┼────────────────────────────────┤
│  Single user checks job    │  POLLING    │  Simplest, good enough         │
│  status occasionally       │             │                                │
├────────────────────────────┼─────────────┼────────────────────────────────┤
│  Dashboard shows job       │  SSE        │  Server pushes updates,        │
│  progress to many users    │             │  auto-reconnect, works         │
│                            │             │  through proxies               │
├────────────────────────────┼─────────────┼────────────────────────────────┤
│  Live collaborative        │  WEBSOCKET  │  Need bi-directional:          │
│  data editing              │             │  users send + receive          │
├────────────────────────────┼─────────────┼────────────────────────────────┤
│  Real-time chat +          │  WEBSOCKET  │  Chat needs two-way,           │
│  notifications             │             │  instant delivery              │
├────────────────────────────┼─────────────┼────────────────────────────────┤
│  Dashboard with 20 live    │  SSE +      │  SSE for updates,              │
│  widgets showing metrics   │  WEB WORKER │  Worker prevents UI jank       │
├────────────────────────────┼─────────────┼────────────────────────────────┤
│  User triggers 5 Databricks│  POLLING +  │  Worker handles 5 polls        │
│  jobs simultaneously       │  WEB WORKER │  without blocking UI           │
├────────────────────────────┼─────────────┼────────────────────────────────┤
│  Simple team: MVP,         │  POLLING    │  Ship fast, optimize later     │
│  prototype, hackathon      │             │                                │
├────────────────────────────┼─────────────┼────────────────────────────────┤
│  Enterprise: 1000+         │  SSE +      │  SSE scales well,              │
│  concurrent users          │  WEB WORKER │  Worker keeps UI smooth        │
│  tracking jobs             │  + Redis    │  Redis prevents Databricks     │
│                            │             │  API rate limits               │
└────────────────────────────┴─────────────┴────────────────────────────────┘
```

---

### 13.10 PERFORMANCE / COST IMPACT COMPARISON

```
SCENARIO: 1000 users, each tracking 1 Databricks job (avg 5 min each)

┌──────────────┬──────────────────┬──────────────────┬──────────────────┐
│              │  POLLING (3s)    │  POLLING (backoff)│  SSE             │
├──────────────┼──────────────────┼──────────────────┼──────────────────┤
│ HTTP requests│ 100,000          │ ~30,000          │ 1,000            │
│ (total)      │ (1000×100 polls) │ (70% reduction)  │ (1 per user)    │
├──────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Databricks   │ 100,000          │ ~30,000          │ ~12,000          │
│ API calls    │ (if no Redis)    │ (with backoff)   │ (Node polls      │
│              │                  │                  │  internally,     │
│              │ ~5,000           │ ~5,000           │  shared across   │
│              │ (with Redis      │ (with Redis      │  users for same  │
│              │  5s TTL)         │  5s TTL)         │  job)            │
├──────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Bandwidth    │ ~50 MB           │ ~15 MB           │ ~2 MB            │
│              │ (headers ×100K)  │                  │ (events only)    │
├──────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Server       │ HIGH             │ MEDIUM           │ LOW-MEDIUM       │
│ CPU load     │ (parse 100K req) │                  │ (hold connections│
│              │                  │                  │  but less parsing)│
├──────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Server       │ LOW              │ LOW              │ MEDIUM           │
│ Memory       │ (stateless)      │ (stateless)      │ (1K connections  │
│              │                  │                  │  in memory)      │
├──────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Max latency  │ 3 seconds        │ Up to 30 seconds │ Near-instant     │
│ (to show     │ (= polling       │ (max backoff     │ (~50ms from when │
│  result)     │  interval)       │  interval)       │  Node knows)     │
├──────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Complexity   │ ⭐               │ ⭐⭐             │ ⭐⭐             │
│ to implement │                  │                  │                  │
└──────────────┴──────────────────┴──────────────────┴──────────────────┘

                              ┌──────────────────┐
                              │  WEBSOCKET       │
                              ├──────────────────┤
                              │ 0 HTTP requests  │
                              │ (after handshake)│
                              │                  │
                              │ ~12K Databricks  │
                              │ API calls (Node  │
                              │ polls internally)│
                              │                  │
                              │ ~1 MB bandwidth  │
                              │                  │
                              │ MEDIUM CPU       │
                              │                  │
                              │ HIGH Memory      │
                              │ (1K persistent   │
                              │  TCP sockets)    │
                              │                  │
                              │ Near-instant     │
                              │                  │
                              │ ⭐⭐⭐ Complex  │
                              └──────────────────┘
```

---

### 13.11 INTERVIEW ANSWER: Polling vs WebSocket vs SSE vs Web Worker

> "For a Databricks job tracking scenario, I'd evaluate four approaches:
>
> **Polling** is simplest — React periodically calls a status endpoint. With exponential backoff and Redis caching on Node, it works for small to medium scale. The downside is wasted requests and delayed notification.
>
> **SSE** (Server-Sent Events) is my preferred approach for this use case. React opens one persistent HTTP connection, and Node pushes status updates in real-time. It auto-reconnects, works through proxies, and since job tracking is one-way (server → client), we don't need bi-directional communication.
>
> **WebSocket** adds bi-directional communication which is overkill for status tracking but useful if the app also has chat or live collaboration. It's more complex — needs heartbeats, reconnection logic, and sticky sessions for scaling.
>
> **Web Workers** don't replace the above — they complement them. A Worker moves polling or SSE listening off the main thread, keeping the UI smooth at 60fps. Essential when handling large payloads or tracking multiple jobs simultaneously.
>
> In production, I'd use **SSE + Web Worker** with Redis caching on Node. This gives real-time updates, smooth UI, and scalable server-side architecture. Databricks only understands REST, so Node always bridges between the frontend communication method and Databricks REST API."
