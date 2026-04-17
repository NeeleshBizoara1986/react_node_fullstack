# Q9 — Data-Driven Workflow Design (10/10 Preparation Guide)

## The Interview Question

> **The position requires building data-driven workflows. Could you explain how you would design a flexible workflow system that allows for different data transformation steps, conditional paths, and the ability to pause/resume processing? What data structures and patterns would you use?**

---

## Assessment Feedback (3/10) — Why You Lost Points

Your original answer:
- Discussed **general software architecture** (separation of concerns) instead of workflow design
- **Failed to address** the specific requirements: transformation steps, conditional paths, pause/resume
- **No data structures** mentioned (no DAG, no state machine)
- **No patterns** mentioned (no Pipeline, no Command, no Strategy)
- **No reference** to existing workflow tools (Airflow, Databricks Workflows, Step Functions)

To score **10/10**, you must:
1. Lead with **DAG** as the core data structure (not an array)
2. Show a **state machine** for workflow lifecycle management
3. Explain **checkpoint-based pause/resume** with idempotent steps
4. Show **decision nodes** for conditional branching
5. Name specific patterns: **Pipeline, Command, Strategy, Chain of Responsibility**
6. Show a **config-driven workflow definition** (JSON/DB, not hardcoded)
7. Provide an **orchestrator implementation** with code
8. Reference real-world tools: **Apache Airflow, Databricks Workflows, AWS Step Functions**

---

## 10/10 Interview Answer (Memorize This)

> I would design the workflow engine using a **Directed Acyclic Graph (DAG)** where each node represents a transformation step, validation, decision point, or external job like a Databricks task. I choose a DAG over a simple array because data workflows need **branching**, **parallel execution**, and **conditional paths** — an array only supports linear pipelines.
>
> Workflow definitions would be **config-driven** — stored as JSON in a database rather than hardcoded — so adding a new step or changing a condition is a configuration change, not a code deploy. Each workflow definition is **versioned** so running instances continue with their original definition while new runs use the updated version.
>
> For execution, I separate the system into an **orchestrator** and **workers**. The orchestrator reads the workflow definition, maintains a **state machine** for each workflow instance (CREATED → RUNNING → PAUSED → COMPLETED | FAILED), evaluates decision nodes using the workflow context, and enqueues the next executable step. Workers execute actual tasks — data transforms, validations, Databricks jobs — and report results back.
>
> For **pause/resume**, I persist a **checkpoint** after every completed step: current position, completed steps, outputs, and execution context. Resume loads the checkpoint and continues from the last unfinished step. This works reliably because each step is designed to be **idempotent** — running a step twice produces the same result, which is critical for crash recovery and retries.
>
> For **conditional paths**, I use **decision nodes** that evaluate rules against the workflow context. For example: if a file has more than 1 million rows, route to Databricks for processing; otherwise, process locally in Node.js.
>
> The design patterns I use are: **Pipeline pattern** for the overall step-by-step flow, **Command pattern** where each step is an executable command implementing a common interface, **Strategy pattern** for pluggable transformation types (CSV vs JSON vs Parquet), **Chain of Responsibility** for pre/post-step hooks (logging, validation, retries), and **queue-based async processing** for scalability.
>
> This is conceptually similar to how **Apache Airflow** orchestrates DAGs, how **Databricks Workflows** chain notebook tasks, and how **AWS Step Functions** use state machines — but tailored for our Node.js + Databricks stack.

---

## Why This Is a 10/10 Answer

| Criteria | Covered? |
|----------|----------|
| DAG as data structure | ✅ With explanation of why not an array |
| State machine for lifecycle | ✅ With valid state transitions |
| Pause/resume with checkpoints | ✅ With idempotency |
| Conditional paths (decision nodes) | ✅ With concrete example |
| Pipeline pattern | ✅ |
| Command pattern | ✅ With interface |
| Strategy pattern | ✅ For pluggable transforms |
| Chain of Responsibility | ✅ For hooks |
| Config-driven definition | ✅ JSON + versioning |
| Workflow versioning | ✅ |
| Orchestrator implementation | ✅ With code |
| Reference to Airflow/Databricks Workflows | ✅ |
| Databricks integration | ✅ |

