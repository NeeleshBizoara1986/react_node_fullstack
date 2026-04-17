# Q9 — Data-Driven Workflow Design (10/10 Preparation Guide)

## The Interview Question

> **The position requires building data-driven workflows. Could you explain how you would design a flexible workflow system that allows for different data transformation steps, conditional paths, and the ability to pause/resume processing? What data structures and patterns would you use?**

---

## What the Interviewer Really Wants

This question is not asking for only general architecture or separation of concerns.

The interviewer wants to know whether you can design a real **workflow engine** that supports:

- multiple transformation steps
- branching / conditional paths
- pause and resume
- state tracking
- scalability
- fault tolerance
- integration with systems like Databricks

In simple words, they want to know if you can design something similar to:

- Apache Airflow
- Databricks Workflows
- AWS Step Functions
- a custom workflow orchestration system

---

## 10/10 Interview Answer

> I would design the workflow engine using a **Directed Acyclic Graph (DAG)** where each node represents a transformation step, validation step, decision, or external job such as a Databricks task. I prefer a DAG because it supports not only linear execution but also **branching, optional paths, and parallel steps**, which are common in data workflows.
>
> To make the system flexible, I would store workflow definitions as **metadata in JSON or database configuration** instead of hardcoding them. That way, adding a new transformation step or changing a branch condition does not require redeploying the whole application.
>
> For execution, I would separate the system into a **workflow orchestrator** and **workers**. The orchestrator is responsible for reading the workflow definition, tracking the workflow instance state, evaluating conditions, and deciding the next executable step. Workers perform the actual work, such as transforming data, validating files, enriching records, or triggering Databricks jobs.
>
> For state management, I would use a **state machine model**. A workflow instance would move through states like `CREATED`, `RUNNING`, `PAUSED`, `COMPLETED`, or `FAILED`, while each step would have its own lifecycle like `PENDING`, `RUNNING`, `SUCCESS`, or `FAILED`.
>
> For **pause and resume**, I would persist a checkpoint after every completed step, including the current step, execution context, completed steps, and outputs. That allows the workflow to resume safely from the last successful checkpoint instead of restarting from the beginning. To make resume reliable, I would design each step to be as **idempotent** as possible.
>
> For **conditional paths**, I would use decision nodes that evaluate rules against the workflow context. For example, if a file contains more than one million rows, the workflow can branch to a Databricks processing step; otherwise, it can follow a lighter local processing path.
>
> The main patterns I would use are:
>
> - **DAG** for workflow modeling
> - **State machine** for lifecycle management
> - **Command pattern** for step execution
> - **Strategy pattern** for pluggable transformation types
> - **Queue-based async processing** for scalability
>
> This design gives flexibility, scalability, failure recovery, and strong support for pause/resume and branching.

---

## Easy Mental Model

Think of a workflow like a **Google Maps route**.

- each **step** is a checkpoint
- some steps are always next
- some steps depend on conditions
- you can stop the trip and continue later
- the system remembers where you left off

Example:

```text
Upload File
   ↓
Validate File
   ↓
Is file valid?
  /   \
Yes    No
↓       ↓
Transform Data   Send Error Report
↓
Enrich Data
↓
Run Databricks Aggregation
↓
Generate Final Output
```

That is a workflow.

---

## High-Level Architecture

```text
React UI
   ↓
Node.js API / Workflow Orchestrator
   ↓
Database (workflow definitions + state)
   ↓
Queue
   ↓
Workers / Databricks Jobs
```

### Responsibilities of Each Layer

#### React UI
- create workflow
- start workflow
- pause workflow
- resume workflow
- show workflow progress/status

#### Node.js Orchestrator
- reads workflow definition
- figures out next executable step
- sends work to queue/workers
- stores progress
- applies branching logic

#### Database
Stores:
- workflow definitions
- workflow instance state
- step execution status
- pause/resume checkpoints

#### Queue / Workers
- execute tasks asynchronously
- prevent API from blocking
- scale independently

#### Databricks
- runs heavy transformations or analytics steps
- should be used for big-data or compute-heavy nodes

---

## Best Data Structure: DAG

### Why DAG?

Use a **Directed Acyclic Graph**.

- **Directed** = steps move forward
- **Acyclic** = no infinite loops
- **Graph** = supports branching and multiple execution paths

