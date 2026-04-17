# Q5 — Security Across Frontend, Node.js, and Databricks (10/10 Preparation Guide)

## The Interview Question

> **Imagine you're tasked with implementing secure API communication between the frontend, Node.js backend, and Databricks. How would you approach authentication and authorization across these layers? What security considerations would you keep in mind, especially when handling sensitive data?**

---

## Assessment Feedback (6/10) — Why You Lost Points

Your original answer:
- Touched on OAuth, cookies, XSS, CORS but **lacked implementation details**
- **Missing structured multi-layer approach** (answer was a random list of security terms)
- **No Databricks-specific security** (service accounts, OAuth M2M)
- **No secrets management** (Key Vault, etc.)
- **No network security** (VPC, private endpoints)

To score **10/10**, you must:
1. **Structure by layer**: React → Node → Databricks (not a random list)
2. Clearly separate **authentication vs authorization**
3. Show **service-to-service auth** (Node → Databricks uses service account, NOT user tokens)
4. Mention **secrets management**, **network security**, **audit logging**
5. Provide **code examples** for JWT middleware, RBAC, input validation, Helmet/CORS

---

## 10/10 Interview Answer (Memorize This)

> I would secure this system **in layers**, with distinct trust boundaries at each transition.
>
> **Authentication vs Authorization first**: Authentication proves identity ("who are you?"); Authorization determines permissions ("what can you do?"). Every layer handles both.
>
> **Layer 1 — React to Node.js**: Users authenticate via OAuth 2.0 with a trusted identity provider like Azure AD or Auth0. After login, I prefer **HttpOnly secure cookies** for session management in browser apps because JavaScript cannot read them, which eliminates token theft via XSS. If JWT is required (e.g., for mobile clients), I keep tokens short-lived and store them in memory, never `localStorage`. All communication is over HTTPS. React also uses Content Security Policy headers to prevent XSS injection.
>
> **Layer 2 — Node.js Backend**: Every incoming request passes through a **middleware pipeline**: first JWT/session verification, then RBAC authorization (viewer can read, analyst can trigger jobs, admin can manage users), then **Zod input validation** against a schema, then **Helmet** for security headers, **CORS** restricted to known origins, and **rate limiting** to prevent abuse. I use structured logging with `requestId` correlation for audit trails.
>
> **Layer 3 — Node.js to Databricks**: This is where many candidates fail. I **never** pass the end-user's browser token to Databricks. Instead, Node.js authenticates to Databricks using a **service principal with OAuth machine-to-machine (M2M) flow** — short-lived tokens, automatic rotation, clear service identity. The service account follows **least privilege**: it can run approved jobs and read specific outputs, but cannot manage users or delete tables. Credentials are stored in **Azure Key Vault** or equivalent, never in source code or `.env` files committed to Git.
>
> **Data protection**: Sensitive data is encrypted **in transit** (HTTPS/TLS everywhere) and **at rest** (encrypted storage, encrypted Delta Lake). I never log passwords, full tokens, or PII — only masked values with `requestId` for traceability.
>
> **Network security**: If possible, Databricks is behind **private endpoints / VPC peering** with IP allowlisting, so even if a token is stolen, network restrictions reduce blast radius.
>
> **Audit and observability**: Structured logs with correlation IDs tracing user → Node → Databricks. Alerts on suspicious patterns: unusual job volume, failed auth attempts, privilege escalation. In Databricks, I enable **Unity Catalog** for table-level access control and audit logging.

---

## Why This Is a 10/10 Answer

| Criteria | Covered? |
|----------|----------|
| Authentication vs authorization | ✅ Clearly separated |
| Layer-by-layer structure | ✅ React → Node → Databricks |
| Frontend security (cookies, CSP, XSS) | ✅ |
| Backend security (JWT, RBAC, Helmet, CORS, rate limit, validation) | ✅ |
| Service-to-service auth (OAuth M2M, service principal) | ✅ |
| Secrets management (Key Vault) | ✅ |
| Data protection (transit + rest) | ✅ |
| Network security (VPC, private endpoints) | ✅ |
| Audit and observability | ✅ |
| Databricks-specific (Unity Catalog) | ✅ |

---

## Authentication vs Authorization — Say This First

| | Authentication | Authorization |
|-|----------------|---------------|
| **Question** | Who are you? | What are you allowed to do? |
| **Examples** | OAuth login, JWT verification, session cookie | RBAC roles, resource permissions, API scopes |
| **Fails** | 401 Unauthorized (wrong identity) | 403 Forbidden (not permitted) |