---

## Easy Mental Model

Think of a workflow like a **Google Maps route with multiple stops**:

```
Each stop        = a workflow step (transform, validate, enrich)
Fork in the road = decision node (conditional path)
Stopping for gas = pause (checkpoint saved)
Resume driving   = resume from checkpoint
Detour           = error handling, alternative path
Final destination = workflow completed
```

The route is defined in advance (config), but execution is dynamic based on real-time conditions (workflow context).

---

## Why DAG, Not an Array?

### Array (Limited)

```
Step1 → Step2 → Step3 → Step4
```

Only supports linear execution. No branching, no parallel steps, no conditional skips.

### DAG (Flexible)

```
         Upload
           │
         Validate
           │
      ┌────┴────┐
      │ Valid?   │
     Yes        No
      │          │
  Transform   Send Error
      │
   ┌──┴──┐
   │      │
 Enrich  Index       ← parallel execution
   │      │
   └──┬──┘
      │
  Databricks Job
      │
 Generate Report
```

**DAG supports:**
- Branching (conditional paths)
- Parallel execution (multiple branches simultaneously)
- Merging (join after parallel branches)
- Skip/optional steps
- Resume from any node

This is the same approach used by Apache Airflow, Databricks Workflows, and AWS Step Functions.

---

## Reference Architectures

### Apache Airflow

- Workflows defined as **Python DAGs**
- Tasks have dependencies (`task_a >> task_b`)
- Scheduler evaluates next runnable tasks
- State stored in metadata database
- Supports retry, timeout, branching operators

### Databricks Workflows

- Chain **notebook tasks** into multi-step jobs
- Conditional paths using `dbutils.notebook.exit("value")`
- Auto-retry failed tasks
- Managed scheduling and monitoring

### AWS Step Functions

- Workflow defined as **JSON state machine**
- States: Task, Choice (branch), Parallel, Wait, Pass
- Built-in retry and error handling
- Visual workflow editor

**Interview line**: "My design is conceptually similar to how Airflow orchestrates DAGs and Step Functions use state machines — but built in Node.js with BullMQ for our specific stack."

---

## Config-Driven Workflow Definition

Store workflow definitions in JSON (or database), not in code:

```json
{
  "workflowId": "customer-import",
  "name": "Customer Import Pipeline",
  "version": 3,
  "steps": [
    {
      "id": "upload",
      "type": "task",
      "action": "uploadFile",
      "next": ["validate"],
      "timeout": 60000,
      "retries": 0
    },
    {
      "id": "validate",
      "type": "task",
      "action": "validateSchema",
      "next": ["decision_valid"],
      "timeout": 30000,
      "retries": 2
    },
    {
      "id": "decision_valid",
      "type": "decision",
      "condition": {
        "field": "validationResult",
        "operator": "==",
        "value": "passed"
      },
      "onTrue": "check_size",
      "onFalse": "send_error"
    },
    {
      "id": "check_size",
      "type": "decision",
      "condition": {
        "field": "recordCount",
        "operator": ">",
        "value": 1000000
      },
      "onTrue": "databricks_transform",
      "onFalse": "local_transform"
    },
    {
      "id": "databricks_transform",
      "type": "task",
      "action": "runDatabricksTransform",
      "next": ["enrich"],
      "timeout": 600000,
      "retries": 3
    },
    {
      "id": "local_transform",
      "type": "task",
      "action": "transformLocally",
      "next": ["enrich"],
      "timeout": 120000,
      "retries": 2
    },
    {
      "id": "enrich",
      "type": "task",
      "action": "enrichWithCatalog",
      "next": ["aggregate", "index"],
      "timeout": 120000,
      "retries": 2
    },
    {
      "id": "aggregate",
      "type": "task",
      "action": "aggregateTotals",
      "next": ["generate_report"],
      "timeout": 60000,
      "retries": 1
    },
    {
      "id": "index",
      "type": "task",
      "action": "indexForSearch",
      "next": ["generate_report"],
      "timeout": 60000,
      "retries": 1
    },
    {
      "id": "generate_report",
      "type": "task",
      "action": "generateReport",
      "joinFrom": ["aggregate", "index"],
      "next": [],
      "timeout": 30000,
      "retries": 1
    },
    {
      "id": "send_error",
      "type": "task",
      "action": "sendValidationError",
      "next": [],
      "timeout": 10000,
      "retries": 0
    }
  ]
}
```