### Why Not Just an Array?

An array works only for a simple linear pipeline:

```text
Step1 → Step2 → Step3
```

But if you need:

- branching
- optional paths
- parallel steps
- resume from a specific node

then a **DAG** is much better.

---

## Example Workflow Definition (Config-Driven)

```json
{
  "workflowId": "customer-import",
  "name": "Customer Import Workflow",
  "steps": [
    {
      "id": "upload",
      "type": "task",
      "action": "uploadFile",
      "next": ["validate"]
    },
    {
      "id": "validate",
      "type": "task",
      "action": "validateFile",
      "next": ["decision_valid"]
    },
    {
      "id": "decision_valid",
      "type": "decision",
      "condition": "fileIsValid",
      "onTrue": "transform",
      "onFalse": "reject"
    },
    {
      "id": "transform",
      "type": "task",
      "action": "transformData",
      "next": ["enrich"]
    },
    {
      "id": "enrich",
      "type": "task",
      "action": "enrichData",
      "next": ["databricks_job"]
    },
    {
      "id": "databricks_job",
      "type": "task",
      "action": "runDatabricksAggregation",
      "next": ["complete"]
    },
    {
      "id": "reject",
      "type": "task",
      "action": "sendValidationError",
      "next": []
    },
    {
      "id": "complete",
      "type": "task",
      "action": "generateReport",
      "next": []
    }
  ]
}
```

### Why Store This as Configuration?

Because then you can:

- add new steps without changing code
- modify conditions without redeploying
- support many workflow types with one engine
- version your workflows

---

## Workflow Diagram

```text
         ┌─────────────┐
         │ Upload File │
         └──────┬──────┘
                │
                ▼
        ┌───────────────┐
        │ Validate File │
        └──────┬────────┘
               │
               ▼
      ┌───────────────────┐
      │ Decision: Valid ? │
      └──────┬───────┬────┘
             │Yes    │No
             ▼       ▼
   ┌────────────────┐  ┌────────────────────┐
   │ Transform Data │  │ Send Error Report  │
   └──────┬─────────┘  └────────────────────┘
          │
          ▼
   ┌──────────────┐
   │ Enrich Data  │
   └──────┬───────┘
          │
          ▼
 ┌──────────────────────┐
 │ Run Databricks Job   │
 └──────┬───────────────┘
        │
        ▼
 ┌──────────────────────┐
 │ Generate Final Report│
 └──────────────────────┘
```

---

## State Machine Pattern

A workflow is not only a set of steps. It also has a **lifecycle**.

### Workflow States

```text
CREATED → RUNNING → PAUSED → RESUMED → COMPLETED
                    ↓
                  FAILED
```

### Step States

```text
PENDING → RUNNING → SUCCESS
              ↓
            FAILED
              ↓
            RETRYING
```

### Why State Machine?

Because it clearly defines valid transitions.

Examples:
- you can pause only when workflow is `RUNNING`
- you can resume only when workflow is `PAUSED`
- you cannot resume a `COMPLETED` workflow

This makes the system predictable and prevents invalid operations.

---

## Pause / Resume Design

This is one of the most important parts of the answer.

### Core Idea

After every successful step, store a **checkpoint**.

Each checkpoint should include:

- workflow instance id
- current step id
- workflow status
- completed steps
- step outputs
- execution context
- timestamps

### Example Checkpoint

```json
{
  "workflowInstanceId": "wf_1001",
  "workflowId": "customer-import",
  "currentStep": "enrich",
  "status": "PAUSED",
  "completedSteps": ["upload", "validate", "transform"],
  "context": {
    "filePath": "/uploads/customers.csv",
    "validRows": 9000,
    "invalidRows": 120
  },
  "updatedAt": "2026-04-17T10:20:00Z"
}
```

### Resume Logic

When the user clicks **Resume**:

1. load the workflow instance from the database
2. read the `currentStep`
3. restore the saved context
4. continue execution from the last unfinished step

### Important Interview Line

> Pause/resume becomes reliable only if the engine is **checkpoint-based** and the steps are as **idempotent** as possible.

---

## What Is Idempotent and Why It Matters?

Idempotent means a step can run more than once without causing incorrect results.

### Example

#### Bad
- insert the same customer record again → duplicates

#### Good
- upsert customer by unique key → safe to rerun

