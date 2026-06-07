---
tags: [concept, serverless, orchestration, workflow, state-machine, step-functions]
aliases: [Step Functions, AWS Step Functions, State Machine, SFN, Workflow Orchestration]
date: 2026-05-29
---

# AWS Step Functions

**AWS Step Functions** is a serverless **visual workflow orchestration** service. It coordinates multiple AWS services ([[AWS Lambda]], [[Containers on AWS|ECS]], [[Amazon DynamoDB]], [[Amazon SQS]], [[Amazon SNS]], etc.) into complex business workflows using **state machines** defined in JSON (Amazon States Language — ASL).

> [!IMPORTANT]
> **Core exam concept:** Step Functions is the answer for **orchestrating multi-step workflows**, **coordinating Lambda functions**, or **handling complex branching logic with retries and error handling**. "Chain multiple Lambda functions" or "visual workflow" → **Step Functions**. Do NOT have Lambda invoke Lambda directly — use Step Functions instead.

---

## Why Step Functions?

```
  WITHOUT Step Functions (Lambda calling Lambda):
  ───────────────────────────────────────────────
  ┌──────────┐    invoke    ┌──────────┐    invoke    ┌──────────┐
  │ Lambda A │────────────►│ Lambda B │────────────►│ Lambda C │
  └──────────┘             └──────────┘             └──────────┘

  Problems:
  ❌ Tight coupling (A knows about B, B knows about C)
  ❌ Error handling is your responsibility (retry logic in code)
  ❌ No visibility into workflow state
  ❌ Lambda A pays for time waiting on B
  ❌ Hard to modify flow (add steps, parallel branches)

  WITH Step Functions:
  ─────────────────────
  ┌──────────────────────────────────────────────────────────┐
  │  Step Functions State Machine                             │
  │                                                          │
  │  ┌──────────┐   ┌──────────┐   ┌──────────┐            │
  │  │ Lambda A │──►│ Lambda B │──►│ Lambda C │            │
  │  └──────────┘   └────┬─────┘   └──────────┘            │
  │                      │ fail                              │
  │                      ▼                                   │
  │                 ┌──────────┐                              │
  │                 │ Retry ×3 │──► Catch → Fallback         │
  │                 └──────────┘                              │
  └──────────────────────────────────────────────────────────┘

  Benefits:
  ✅ Decoupled — each step is independent
  ✅ Built-in retry + error handling (Retry/Catch)
  ✅ Visual debugging — see exactly where it failed
  ✅ No Lambda wasted waiting time
  ✅ Easy to modify, add steps, parallel branches
```

---

## How State Machines Work

```
  ┌────────────────────────────────────────────────────────────┐
  │                    State Machine (JSON/ASL)                  │
  │                                                            │
  │  StartAt: "ValidateOrder"                                  │
  │                                                            │
  │  ┌─────────────┐                                           │
  │  │ Validate    │ ← Task State (invoke Lambda)              │
  │  │ Order       │                                           │
  │  └──────┬──────┘                                           │
  │         │                                                  │
  │  ┌──────▼──────┐                                           │
  │  │ Check       │ ← Choice State (if/else branching)        │
  │  │ Inventory   │                                           │
  │  └──┬──────┬───┘                                           │
  │     │      │                                               │
  │  in stock  out of stock                                    │
  │     │      │                                               │
  │  ┌──▼───┐  ┌──▼──────────┐                                │
  │  │Charge│  │ Notify      │                                │
  │  │Card  │  │ Backorder   │                                │
  │  └──┬───┘  └──────┬──────┘                                │
  │     │             │                                        │
  │  ┌──▼───┐         │                                       │
  │  │ Ship │         │                                       │
  │  │Order │         │                                       │
  │  └──┬───┘         │                                       │
  │     │             │                                        │
  │  ┌──▼─────────────▼──┐                                    │
  │  │     Success       │ ← Succeed State                    │
  │  └───────────────────┘                                    │
  └────────────────────────────────────────────────────────────┘
```