### Why Config-Driven?

| Benefit | Explanation |
|---------|------------|
| **No redeploy** | Add/remove/modify steps without code changes |
| **Version control** | Running instances use their original definition version |
| **Multiple workflows** | Same engine, different definitions |
| **Non-engineer editing** | Business analysts can modify workflows via UI |
| **Auditability** | Track what changed between versions |

---

## Workflow Diagram

```
         ┌─────────────────┐
         │  Upload File     │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ Validate Schema  │
         └────────┬────────┘
                  │
                  ▼
        ┌──────────────────┐
        │ Decision: Valid?  │
        └───────┬──────┬───┘
              Yes       No
                │        │
                ▼        ▼
     ┌──────────────┐  ┌────────────────┐
     │ Check Size   │  │ Send Error     │
     └──────┬───┬───┘  └────────────────┘
            │   │
          >1M  ≤1M
            │   │
            ▼   ▼
  ┌──────────┐ ┌──────────────┐
  │Databricks│ │ Local        │
  │Transform │ │ Transform    │
  └────┬─────┘ └──────┬───────┘
       │               │
       └───────┬───────┘
               │
               ▼
      ┌─────────────────┐
      │  Enrich Data     │
      └───────┬─────────┘
              │
        ┌─────┴─────┐
        │           │
        ▼           ▼
  ┌──────────┐  ┌───────────┐
  │ Aggregate│  │ Index     │  ← parallel execution
  └────┬─────┘  └─────┬─────┘
       │               │
       └───────┬───────┘
               │ (join: both must complete)
               ▼
      ┌─────────────────┐
      │ Generate Report  │
      └─────────────────┘
```

---

## State Machine — Workflow Lifecycle

### Workflow Instance States

```
CREATED → RUNNING → COMPLETED
              │
              ├──→ PAUSED → RESUMED → RUNNING
              │
              ├──→ FAILED
              │
              └──→ CANCELLED
```

**Valid transitions:**

| From | To | Trigger |
|------|----|---------|
| CREATED | RUNNING | Start workflow |
| RUNNING | COMPLETED | All steps finished |
| RUNNING | PAUSED | User clicks Pause / system pause |
| RUNNING | FAILED | Step failed after retries exhausted |
| RUNNING | CANCELLED | User cancels |
| PAUSED | RUNNING | User clicks Resume |
| FAILED | RUNNING | User retries from failed step |

### Step States

```
PENDING → RUNNING → SUCCESS
              │
              ├──→ FAILED → RETRYING → RUNNING (retry loop)
              │
              └──→ SKIPPED (conditional path not taken)
```

### Why State Machine?

Because it prevents invalid operations:
- Cannot resume a COMPLETED workflow
- Cannot pause a CREATED workflow
- Cannot start a RUNNING workflow again

---

## Pause / Resume Design

### Core Idea: Checkpoints

After every successful step, persist a **checkpoint**:

```json
{
  "instanceId": "wf-1001",
  "workflowId": "customer-import",
  "workflowVersion": 3,
  "status": "PAUSED",
  "currentStep": "enrich",
  "completedSteps": ["upload", "validate", "decision_valid", "check_size", "local_transform"],
  "context": {
    "filePath": "/uploads/customers.csv",
    "recordCount": 50000,
    "validationResult": "passed",
    "transformedPath": "/processed/customers_clean.csv"
  },
  "stepOutputs": {
    "upload": { "filePath": "/uploads/customers.csv", "size": 1048576 },
    "validate": { "validationResult": "passed", "recordCount": 50000 },
    "local_transform": { "transformedPath": "/processed/customers_clean.csv" }
  },
  "pausedAt": "2026-04-17T10:20:00Z",
  "updatedAt": "2026-04-17T10:20:00Z"
}
```