Why it matters:
- system crash recovery
- retries
- pause/resume safety
- distributed execution consistency

---

## Conditional Paths (Branching)

A flexible workflow must support branching.

### How Branching Works

Use **decision nodes**.

A decision node evaluates workflow context and chooses the next path.

Example:

```text
If total rows > 1 million → send to Databricks
Else → process locally in Node.js
```

### Diagram

```text
            ┌───────────────────────┐
            │ Count Records in File │
            └──────────┬────────────┘
                       │
                       ▼
           ┌────────────────────────────┐
           │ Decision: rows > 1,000,000 │
           └───────────┬─────────┬──────┘
                       │Yes      │No
                       ▼         ▼
          ┌──────────────────┐  ┌────────────────┐
          │ Run in Databricks│  │ Run in Node.js │
          └──────────────────┘  └────────────────┘
```

### Example Decision Config

```json
{
  "id": "size_check",
  "type": "decision",
  "condition": {
    "field": "recordCount",
    "operator": ">",
    "value": 1000000
  },
  "onTrue": "databricks_processing",
  "onFalse": "local_processing"
}
```

---

## Patterns to Mention in the Interview

Because the question explicitly asks about **data structures and patterns**, say them directly.

### 1. DAG
Use for workflow modeling.

### 2. State Machine
Use for workflow lifecycle and valid transitions.

### 3. Command Pattern
Each step is implemented as a command or executable unit.

Example:

```ts
interface WorkflowStep {
  execute(context: WorkflowContext): Promise<WorkflowContext>;
}
```

Possible implementations:
- `ValidateFileStep`
- `TransformDataStep`
- `EnrichDataStep`
- `RunDatabricksStep`

### 4. Strategy Pattern
Use for pluggable transformation logic.

Examples:
- CSV transformation strategy
- JSON transformation strategy
- local processing strategy
- Databricks processing strategy

### 5. Queue-Based Async Processing
Use queues so API responses stay fast and workers process tasks independently.

### 6. Event-Driven Pattern
Emit events such as:
- step started
- step completed
- step failed
- workflow paused
- workflow resumed

This helps UI updates, logging, notifications, and observability.

---

## Execution Model

### Workflow Execution Flow

```text
1. User starts workflow
2. Orchestrator loads workflow definition
3. Creates workflow instance in DB
4. Marks first step as RUNNING
5. Sends step to queue
6. Worker executes step
7. Worker stores output + updates state
8. Orchestrator determines next step(s)
9. Repeat until completed / paused / failed
```

### Execution Diagram

```text
User
 │
 ▼
Start Workflow
 │
 ▼
Node.js Orchestrator
 │
 ├── Load workflow definition
 │
 ├── Create workflow instance
 │
 ├── Push first step to queue
 │
 ▼
Worker
 │
 ├── Execute step
 ├── Save output
 ├── Update DB
 └── Emit event
 │
 ▼
Orchestrator
 │
 ├── Evaluate conditions
 ├── Decide next step
 └── Queue next task
```

---

## Example with Databricks

Since your interview context includes Databricks, bring that into the answer.

### Example Workflow

A retail data pipeline might look like this:

1. upload sales data
2. validate schema
3. clean missing values
4. if file is large → process in Databricks
5. else → process locally
6. enrich with product catalog
7. aggregate daily totals
8. generate dashboard dataset

### Diagram

```text
Validate
   ↓
Decision: Big file?
  /   \
Yes    No
↓       ↓
Databricks Job   Local Transform
      \          /
       ▼        ▼
      Enrich and Aggregate
```

### Why This Is Good

Because:
- workflow engine handles orchestration
- Databricks handles heavy compute
- Node.js coordinates execution but does not do heavy data crunching

That is a strong fullstack answer.

---

## Database Tables You Can Mention

Mentioning persistence adds depth and sounds senior.

### `workflow_definitions`
Stores reusable workflow templates.

| id | name | version | definition_json |
|----|------|---------|-----------------|

### `workflow_instances`
Stores actual running workflows.

| id | workflow_id | status | current_step | context_json |
|----|-------------|--------|--------------|--------------|

### `step_executions`
Stores step execution history.

| id | workflow_instance_id | step_id | status | started_at | finished_at | output_json |
|----|----------------------|---------|--------|------------|-------------|-------------|