---

## State Types

| State | Purpose | Example |
|---|---|---|
| **Task** | Execute work — invoke Lambda, ECS, DynamoDB, SQS, SNS, Glue, SageMaker, etc. | Call a Lambda function to process payment |
| **Choice** | Branching logic (if/else based on input) | If `order.total > 100` → apply discount |
| **Parallel** | Run multiple branches **simultaneously** | Process image + send notification at the same time |
| **Map** | Iterate over a collection (for-each loop) | Process each item in an order |
| **Wait** | Pause for a specified duration or until a timestamp | Wait 24 hours before sending reminder |
| **Pass** | Pass input to output, optionally transforming data | Add default values, restructure JSON |
| **Succeed** | End the workflow with success | Terminal state — workflow complete |
| **Fail** | End the workflow with failure (error + cause) | Terminal state — workflow failed |

### Parallel State

```
  ┌──────────────────────────────────────────────┐
  │  Parallel State                               │
  │                                               │
  │  Branch 1:          Branch 2:        Branch 3:│
  │  ┌──────────┐      ┌──────────┐    ┌────────┐│
  │  │ Resize   │      │ Extract  │    │ Send   ││
  │  │ Image    │      │ Metadata │    │ Notif  ││
  │  │ (Lambda) │      │ (Lambda) │    │ (SNS)  ││
  │  └──────────┘      └──────────┘    └────────┘│
  │       │                 │               │     │
  │       └─────────────────┼───────────────┘     │
  │                         ▼                     │
  │                    Aggregate                   │
  │                    Results                     │
  └──────────────────────────────────────────────┘

  All branches run concurrently
  All must succeed for the Parallel state to succeed
  Results aggregated as an array
```

### Map State

```
  Input: { "items": ["A", "B", "C", "D"] }

  ┌──────────────────────────────────────────────┐
  │  Map State (Inline or Distributed)            │
  │                                               │
  │  Iteration 1: Process "A" → ┌──────────┐    │
  │  Iteration 2: Process "B" → │  Lambda   │   │
  │  Iteration 3: Process "C" → │ (per item)│   │
  │  Iteration 4: Process "D" → └──────────┘    │
  │                                               │
  │  Inline Map:      Up to 40 concurrent         │
  │  Distributed Map: Up to 10,000 concurrent     │
  │                   (for large-scale processing) │
  └──────────────────────────────────────────────┘
```

> [!TIP]
> **Exam Pattern:** "Process thousands of items in parallel" → **Distributed Map** state. "Process a few items in a list" → **Inline Map** state. "Run multiple independent tasks simultaneously" → **Parallel** state.

---

## Standard vs Express Workflows

| Feature | Standard Workflows | Express Workflows |
|---|---|---|
| **Max duration** | **1 year** | **5 minutes** |
| **Execution guarantee** | **Exactly-once** | **At-least-once** |
| **Execution start rate** | 2,000/sec | 100,000/sec |
| **State transition rate** | 4,000/sec per account | Nearly unlimited |
| **Pricing** | Per **state transition** ($0.025 per 1,000) | Per **execution** (duration + memory) |
| **Execution history** | 90-day visual history in console | CloudWatch Logs only |
| **Visual debugging** | ✅ Full step-by-step replay | ❌ Limited |
| **Use case** | Order processing, ETL, human approval, long-running | IoT data, streaming, high-volume event processing |

### Express Workflow Sub-Types

| Sub-Type | Invocation | Behavior |
|---|---|---|
| **Synchronous Express** | Caller waits for result | Returns output to caller. For API Gateway → Step Functions. |
| **Asynchronous Express** | Fire-and-forget | Results in CloudWatch Logs. For high-volume event processing. |