**Interview line**: "Authentication answers who the caller is; authorization answers what they can do. Every layer in my architecture handles both."

---

## Layer-by-Layer Security Model

```
┌──────────────────────┐
│     React Browser     │  Authentication: OAuth login → token/cookie
│                       │  Security: HTTPS, CSP, HttpOnly cookies, no secrets in JS
└──────────┬───────────┘
           │ HTTPS + HttpOnly cookie / JWT
           ▼
┌──────────────────────┐
│    Node.js Backend    │  Authentication: Verify JWT/session every request
│                       │  Authorization: RBAC middleware
│                       │  Security: Helmet, CORS, rate limit, Zod validation
└──────────┬───────────┘
           │ OAuth M2M token (service principal)
           ▼
┌──────────────────────┐
│      Databricks       │  Authentication: Service principal only
│                       │  Authorization: Unity Catalog, workspace permissions
│                       │  Security: Private endpoints, audit logs
└──────────────────────┘
```

---

## Layer 1: React / Browser Security

### Authentication Strategy

Users authenticate via OAuth 2.0 (Azure AD, Auth0, Okta):

```
User clicks "Login"
  → Redirect to Identity Provider (IdP)
  → User enters credentials at IdP (passwords never touch your app)
  → IdP returns authorization code
  → Node.js exchanges code for tokens (server-side, not browser)
  → Node sets HttpOnly secure cookie with session
```

### Cookie vs JWT — Know Which to Recommend

| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| **HttpOnly Cookie** | JS can't read it = no XSS theft, automatic CSRF protection with SameSite | Requires same-origin setup | Web apps (preferred) |
| **JWT in memory** | Works cross-origin, good for mobile | XSS can steal it if in localStorage | APIs, mobile apps |

**Interview line**: "For browser apps, I prefer HttpOnly secure cookies because they eliminate the entire class of XSS token theft attacks."

### Cookie Flags

```
Set-Cookie: session=abc123;
  HttpOnly;     → JavaScript cannot read this cookie
  Secure;       → Only sent over HTTPS
  SameSite=Lax; → Prevents CSRF on cross-site requests
  Path=/;
  Max-Age=3600; → Short-lived (1 hour)
```

### Frontend Security Checklist

- **HTTPS everywhere** — never load mixed content
- **Content Security Policy (CSP)** — restrict script sources to prevent XSS injection
- **No secrets in React code** — API keys, tokens, DB URLs must never appear in frontend
- **Sanitize rendered content** — prevent XSS when displaying user-generated data
- **Short-lived tokens** — if using JWT, keep lifetime 15-60 minutes

---

## Layer 2: Node.js Backend Security

### Full Middleware Pipeline

```ts
// app.ts — Middleware order matters
import express from "express";
import helmet from "helmet";
import cors from "cors";
import rateLimit from "express-rate-limit";
import { authMiddleware } from "./middleware/auth";
import { rbacMiddleware } from "./middleware/rbac";
import { requestIdMiddleware } from "./middleware/requestId";

const app = express();

// 1. Security headers (before everything)
app.use(helmet());

// 2. CORS — only trusted origins
app.use(cors({
  origin: ["https://app.example.com"],
  credentials: true,
  methods: ["GET", "POST", "PUT", "DELETE"],
}));

// 3. Rate limiting — prevent abuse
app.use(rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 min window
  max: 100,                    // 100 requests per window
  standardHeaders: true,
  message: { error: "Too many requests, try again later" },
}));

// 4. Request ID for tracing
app.use(requestIdMiddleware);

// 5. Body parsing with size limit
app.use(express.json({ limit: "1mb" }));
```

### JWT Verification Middleware

```ts
// middleware/auth.ts
import { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";

const JWT_SECRET = process.env.JWT_SECRET!; // from Key Vault

export function authMiddleware(req: Request, res: Response, next: NextFunction) {
  const token = req.cookies?.session || req.headers.authorization?.replace("Bearer ", "");

  if (!token) {
    return res.status(401).json({ error: "Authentication required" });
  }

  try {
    const decoded = jwt.verify(token, JWT_SECRET, {
      algorithms: ["HS256"],
      issuer: "your-app",
      audience: "your-api",
    });
    req.user = decoded as { id: string; role: string; email: string };
    next();
  } catch (err) {
    return res.status(401).json({ error: "Invalid or expired token" });
  }
}
```