### `workflow_events`
Optional event / audit log.

| id | workflow_instance_id | event_type | payload | created_at |
|----|----------------------|------------|---------|------------|

---

## Simple Pause / Resume Example

Suppose the workflow has five steps:

```text
1. Validate file
2. Clean data
3. Enrich data
4. Databricks aggregation
5. Generate report
```

If the user pauses after step 2, then the database stores:

```text
workflow status = PAUSED
current_step = enrich_data
completed_steps = [validate_file, clean_data]
```

When resumed:

```text
skip validate_file
skip clean_data
run enrich_data
continue to step 4 and 5
```

That is the essence of resume support.

---

## Parallel Execution

If you want to sound even stronger, mention that DAGs also support parallel branches.

```text
          Validate Input
                │
                ▼
         ┌──────────────┐
         │ Split Branch │
         └──────┬───────┘
                │
      ┌─────────┴─────────┐
      ▼                   ▼
Clean Customer Data   Clean Product Data
      │                   │
      └─────────┬─────────┘
                ▼
           Merge Results
```

This is another reason DAG is better than a simple list.

---

## Failure Handling

A strong workflow design should also include error recovery.

### Good Practices

- retry transient failures
- mark permanent failures clearly
- store step-level logs
- use dead-letter queues if needed
- allow resume after issue is fixed

### Example

If a Databricks API call times out:
- retry 3 times
- if still failing, mark step as `FAILED`
- allow operator to resume later after investigating

---

## Simple TypeScript Data Model Example

```ts
type WorkflowStatus = "CREATED" | "RUNNING" | "PAUSED" | "COMPLETED" | "FAILED";
type StepStatus = "PENDING" | "RUNNING" | "SUCCESS" | "FAILED";

interface WorkflowNode {
  id: string;
  type: "task" | "decision";
  next?: string[];
  onTrue?: string;
  onFalse?: string;
}

interface WorkflowInstance {
  id: string;
  workflowId: string;
  status: WorkflowStatus;
  currentStep: string;
  context: Record<string, any>;
}
```

### Execution Idea

```ts
async function executeStep(instance: WorkflowInstance, node: WorkflowNode) {
  if (instance.status === "PAUSED") return;

  if (node.type === "task") {
    const output = await runTask(node.id, instance.context);
    instance.context = { ...instance.context, ...output };
    saveCheckpoint(instance);

    const next = node.next?.[0];
    if (next) moveTo(next);
  }

  if (node.type === "decision") {
    const result = evaluateCondition(node, instance.context);
    const next = result ? node.onTrue : node.onFalse;
    moveTo(next);
  }
}
```

---

## 90-Second Spoken Version

> I would design the workflow system as a configurable workflow engine using a **Directed Acyclic Graph**, where each node represents a transformation step, validation step, decision node, or an external task like a Databricks job. I would store the workflow definition in JSON or a database so that the workflow can be changed without hardcoding logic.
>
> For execution, I would separate the system into an **orchestrator** and **workers**. The orchestrator would manage workflow state, evaluate branching conditions, and decide the next executable step, while workers would execute the actual transformations.
>
> I would use a **state machine** to manage workflow lifecycle, with states like running, paused, completed, and failed. For pause and resume, I would persist checkpoints after every step, including current step, outputs, and workflow context, so the engine can continue from the last successful checkpoint instead of restarting.
>
> For conditional paths, I would use decision nodes that evaluate workflow context, such as file size or validation results, and then route execution to the appropriate next node. The main patterns I would use are DAG, state machine, command pattern, strategy pattern, and queue-based asynchronous processing. This design is flexible, scalable, and reliable for complex data workflows.

---

## Quick Revision Formula

```text
Workflow = DAG + State Machine + Checkpoints + Decision Nodes + Queue Workers
```

### Three Key Lines to Remember

```text
A DAG is used because workflows need branching, optional paths, and parallel steps.
```

```text
Pause/resume works because workflow state is persisted after every step.
```

```text
Conditional branching is handled through decision nodes evaluating workflow context.
```

---

## Why This Answer Scores High

It directly addresses:

- different transformation steps ✅
- conditional paths ✅
- pause/resume ✅
- data structures ✅
- patterns ✅
- scalability ✅
- Databricks relevance ✅

That is exactly the jump from a generic answer to a **10/10 interview answer**.