> [!WARNING]
> **Exam trap:** "Long-running workflow" or "human approval loop" or "exactly-once" → **Standard**. "High-volume, short-duration event processing" or "IoT data" → **Express**. "Call Step Functions from API Gateway and return result" → **Synchronous Express**.

---

## Error Handling: Retry & Catch

```
  ┌──────────────────────────────────────────────────────────┐
  │  Task State with Error Handling:                          │
  │                                                          │
  │  "ProcessPayment": {                                     │
  │    "Type": "Task",                                       │
  │    "Resource": "arn:aws:lambda:...:processPayment",      │
  │                                                          │
  │    "Retry": [                  ← RETRY FIRST             │
  │      {                                                   │
  │        "ErrorEquals": ["ServiceUnavailable"],            │
  │        "IntervalSeconds": 2,   ← wait 2s before retry   │
  │        "MaxAttempts": 3,       ← retry up to 3 times    │
  │        "BackoffRate": 2.0      ← exponential: 2s,4s,8s  │
  │      }                                                   │
  │    ],                                                    │
  │                                                          │
  │    "Catch": [                  ← CATCH IF RETRIES FAIL   │
  │      {                                                   │
  │        "ErrorEquals": ["States.ALL"],                    │
  │        "Next": "HandleFailure" ← go to fallback state   │
  │      }                                                   │
  │    ]                                                     │
  │  }                                                       │
  └──────────────────────────────────────────────────────────┘
```

### Error Types

| Error | Description |
|---|---|
| **States.ALL** | Catch-all for any error |
| **States.Timeout** | Task took longer than TimeoutSeconds |
| **States.TaskFailed** | Task execution failed |
| **States.Permissions** | Insufficient IAM permissions |
| **Custom errors** | Application-specific errors thrown by Lambda |

> [!TIP]
> **Exam Pattern:** "Handle transient errors in serverless workflow" → **Step Functions Retry** with exponential backoff. "Fallback on permanent failure" → **Step Functions Catch** → route to error-handling state. This is preferred over writing retry logic inside Lambda.

---

## Service Integrations

Step Functions can integrate with **200+ AWS services** directly:

### Integration Patterns

| Pattern | Description | Use Case |
|---|---|---|
| **Request-Response** | Call service, get response, move to next state | Invoke Lambda, query DynamoDB |
| **Run a Job (.sync)** | Start a job, wait for it to complete | ECS task, Glue job, SageMaker training |
| **Wait for Callback (.waitForTaskToken)** | Pause workflow, resume when external system calls back | Human approval, external API callback |

```
  Wait for Callback Pattern (Human Approval):
  ────────────────────────────────────────────

  ┌────────────┐     ┌────────────┐     ┌────────────┐
  │ Submit     │────►│ Wait for   │────►│ Process    │
  │ Request    │     │ Approval   │     │ Approved   │
  │ (Lambda)   │     │ (SQS +     │     │ Request    │
  │            │     │ callback)  │     │ (Lambda)   │
  └────────────┘     └─────┬──────┘     └────────────┘
                           │
                    Sends task token
                    to approver via
                    email/Slack
                           │
                    Approver clicks
                    "Approve" → calls
                    SendTaskSuccess API
                    with task token
```

> [!TIP]
> **Exam Pattern:** "Human approval in a workflow" or "wait for external callback" → **Step Functions with `.waitForTaskToken`** pattern. "Run ECS task and wait for completion" → **Run a Job (.sync)** pattern.

---

## Common Architecture Patterns

### Pattern 1: Order Processing Pipeline

```
  API Gateway → Step Functions (Standard):
  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  ┌────────┐
  │Validate │─►│ Reserve  │─►│ Charge   │─►│ Ship   │─►│ Notify │
  │Order    │  │Inventory │  │ Card     │  │ Order  │  │ User   │
  │(Lambda) │  │(DynamoDB)│  │(Lambda)  │  │(ECS)   │  │(SNS)   │
  └─────────┘  └──────────┘  └──────────┘  └────────┘  └────────┘
```