### RBAC Authorization Middleware

```ts
// middleware/rbac.ts
import { Request, Response, NextFunction } from "express";

type Role = "viewer" | "analyst" | "admin";

const permissions: Record<Role, string[]> = {
  viewer:  ["read:jobs", "read:reports"],
  analyst: ["read:jobs", "read:reports", "create:jobs", "pause:own_workflows"],
  admin:   ["read:jobs", "read:reports", "create:jobs", "pause:any_workflow", "manage:users"],
};

export function rbacMiddleware(requiredPermission: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    const userRole = req.user?.role as Role;

    if (!userRole || !permissions[userRole]?.includes(requiredPermission)) {
      return res.status(403).json({ error: "Insufficient permissions" });
    }

    next();
  };
}

// Usage in routes:
// router.post("/jobs", authMiddleware, rbacMiddleware("create:jobs"), jobController.triggerReport);
// router.get("/jobs/:id", authMiddleware, rbacMiddleware("read:jobs"), jobController.getStatus);
```

### Input Validation with Zod

```ts
// middleware/validate.ts
import { z, ZodSchema } from "zod";
import { Request, Response, NextFunction } from "express";

export function validateBody(schema: ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);

    if (!result.success) {
      return res.status(400).json({
        error: "Validation failed",
        details: result.error.issues.map((i) => ({
          field: i.path.join("."),
          message: i.message,
        })),
      });
    }

    req.body = result.data; // Use sanitized/validated data
    next();
  };
}

// Example schema
const triggerReportSchema = z.object({
  metric: z.enum(["revenue", "customers", "orders"]),
  dateRange: z.object({
    start: z.string().datetime(),
    end: z.string().datetime(),
  }),
  filters: z.record(z.string()).optional(),
});

// Usage:
// router.post("/reports", authMiddleware, validateBody(triggerReportSchema), controller.trigger);
```

### Role Permission Matrix

| Role | Read Reports | Trigger Jobs | Pause Workflow | Manage Users |
|------|-------------|-------------|----------------|-------------|
| **Viewer** | ✅ | ❌ | ❌ | ❌ |
| **Analyst** | ✅ | ✅ | ✅ (own only) | ❌ |
| **Admin** | ✅ | ✅ | ✅ (all) | ✅ |

---

## Layer 3: Node.js → Databricks Security

### Critical Rule

```
NEVER pass the end-user's browser token directly to Databricks.
```

**Why:**
- The user token is for your application, not for Databricks
- If you forward user tokens, you lose control over what Databricks access they get
- Impossible to enforce least privilege or audit properly

### Correct Approach: Service Principal with OAuth M2M

```
Node.js service
   │
   │ client_id + client_secret (from Key Vault)
   ▼
Azure AD / Databricks OAuth endpoint
   │
   │ short-lived access token (1-hour TTL)
   ▼
Node.js calls Databricks API with Bearer token
```

### OAuth M2M Token Exchange — Code

```ts
// dao/databricksAuth.ts
import axios from "axios";

let cachedToken: { token: string; expiresAt: number } | null = null;

export async function getDatabricksToken(): Promise<string> {
  // Return cached token if still valid (with 5-min buffer)
  if (cachedToken && cachedToken.expiresAt > Date.now() + 300_000) {
    return cachedToken.token;
  }

  // OAuth M2M token exchange
  const response = await axios.post(
    `${process.env.AZURE_AD_TENANT_URL}/oauth2/v2.0/token`,
    new URLSearchParams({
      client_id: process.env.DATABRICKS_CLIENT_ID!,      // from Key Vault
      client_secret: process.env.DATABRICKS_CLIENT_SECRET!, // from Key Vault
      scope: `${process.env.DATABRICKS_HOST}/.default`,
      grant_type: "client_credentials",
    }),
    { headers: { "Content-Type": "application/x-www-form-urlencoded" } }
  );

  cachedToken = {
    token: response.data.access_token,
    expiresAt: Date.now() + response.data.expires_in * 1000,
  };

  return cachedToken.token;
}

// Usage in Databricks DAO:
export async function callDatabricksApi(path: string, data?: any) {
  const token = await getDatabricksToken();

  return axios({
    method: data ? "POST" : "GET",
    url: `${process.env.DATABRICKS_HOST}/api${path}`,
    headers: { Authorization: `Bearer ${token}` },
    data,
  });
}
```

### Least Privilege for Service Principal

**Bad**: One admin token that can do everything.