### Resume Logic

```
1. Load workflow instance from DB
2. Read currentStep → "enrich"
3. Restore saved context
4. Find step definition for "enrich"
5. Skip all completed steps
6. Execute "enrich" with restored context
7. Continue pipeline from there
```

### Why Idempotency Is Critical for Resume

**Idempotent** means: running a step twice produces the same result.

| Bad (Not Idempotent) | Good (Idempotent) |
|---------------------|-------------------|
| `INSERT INTO customers (...)` → duplicates | `INSERT ... ON CONFLICT DO UPDATE` → safe rerun |
| `count += 1` → wrong total on retry | `count = calculateFromSource()` → correct every time |
| `transferMoney(100)` → double charge | `transferMoney(100, txId: "abc")` → idempotent key prevents double-processing |

**Interview line**: "Pause/resume is reliable because the engine is checkpoint-based and steps are idempotent. Even if a worker crashes mid-step, the retry produces the correct result."

---

## Conditional Paths (Decision Nodes)

### How They Work

Decision nodes evaluate a **condition** against the workflow **context** and route to the appropriate next step.

```
┌────────────────────────────────┐
│ Decision: recordCount > 1M?    │
└───────────┬────────┬───────────┘
          true     false
            │        │
            ▼        ▼
   Databricks    Local Transform
   Transform
```

### Condition Evaluator

```ts
interface Condition {
  field: string;
  operator: "==" | "!=" | ">" | "<" | ">=" | "<=" | "in" | "contains";
  value: any;
}

function evaluateCondition(condition: Condition, context: Record<string, any>): boolean {
  const fieldValue = context[condition.field];

  switch (condition.operator) {
    case "==":       return fieldValue === condition.value;
    case "!=":       return fieldValue !== condition.value;
    case ">":        return fieldValue > condition.value;
    case "<":        return fieldValue < condition.value;
    case ">=":       return fieldValue >= condition.value;
    case "<=":       return fieldValue <= condition.value;
    case "in":       return condition.value.includes(fieldValue);
    case "contains": return String(fieldValue).includes(condition.value);
    default:         return false;
  }
}
```

---

## Design Patterns — Name All Five

### 1. Pipeline Pattern

The overall workflow is a **pipeline** of steps, where each step's output feeds into the next step's input.

```
Input → Step 1 → Step 2 → Step 3 → Output
```

Each step receives context, does work, and returns updated context.

### 2. Command Pattern

Each step implements a common **command interface**:

```ts
interface WorkflowStep {
  execute(context: WorkflowContext): Promise<StepResult>;
  rollback?(context: WorkflowContext): Promise<void>; // Optional: undo on failure
}

interface WorkflowContext {
  instanceId: string;
  workflowId: string;
  data: Record<string, any>;
}

interface StepResult {
  success: boolean;
  output: Record<string, any>;
  error?: string;
}
```

Concrete implementations:

```ts
class ValidateSchemaStep implements WorkflowStep {
  async execute(context: WorkflowContext): Promise<StepResult> {
    const { filePath } = context.data;
    const result = await validateCSVSchema(filePath);
    return {
      success: result.valid,
      output: { validationResult: result.valid ? "passed" : "failed", recordCount: result.rowCount },
      error: result.valid ? undefined : result.errors.join(", "),
    };
  }
}

class RunDatabricksTransformStep implements WorkflowStep {
  async execute(context: WorkflowContext): Promise<StepResult> {
    const runId = await databricksDao.submitJobRun(1, {
      file_path: context.data.filePath,
      record_count: String(context.data.recordCount),
    });

    // Poll until complete
    let status = await databricksDao.getRunStatus(runId);
    while (status.state === "RUNNING" || status.state === "PENDING") {
      await new Promise((r) => setTimeout(r, 5000));
      status = await databricksDao.getRunStatus(runId);
    }

    return {
      success: status.resultState === "SUCCESS",
      output: { transformedPath: `/results/${context.instanceId}/output` },
      error: status.resultState !== "SUCCESS" ? status.stateMessage : undefined,
    };
  }
}

class TransformLocallyStep implements WorkflowStep {
  async execute(context: WorkflowContext): Promise<StepResult> {
    const transformed = await localTransform(context.data.filePath);
    return {
      success: true,
      output: { transformedPath: transformed.outputPath },
    };
  }
}
```