### Pattern 2: ETL / Data Processing

```
  EventBridge (schedule) → Step Functions:
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Extract  │─►│Transform │─►│  Load    │
  │ (Lambda) │  │(Glue Job │  │ (Lambda) │
  │          │  │ .sync)   │  │ → S3/    │
  │          │  │          │  │ Redshift │
  └──────────┘  └──────────┘  └──────────┘
```

### Pattern 3: Saga Pattern (Distributed Transactions)

```
  ┌─────────┐  ┌──────────┐  ┌──────────┐
  │ Book    │─►│ Book     │─►│ Charge   │──► Success
  │ Flight  │  │ Hotel    │  │ Card     │
  └────┬────┘  └────┬─────┘  └────┬─────┘
       │ fail       │ fail        │ fail
       ▼            ▼             ▼
  ┌─────────┐  ┌──────────┐  ┌──────────┐
  │ Cancel  │◄─│ Cancel   │◄─│ Refund   │
  │ Flight  │  │ Hotel    │  │ Card     │
  └─────────┘  └──────────┘  └──────────┘

  Each step has a compensating action (undo)
  Step Functions Catch routes to compensating steps on failure
```

---

## Step Functions vs Other Orchestration

| Feature | Step Functions | SQS + Lambda | EventBridge |
|---|---|---|---|
| **Workflow type** | Complex, multi-step, branching | Simple queue → process | Event routing (1-to-many) |
| **Visual debugging** | ✅ Full execution history | ❌ | ❌ |
| **Error handling** | Built-in Retry + Catch | Manual (DLQ + code) | Manual |
| **Human approval** | ✅ (waitForTaskToken) | ❌ | ❌ |
| **Long-running** | Up to 1 year | ❌ (15 min Lambda) | ❌ |
| **Cost** | Per state transition | Per message + Lambda | Per event |

---

## Key Step Functions Limits

| Parameter | Value |
|---|---|
| **Standard max duration** | **1 year** |
| **Express max duration** | **5 minutes** |
| **Standard execution start rate** | 2,000/sec |
| **Express execution start rate** | 100,000/sec |
| **Max input/output size** | **256 KB** per state |
| **Max execution history events** | 25,000 events |
| **State machine definition** | Max **1 MB** (ASL JSON) |
| **Max open executions** | 1,000,000 (Standard) |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Step Functions:**
> - "Orchestrate multiple Lambda functions" → **Step Functions** (NOT Lambda calling Lambda)
> - "Visual workflow" or "visual debugging" → **Step Functions**
> - "Complex branching logic with retries" → **Step Functions Retry + Catch**
> - "Human approval workflow" → **Step Functions waitForTaskToken**
> - "Long-running process (> 15 min)" → **Step Functions Standard** (up to 1 year)
> - "Process thousands of items in parallel" → **Step Functions Distributed Map**
> - "Saga pattern / distributed transactions" → **Step Functions** with compensating steps
> - "High-volume, short event processing" → **Step Functions Express**
> - "ETL pipeline coordination" → **Step Functions + Glue (.sync)**
> - "Call Step Functions from API" → **API Gateway + Synchronous Express**
>
> **Key facts:**
> - Standard: 1 year max, exactly-once, per state transition pricing.
> - Express: 5 minutes max, at-least-once, per execution pricing.
> - Retry: exponential backoff (IntervalSeconds × BackoffRate^attempt).
> - Catch: fallback to another state after all retries fail.
> - Integration patterns: Request-Response, Run a Job (.sync), Wait for Callback.
> - Max input/output per state: 256 KB. Use S3 for larger payloads.
> - States: Task, Choice, Parallel, Map, Wait, Pass, Succeed, Fail.
> - Distributed Map: up to 10,000 concurrent iterations (for large-scale).
> - Step Functions can directly integrate with 200+ AWS services — no Lambda needed for simple tasks.