**Good**: Service principal scoped to only what it needs:

| Permission | Allowed? |
|-----------|----------|
| Run specific approved jobs | ✅ |
| Read output tables | ✅ |
| Read specific Delta paths | ✅ |
| Manage users | ❌ |
| Delete tables | ❌ |
| Access other workspaces | ❌ |
| Create clusters | ❌ |

### Databricks Unity Catalog

Unity Catalog adds **table-level access control** and **audit logging** inside Databricks.

What it provides:
- **Fine-grained permissions**: Grant SELECT on specific tables to the service principal
- **Column masking**: Hide sensitive columns from certain principals
- **Row-level security**: Filter rows based on identity
- **Full audit trail**: Who queried what, when

**Interview line**: "Inside Databricks, I would use Unity Catalog for fine-grained table permissions, column masking, and audit logging on top of the service principal."

---

## Secrets Management

### Never Do This

```
❌ Hardcode tokens in source code
❌ Put secrets in .env committed to Git
❌ Store credentials in frontend JavaScript
❌ Share one token across all environments
❌ Use long-lived tokens that never rotate
```

### Do This Instead

```
✅ Use Azure Key Vault / AWS Secrets Manager / HashiCorp Vault
✅ Load secrets at runtime, not build time
✅ Rotate secrets on a schedule (quarterly or more)
✅ Different credentials per environment (dev/staging/prod)
✅ Use managed identities where possible (no secrets at all)
```

### What Goes in the Vault

| Secret | Where Used |
|--------|-----------|
| JWT signing key | Node.js auth middleware |
| Databricks client_id + client_secret | Node.js Databricks DAO |
| Database connection string | Node.js DB layer |
| Redis password | Node.js cache layer |
| OAuth provider secrets | Node.js auth service |

---

## Data Protection

### In Transit

| Path | Protection |
|------|-----------|
| Browser → Node.js | HTTPS (TLS 1.2+) |
| Node.js → Databricks | HTTPS (TLS 1.2+) |
| Node.js → Redis | TLS-encrypted connection |
| Node.js → Database | TLS-encrypted connection |

### At Rest

| Data | Protection |
|------|-----------|
| Database records | Encrypted disk (AES-256) |
| Delta Lake files | Encrypted object storage |
| Redis cache | Encrypted in-memory store |
| Logs | Redacted sensitive fields |

### In Logs — What to NEVER Log

```
❌ Passwords or password hashes
❌ Full JWT tokens
❌ API keys / secrets
❌ Credit card numbers
❌ Full SSN / ID numbers
❌ Complete PII without masking
```

**What to log instead:**

```ts
logger.info({
  requestId: "abc-123",
  userId: "user-456",
  action: "trigger_job",
  resource: "reports",
  role: "analyst",
  ip: req.ip,
  status: 202,
  // token: "REDACTED" — never log tokens
});
```

---

## Network Security

Mention this to sound senior-level.

### Controls

| Control | Purpose |
|---------|---------|
| **Private endpoints** | Databricks only reachable from your VPC/VNet |
| **VPC peering** | Node.js and Databricks in same private network |
| **IP allowlisting** | Only known backend IPs can reach Databricks |
| **Firewall rules** | Block all traffic except required ports |
| **No public access** | Databricks workspace not exposed to internet |

**Why it matters**: Even if a token is stolen, network restrictions prevent access from outside the trusted network.

**Interview line**: "I would place Databricks behind private endpoints so that even if credentials are compromised, network restrictions prevent unauthorized access."

---

## Audit and Observability

### Structured Audit Logging

Every security-relevant action should be logged:

```ts
// Log format
{
  "timestamp": "2026-04-17T10:20:00Z",
  "requestId": "abc-123",
  "userId": "user-456",
  "role": "analyst",
  "action": "trigger_databricks_job",
  "resource": "job:revenue-report",
  "status": "success",
  "ip": "10.0.1.50",
  "userAgent": "Mozilla/5.0..."
}
```

### Monitoring Alerts

| Alert | Trigger |
|-------|---------|
| Brute force attempt | >10 failed logins from same IP in 5 min |
| Privilege escalation | User accessing resources above their role |
| Unusual volume | >50 Databricks jobs in 1 hour (normal is 5) |
| Token abuse | Same token used from multiple IPs |
| Secret access | Unauthorized Key Vault access attempt |

---

## OWASP Top 10 — How Each Layer Addresses Them