### 3. Strategy Pattern

Pluggable transformation strategies based on file type or size:

```ts
interface TransformStrategy {
  transform(input: string, context: Record<string, any>): Promise<string>;
}

class CSVTransformStrategy implements TransformStrategy {
  async transform(input: string, context: Record<string, any>): Promise<string> {
    // CSV-specific transformation
    return outputPath;
  }
}

class JSONTransformStrategy implements TransformStrategy {
  async transform(input: string, context: Record<string, any>): Promise<string> {
    // JSON-specific transformation
    return outputPath;
  }
}

class DatabricksTransformStrategy implements TransformStrategy {
  async transform(input: string, context: Record<string, any>): Promise<string> {
    // Offload to Databricks for large datasets
    const runId = await databricksDao.submitJobRun(1, { file_path: input });
    await waitForCompletion(runId);
    return `/results/${context.instanceId}/output`;
  }
}

// Factory selects strategy based on context
function getTransformStrategy(context: Record<string, any>): TransformStrategy {
  if (context.recordCount > 1_000_000) return new DatabricksTransformStrategy();
  if (context.fileType === "json") return new JSONTransformStrategy();
  return new CSVTransformStrategy();
}
```

### 4. Chain of Responsibility

Pre/post hooks that wrap each step execution:

```ts
// Each handler in the chain can modify behavior before/after step execution
type StepHandler = (
  step: WorkflowStep,
  context: WorkflowContext,
  next: () => Promise<StepResult>
) => Promise<StepResult>;

const loggingHandler: StepHandler = async (step, context, next) => {
  logger.info({ stepId: step.constructor.name, event: "step_started" });
  const start = Date.now();
  const result = await next();
  logger.info({ stepId: step.constructor.name, duration: Date.now() - start, event: "step_completed" });
  return result;
};

const retryHandler: StepHandler = async (step, context, next) => {
  let attempts = 0;
  const maxRetries = 3;
  while (attempts <= maxRetries) {
    try {
      return await next();
    } catch (err) {
      attempts++;
      if (attempts > maxRetries) throw err;
      await new Promise((r) => setTimeout(r, 2000 * Math.pow(2, attempts))); // exponential backoff
    }
  }
  throw new Error("Unreachable");
};

const timeoutHandler: StepHandler = async (step, context, next) => {
  const timeout = 300_000; // 5 min
  return Promise.race([
    next(),
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error("Step timed out")), timeout)
    ),
  ]);
};
```

### 5. Queue-Based Async Processing

Each step is enqueued as a job in BullMQ, so the system scales and handles failures gracefully.

---

## Orchestrator Implementation

```ts
// orchestrator/workflowEngine.ts

interface WorkflowDefinition {
  workflowId: string;
  version: number;
  steps: StepDefinition[];
}

interface StepDefinition {
  id: string;
  type: "task" | "decision";
  action?: string;
  next?: string[];
  joinFrom?: string[];
  condition?: Condition;
  onTrue?: string;
  onFalse?: string;
  timeout?: number;
  retries?: number;
}

interface WorkflowInstance {
  instanceId: string;
  workflowId: string;
  workflowVersion: number;
  status: "CREATED" | "RUNNING" | "PAUSED" | "COMPLETED" | "FAILED" | "CANCELLED";
  currentSteps: string[];
  completedSteps: string[];
  context: Record<string, any>;
  stepOutputs: Record<string, any>;
}

// Step registry: maps action names to implementations
const stepRegistry: Record<string, WorkflowStep> = {
  uploadFile: new UploadFileStep(),
  validateSchema: new ValidateSchemaStep(),
  runDatabricksTransform: new RunDatabricksTransformStep(),
  transformLocally: new TransformLocallyStep(),
  enrichWithCatalog: new EnrichStep(),
  aggregateTotals: new AggregateStep(),
  indexForSearch: new IndexStep(),
  generateReport: new GenerateReportStep(),
  sendValidationError: new SendErrorStep(),
};

class WorkflowEngine {
  async startWorkflow(workflowId: string, initialData: Record<string, any>): Promise<string> {
    // 1. Load definition
    const definition = await workflowRepository.getDefinition(workflowId);

    // 2. Create instance
    const instance: WorkflowInstance = {
      instanceId: generateId(),
      workflowId: definition.workflowId,
      workflowVersion: definition.version,
      status: "RUNNING",
      currentSteps: [definition.steps[0].id], // Start at first step
      completedSteps: [],
      context: initialData,
      stepOutputs: {},
    };

    await workflowRepository.saveInstance(instance);

    // 3. Execute first step
    await this.executeNextSteps(instance, definition);

    return instance.instanceId;
  }

  async executeNextSteps(instance: WorkflowInstance, definition: WorkflowDefinition) {
    while (instance.currentSteps.length > 0 && instance.status === "RUNNING") {
      const executableSteps = this.getExecutableSteps(instance, definition);

      if (executableSteps.length === 0) break;

      // Execute all ready steps (supports parallel execution)
      const results = await Promise.allSettled(
        executableSteps.map((stepDef) => this.executeStep(instance, stepDef, definition))
      );

      // Check if any step failed
      for (const result of results) {
        if (result.status === "rejected") {
          instance.status = "FAILED";
          await workflowRepository.saveInstance(instance);
          return;
        }
      }

      // Determine next steps
      this.advanceToNextSteps(instance, definition);
      await workflowRepository.saveInstance(instance); // checkpoint
    }

    // Check if workflow is complete (no more steps to execute)
    if (instance.currentSteps.length === 0 && instance.status === "RUNNING") {
      instance.status = "COMPLETED";
      await workflowRepository.saveInstance(instance);
    }
  }

  private async executeStep(
    instance: WorkflowInstance,
    stepDef: StepDefinition,
    definition: WorkflowDefinition
  ): Promise<void> {
    if (stepDef.type === "decision") {
      // Decision node: evaluate condition and route
      const result = evaluateCondition(stepDef.condition!, instance.context);
      const nextStepId = result ? stepDef.onTrue! : stepDef.onFalse!;
      instance.completedSteps.push(stepDef.id);
      instance.currentSteps = instance.currentSteps
        .filter((s) => s !== stepDef.id)
        .concat(nextStepId);
      return;
    }

    // Task node: execute the action
    const step = stepRegistry[stepDef.action!];
    if (!step) throw new Error(`Unknown action: ${stepDef.action}`);

    const context: WorkflowContext = {
      instanceId: instance.instanceId,
      workflowId: instance.workflowId,
      data: instance.context,
    };

    const result = await step.execute(context);

    if (!result.success) {
      throw new Error(result.error ?? `Step ${stepDef.id} failed`);
    }

    // Merge step output into workflow context
    instance.context = { ...instance.context, ...result.output };
    instance.stepOutputs[stepDef.id] = result.output;
    instance.completedSteps.push(stepDef.id);
    instance.currentSteps = instance.currentSteps.filter((s) => s !== stepDef.id);
  }

  private getExecutableSteps(instance: WorkflowInstance, definition: WorkflowDefinition): StepDefinition[] {
    return instance.currentSteps
      .map((id) => definition.steps.find((s) => s.id === id)!)
      .filter((stepDef) => {
        // For join nodes, check if all dependencies are completed
        if (stepDef.joinFrom) {
          return stepDef.joinFrom.every((dep) => instance.completedSteps.includes(dep));
        }
        return true;
      });
  }

  private advanceToNextSteps(instance: WorkflowInstance, definition: WorkflowDefinition) {
    const newSteps: string[] = [];

    for (const completedId of instance.completedSteps) {
      const stepDef = definition.steps.find((s) => s.id === completedId);
      if (stepDef?.next) {
        for (const nextId of stepDef.next) {
          if (
            !instance.completedSteps.includes(nextId) &&
            !instance.currentSteps.includes(nextId) &&
            !newSteps.includes(nextId)
          ) {
            newSteps.push(nextId);
          }
        }
      }
    }

    instance.currentSteps = [...instance.currentSteps, ...newSteps];
  }

  // ─── Pause ───
  async pauseWorkflow(instanceId: string) {
    const instance = await workflowRepository.getInstance(instanceId);
    if (instance.status !== "RUNNING") {
      throw new Error(`Cannot pause workflow in ${instance.status} state`);
    }
    instance.status = "PAUSED";
    await workflowRepository.saveInstance(instance);
  }

  // ─── Resume ───
  async resumeWorkflow(instanceId: string) {
    const instance = await workflowRepository.getInstance(instanceId);
    if (instance.status !== "PAUSED") {
      throw new Error(`Cannot resume workflow in ${instance.status} state`);
    }
    instance.status = "RUNNING";
    const definition = await workflowRepository.getDefinition(instance.workflowId);
    await this.executeNextSteps(instance, definition);
  }
}
```