| OWASP Threat | Your Mitigation |
|-------------|----------------|
| **A01: Broken Access Control** | RBAC middleware, least-privilege service principal |
| **A02: Cryptographic Failures** | TLS everywhere, encrypted at rest, Key Vault for secrets |
| **A03: Injection** | Zod input validation, parameterized queries |
| **A04: Insecure Design** | Layer-by-layer security, defense in depth |
| **A05: Security Misconfiguration** | Helmet headers, strict CORS, no default credentials |
| **A06: Vulnerable Components** | npm audit, Dependabot, regular dependency updates |
| **A07: Auth Failures** | OAuth 2.0, short-lived tokens, HttpOnly cookies |
| **A08: Data Integrity Failures** | Signed tokens (JWT), integrity checks on data pipelines |
| **A09: Logging Failures** | Structured logs, correlation IDs, audit trails |
| **A10: SSRF** | Validate/allowlist URLs before making server-side requests |

---

## Secure Request Flow — End to End

```
1. User clicks "Login" → redirected to Azure AD
2. Azure AD authenticates → returns authorization code
3. Node.js exchanges code for tokens → sets HttpOnly secure cookie
4. User clicks "Generate Report"
5. React sends POST /api/reports with cookie over HTTPS
6. Node.js middleware pipeline:
   a. Verify JWT/session (auth)
   b. Check RBAC permission (authorization)
   c. Validate input with Zod (injection prevention)
   d. Rate limit check (abuse prevention)
7. Node.js service gets Databricks OAuth M2M token from cache (or refreshes from Azure AD)
8. Credentials from Azure Key Vault (not hardcoded)
9. Node.js calls Databricks Jobs API with short-lived service token
10. Databricks validates service principal, checks Unity Catalog permissions
11. Databricks starts job, returns runId
12. Node.js logs audit event: { userId, action, jobId, requestId }
13. Node.js returns safe response to React (no internal details exposed)
```

---

## Common Interview Mistakes to Avoid

| Mistake | Why It's Weak | What to Say Instead |
|---------|--------------|-------------------|
| Only mention JWT + CORS | Too generic, doesn't show depth | Structure answer by layer with specific controls |
| Ignore authorization | Authentication alone is not enough | Separate auth from authz, show RBAC |
| Let frontend talk to Databricks | Exposes internal platform | "React never calls Databricks. Node uses a service principal." |
| Skip secrets management | Shows immaturity | "All credentials stored in Key Vault, rotated quarterly" |
| Forget about Databricks security | Misses the question | "Databricks uses Unity Catalog for table-level ACL + audit" |
| No code examples | Sounds theoretical | Show JWT middleware + RBAC middleware + Zod validation |

---

## 90-Second Spoken Version

> I would secure this system in layers with distinct trust boundaries. For authentication versus authorization: authentication proves identity, authorization determines permissions. Every layer handles both.
>
> For React to Node: users authenticate via OAuth with Azure AD. I prefer HttpOnly secure cookies for browser sessions because JavaScript cannot read them, eliminating XSS token theft. All communication over HTTPS with Content Security Policy headers.
>
> For Node.js: every request passes through a middleware pipeline — JWT verification, RBAC authorization checking viewer/analyst/admin roles, Zod input validation, Helmet for security headers, strict CORS, and rate limiting. All logged with correlation IDs.
>
> For Node to Databricks: I never forward the user's browser token. Instead, Node authenticates with a service principal using OAuth machine-to-machine flow — short-lived tokens, automatic rotation, clear service identity. The service principal follows least privilege: it can run approved jobs and read specific tables, nothing more. Credentials stored in Azure Key Vault, never in code.
>
> For data protection: encrypted in transit with TLS, encrypted at rest, never log secrets or PII. For network security: Databricks behind private endpoints with IP allowlisting. Inside Databricks, I use Unity Catalog for table-level access control and audit logging. This gives defense in depth across identity, authorization, secrets, transport, network, and observability.

---

## Quick Revision Formula

```
Security = Auth (OAuth) + Authz (RBAC) + Service Trust (M2M) + Secrets (Vault) +
           Validation (Zod) + Headers (Helmet) + Network (VPC) + Audit (Logs) +
           Databricks (Unity Catalog)
```

### Three Key Lines to Remember

```
1. Authentication = who are you; Authorization = what can you do. Every layer handles both.
2. Node.js calls Databricks with an OAuth M2M service principal, never with user tokens.
3. Secrets in Key Vault, data encrypted in transit + at rest, Databricks behind private endpoints.
```