---

## Database Tables

### `workflow_definitions`

| Column | Type | Description |
|--------|------|------------|
| id | UUID | Primary key |
| workflow_id | VARCHAR | Logical identifier (e.g., "customer-import") |
| version | INT | Version number (auto-increment per workflow_id) |
| name | VARCHAR | Human-readable name |
| definition_json | JSONB | The full DAG definition |
| created_at | TIMESTAMP | When created |
| is_active | BOOLEAN | Is this the current active version? |

### `workflow_instances`

| Column | Type | Description |
|--------|------|------------|
| id | UUID | Instance primary key |
| workflow_id | VARCHAR | Which workflow definition |
| workflow_version | INT | Pinned to this version |
| status | ENUM | CREATED/RUNNING/PAUSED/COMPLETED/FAILED/CANCELLED |
| current_steps | JSONB | Array of step IDs currently executing |
| completed_steps | JSONB | Array of completed step IDs |
| context_json | JSONB | Current workflow context (inputs + accumulated outputs) |
| step_outputs | JSONB | Output from each completed step |
| started_by | VARCHAR | User who triggered it |
| created_at | TIMESTAMP | When created |
| updated_at | TIMESTAMP | Last checkpoint save |

### `step_executions`

| Column | Type | Description |
|--------|------|------------|
| id | UUID | Primary key |
| instance_id | UUID | FK to workflow_instances |
| step_id | VARCHAR | Which step in the DAG |
| status | ENUM | PENDING/RUNNING/SUCCESS/FAILED/SKIPPED |
| attempt | INT | Retry attempt number |
| started_at | TIMESTAMP | When started |
| finished_at | TIMESTAMP | When finished |
| output_json | JSONB | Step output (for debugging and audit) |
| error | TEXT | Error message if failed |

### `workflow_events` (Audit)

| Column | Type | Description |
|--------|------|------------|
| id | UUID | Primary key |
| instance_id | UUID | FK to workflow_instances |
| event_type | VARCHAR | started/paused/resumed/step_completed/step_failed/completed |
| payload | JSONB | Event details |
| created_at | TIMESTAMP | When event occurred |

---

## Timeout and Webhook Patterns

### Per-Step Timeout

Each step definition includes a `timeout` field. The orchestrator enforces it:

```ts
async function executeWithTimeout(step: WorkflowStep, context: WorkflowContext, timeoutMs: number) {
  return Promise.race([
    step.execute(context),
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error(`Step timed out after ${timeoutMs}ms`)), timeoutMs)
    ),
  ]);
}
```

### Webhook/Callback for External Steps

For long-running external steps (like Databricks jobs), instead of polling:

```ts
// 1. Start external job and register callback URL
class ExternalJobStep implements WorkflowStep {
  async execute(context: WorkflowContext): Promise<StepResult> {
    const callbackUrl = `${API_BASE}/api/workflows/${context.instanceId}/callback/${this.stepId}`;

    await externalService.startJob({
      params: context.data,
      callbackUrl,  // External service calls this when done
    });

    // Return "pending" — orchestrator pauses this step until callback arrives
    return { success: true, output: { awaitingCallback: true } };
  }
}

// 2. Callback endpoint resumes the workflow
app.post("/api/workflows/:instanceId/callback/:stepId", async (req, res) => {
  const { instanceId, stepId } = req.params;
  const result = req.body;

  // Store callback result and resume workflow
  await workflowEngine.completeExternalStep(instanceId, stepId, result);
  res.status(200).json({ received: true });
});
```

---

## Testing Workflows

### Unit Test: Individual Steps

```ts
test("ValidateSchemaStep passes for valid CSV", async () => {
  const step = new ValidateSchemaStep();
  const context: WorkflowContext = {
    instanceId: "test-1",
    workflowId: "test",
    data: { filePath: "/test/valid.csv" },
  };

  const result = await step.execute(context);
  expect(result.success).toBe(true);
  expect(result.output.validationResult).toBe("passed");
});
```

### Integration Test: Full Workflow Execution

```ts
test("Customer import workflow completes for small file", async () => {
  const engine = new WorkflowEngine();
  const instanceId = await engine.startWorkflow("customer-import", {
    filePath: "/test/small_customers.csv",
    recordCount: 500,
  });

  const instance = await workflowRepository.getInstance(instanceId);
  expect(instance.status).toBe("COMPLETED");
  expect(instance.completedSteps).toContain("local_transform"); // Not Databricks (file is small)
  expect(instance.completedSteps).not.toContain("databricks_transform");
});
```

### Test: Pause/Resume

```ts
test("Workflow can pause and resume from checkpoint", async () => {
  const engine = new WorkflowEngine();
  const instanceId = await engine.startWorkflow("customer-import", { filePath: "/test/data.csv" });

  // Pause after validation
  await engine.pauseWorkflow(instanceId);
  const paused = await workflowRepository.getInstance(instanceId);
  expect(paused.status).toBe("PAUSED");
  expect(paused.completedSteps).toContain("validate");

  // Resume — should NOT re-run completed steps
  await engine.resumeWorkflow(instanceId);
  const completed = await workflowRepository.getInstance(instanceId);
  expect(completed.status).toBe("COMPLETED");
});
```

---

## 90-Second Spoken Version

> I would design the workflow engine using a Directed Acyclic Graph where each node represents a transformation step, validation, decision point, or external task like a Databricks job. I choose a DAG over a simple array because data workflows need branching, parallel execution, and conditional paths.
>
> Workflow definitions are config-driven — stored as versioned JSON in a database, not hardcoded — so adding steps or changing conditions is a configuration change, not a code deploy.
>
> For execution, I separate the orchestrator from workers. The orchestrator maintains a state machine for each workflow instance — CREATED, RUNNING, PAUSED, COMPLETED, FAILED — evaluates decision nodes, and enqueues executable steps. Workers execute actual tasks and report results.
>
> For pause/resume, I persist checkpoints after every step: current position, completed steps, outputs, and context. Resume loads the checkpoint and continues from the last unfinished step. This works because steps are designed to be idempotent.
>
> For conditional paths, decision nodes evaluate workflow context — for example, routing large files to Databricks and small files to local processing.
>
> The patterns I use: Pipeline for step-by-step flow, Command pattern where each step implements a common interface, Strategy for pluggable transformation types, Chain of Responsibility for hooks like logging and retries, and queue-based async processing for scalability. This is conceptually similar to how Airflow orchestrates DAGs and Step Functions use state machines, but built for our Node.js stack.

---

## Quick Revision Formula

```
Workflow = DAG structure + State Machine lifecycle + Checkpoint resume + Decision branching
Patterns = Pipeline + Command + Strategy + Chain of Responsibility + Queue
Data     = workflow_definitions (versioned) + workflow_instances (state) + step_executions (history)
Similar  = Apache Airflow DAGs / AWS Step Functions / Databricks Workflows
```

### Three Key Lines to Remember

```
1. A DAG supports branching, parallel paths, and conditional execution — an array cannot.
2. Pause/resume works because state is checkpointed after every step and steps are idempotent.
3. Patterns: Pipeline for flow, Command for steps, Strategy for transforms, Chain of Responsibility for hooks.
```
