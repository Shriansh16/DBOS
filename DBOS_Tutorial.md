# DBOS Tutorial: A Beginner's Guide to Durable Workflows

## Table of Contents
1. [Introduction](#introduction)
2. [What is DBOS?](#what-is-dbos)
3. [Why Use DBOS?](#why-use-dbos)
4. [Core Concepts](#core-concepts)
5. [Setting Up Your Environment](#setting-up-your-environment)
6. [Your First DBOS Program](#your-first-dbos-program)
7. [Understanding Workflows and Steps](#understanding-workflows-and-steps)
8. [Durable Execution in Action](#durable-execution-in-action)
9. [Retry Mechanisms](#retry-mechanisms)
10. [Queues and Parallelism](#queues-and-parallelism)
11. [How DBOS Works Under the Hood](#how-dbos-works-under-the-hood)
12. [Common Use Cases](#common-use-cases)
13. [Best Practices](#best-practices)
14. [Next Steps](#next-steps)

---

## Introduction

Welcome to DBOS! This tutorial will guide you through everything you need to know to get started with DBOS, from basic concepts to building your first durable application.

**What you'll learn:**
- What DBOS is and why it's useful
- How to set up DBOS in your Python environment
- How to create durable workflows and steps
- How to use queues for parallel execution
- How DBOS handles failures and recovery

**Prerequisites:**
- Basic Python knowledge
- Familiarity with functions and decorators
- A computer with Python 3.7+ installed

---

## What is DBOS?

**DBOS** (Database Operating System) is a lightweight library that helps you write **durable workflows** built on top of PostgreSQL. In simple terms, it makes your code **fault-tolerant**—meaning your programs can survive crashes, failures, and restarts without losing progress or duplicating work.

### The Problem DBOS Solves

Imagine you're building an e-commerce checkout system that needs to:
1. Validate payment
2. Check inventory
3. Ship the order
4. Notify the customer

What happens if your server crashes after step 1 but before step 2? The customer has been charged, but the order might never ship. Traditional error handling requires complex retry logic, database checkpoints, and careful state management.

DBOS solves this by automatically checkpointing your program's state to a database. If anything fails, DBOS can recover from exactly where you left off.

### Key Characteristics

- **Lightweight**: Just a library you install—no external orchestration server needed
- **Postgres-backed**: Uses PostgreSQL (or SQLite for development) to store state
- **Automatic recovery**: Handles failures and restarts automatically
- **Simple API**: Just add decorators to your functions

---

## Why Use DBOS?

### 1. **Reliability Without Complexity**

Instead of writing complex retry logic and manual checkpoints, you just annotate your functions. DBOS handles the rest.

### 2. **Perfect for Long-Running Tasks**

DBOS excels at tasks that take minutes, hours, or even days to complete:
- AI agents that need multiple iterations
- Data pipelines processing large datasets
- Payment processing workflows
- Background jobs that must complete reliably

### 3. **Built-in Observability**

DBOS tracks every workflow and step, making it easy to see what's running, what completed, and what failed.

### 4. **Distributed by Default**

DBOS naturally works in distributed environments. Multiple servers can share the same database, and workflows automatically recover on any available server.

---

## Core Concepts

Before diving into code, let's understand the fundamental concepts:

### Workflows

A **workflow** is a function that orchestrates multiple operations. It's the top-level unit of work that DBOS tracks and can recover.

**Characteristics:**
- Must be deterministic (same inputs → same execution path)
- Can contain multiple steps
- Automatically checkpointed to the database

### Steps

A **step** is an individual operation within a workflow. Steps are:
- **Idempotent**: Safe to retry multiple times
- **Checkpointed**: Their outputs are saved to the database
- **Recoverable**: If a workflow fails, steps that completed don't re-run

### Checkpoints

A **checkpoint** is a saved snapshot of your workflow's state. DBOS automatically creates checkpoints:
- When a workflow starts (saves inputs)
- After each step completes (saves step outputs)
- When a workflow finishes (saves final result)

### Recovery

When a failure occurs, DBOS:
1. Detects interrupted workflows
2. Restarts them with saved inputs
3. Skips steps that already completed (uses saved outputs)
4. Continues from the first incomplete step

---

## Setting Up Your Environment

Let's get DBOS installed and ready to use.

### Step 1: Create a Project Folder

Create a new folder for your DBOS project:

```bash
mkdir dbos-tutorial
cd dbos-tutorial
```

### Step 2: Create a Virtual Environment

**On macOS or Linux:**
```bash
python3 -m venv .venv
source .venv/bin/activate
```

**On Windows (PowerShell):**
```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

**On Windows (cmd):**
```cmd
python -m venv .venv
.venv\Scripts\activate.bat
```

### Step 3: Install DBOS

```bash
pip install dbos
```

That's it! DBOS is now installed. For this tutorial, we'll use SQLite (the default), which requires no additional setup. For production, you'd connect to PostgreSQL.

---

## Your First DBOS Program

Let's create a simple program to see DBOS in action.

### Example 1: Basic Workflow with Steps

Create a file called `main.py`:

```python
import os
from dbos import DBOS, DBOSConfig

# Configure DBOS
config: DBOSConfig = {
    "name": "dbos-tutorial",
    "system_database_url": os.environ.get("DBOS_SYSTEM_DATABASE_URL"),
}
DBOS(config=config)

# Define a step
@DBOS.step()
def step_one():
    print("Step one completed!")
    return "Result from step one"

# Define another step
@DBOS.step()
def step_two():
    print("Step two completed!")
    return "Result from step two"

# Define a workflow that uses the steps
@DBOS.workflow()
def my_first_workflow():
    result1 = step_one()
    result2 = step_two()
    print(f"Workflow completed! Got: {result1} and {result2}")
    return "Workflow finished successfully"

# Run the program
if __name__ == "__main__":
    DBOS.launch()
    my_first_workflow()
```

### Running Your First Program

Run it with:
```bash
python main.py
```

You should see output like:
```
15:41:06 [    INFO] (dbos:_dbos.py:534) DBOS launched!
Step one completed!
Step two completed!
Workflow completed! Got: Result from step one and Result from step two
```

### What Just Happened?

1. **`DBOS(config=config)`**: Initializes DBOS with your configuration
2. **`@DBOS.step()`**: Marks functions as steps that will be checkpointed
3. **`@DBOS.workflow()`**: Marks the main function as a workflow
4. **`DBOS.launch()`**: Starts DBOS and connects to the database
5. The workflow ran both steps in sequence

At this point, DBOS has saved checkpoints of your workflow and both steps to its database (SQLite by default).

---

## Understanding Workflows and Steps

### The Decorators Explained

#### `@DBOS.workflow()`

This decorator marks a function as a workflow. Workflows:
- Are the top-level unit of execution
- Can call steps and other functions
- Are automatically checkpointed
- Must be deterministic (more on this later)

#### `@DBOS.step()`

This decorator marks a function as a step. Steps:
- Are individual operations within a workflow
- Are checkpointed after completion
- Should be idempotent (safe to retry)
- Can return values that are saved

### Important Rules

#### 1. Workflows Must Be Deterministic

A deterministic function always produces the same output for the same input. This is crucial for recovery.

**❌ Bad (Non-deterministic):**
```python
@DBOS.workflow()
def bad_workflow():
    current_time = time.time()  # Different each time!
    random_num = random.randint(1, 10)  # Different each time!
    step_one()
```

**✅ Good (Deterministic):**
```python
@DBOS.workflow()
def good_workflow():
    step_one()  # Always calls step_one()
    step_two()  # Always calls step_two() in this order
```

If you need non-deterministic operations (like getting the current time or calling an API), do them **inside a step**, not in the workflow function itself.

#### 2. Steps Should Be Idempotent

An idempotent operation can be safely repeated without side effects.

**✅ Good (Idempotent):**
```python
@DBOS.step()
def process_payment(order_id):
    # Check if already processed
    if payment_already_processed(order_id):
        return get_existing_result(order_id)
    # Process payment
    result = charge_customer(order_id)
    save_payment_result(order_id, result)
    return result
```

**❌ Bad (Not Idempotent):**
```python
@DBOS.step()
def bad_step():
    send_email()  # Sends email every time - not idempotent!
```

---

## Durable Execution in Action

Let's see DBOS's recovery in action with a real example.

### Example 2: HTTP Server with Recovery

Create a new `main.py`:

```python
import os
import uvicorn
from dbos import DBOS, DBOSConfig
from fastapi import FastAPI

app = FastAPI()

# Configure DBOS
config: DBOSConfig = {
    "name": "dbos-tutorial",
    "system_database_url": os.environ.get("DBOS_SYSTEM_DATABASE_URL"),
}
DBOS(config=config)

@DBOS.step()
def step_one():
    print("Step one completed!")
    return "Step 1 done"

@DBOS.step()
def step_two():
    print("Step two completed!")
    return "Step 2 done"

@app.get("/")
@DBOS.workflow()
def dbos_workflow():
    result1 = step_one()
    
    # Simulate some work that takes time
    for i in range(5):
        print(f"Working... {i+1}/5 (Press Ctrl+C to interrupt!)")
        DBOS.sleep(1)  # DBOS-aware sleep
    
    result2 = step_two()
    return {"status": "complete", "step1": result1, "step2": result2}

if __name__ == "__main__":
    DBOS.launch()
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Install FastAPI

```bash
pip install 'fastapi[standard]'
```

### Test the Recovery

1. **Start the server:**
   ```bash
   python main.py
   ```

2. **Visit** http://localhost:8000 in your browser or use curl:
   ```bash
   curl http://localhost:8000
   ```

3. **Watch the terminal.** You should see:
   ```
   Step one completed!
   Working... 1/5 (Press Ctrl+C to interrupt!)
   Working... 2/5 (Press Ctrl+C to interrupt!)
   ```

4. **While it's running, press Ctrl+C** to stop the server (you might need to press it a few times).

5. **Restart the server:**
   ```bash
   python main.py
   ```

6. **Visit** http://localhost:8000 again.

### What You Should See

After restarting, you should see:
```
Working... 3/5 (Press Ctrl+C to interrupt!)
Working... 4/5 (Press Ctrl+C to interrupt!)
Working... 5/5 (Press Ctrl+C to interrupt!)
Step two completed!
```

**Notice:** Step one didn't run again! DBOS:
1. Detected the interrupted workflow
2. Restored the saved state
3. Skipped step_one (it already completed)
4. Continued from the loop (which was in progress)
5. Completed step_two

This is **durable execution** in action!

---

## Retry Mechanisms

DBOS supports retrying failing steps so transient errors (timeouts, network blips, etc.) can be retried automatically.

### Basic retry usage

Mark a step to allow retries and set the maximum attempts (including the initial try):

```python
@DBOS.step(retries_allowed=True, max_attempts=3)
def unreliable_api_call(data: str):
    # This will retry up to 3 times if it raises an exception
    response = requests.post("https://api.example.com/data", json={"data": data})
    response.raise_for_status()
    return response.json()
```

### When to use retries

- Use retries for transient or recoverable failures (timeouts, connection errors, 5xx server errors).
- Do not rely on retries for permanent client errors (4xx) — handle those explicitly.

### Best practices

- Idempotency: Steps that can be retried must be idempotent or safe to re-run. For example, write operations should detect duplicates or be guarded by a check.
- Limit attempts: Keep `max_attempts` small for quick failure feedback, or larger when external systems are known to be flaky.
- Inspect and fail fast on permanent errors: re-raise non-retriable exceptions so the workflow fails instead of retrying.

### Example: Upload with retries

```python
@DBOS.step(retries_allowed=True, max_attempts=5)
def upload_to_s3(file_path: str, bucket: str, key: str):
    try:
        s3_client.upload_file(file_path, bucket, key)
        DBOS.logger.info(f"Successfully uploaded {key}")
        return {"bucket": bucket, "key": key, "status": "success"}
    except Exception as e:
        DBOS.logger.warning(f"Upload failed: {e}, will retry")
        raise  # Re-raise to trigger DBOS retry

@DBOS.workflow()
def file_processing_workflow(files: list[str]):
    results = []
    for file_path in files:
        # This upload will be retried up to 5 times on failure
        result = upload_to_s3(file_path, "my-bucket", f"uploads/{file_path}")
        results.append(result)
    return results
```

### Important notes

- If a step exhausts its retries and still fails, the workflow is marked as failed.
- Only successful step executions are checkpointed; failed attempts do not create checkpoints.
- During recovery, DBOS will skip steps that already completed successfully and continue from the first incomplete step.

## Queues and Parallelism

DBOS queues let you run multiple operations concurrently and distribute work across servers.

### What Are Queues?

A **queue** is a collection of tasks waiting to be executed. You can:
- Enqueue functions to run asynchronously
- Control how many run concurrently
- Wait for results
- Distribute work across multiple servers

### Example 3: Parallel Execution with Queues

Replace your `main.py` with:

```python
import os
import time
import uvicorn
from dbos import DBOS, DBOSConfig, Queue
from fastapi import FastAPI

app = FastAPI()

config: DBOSConfig = {
    "name": "dbos-tutorial",
    "system_database_url": os.environ.get("DBOS_SYSTEM_DATABASE_URL"),
}
DBOS(config=config)

# Create a queue
queue = Queue("example-queue")

@DBOS.step()
def process_item(item_id: int):
    """Simulate processing an item (takes 2 seconds)"""
    print(f"Processing item {item_id}...")
    time.sleep(2)
    result = f"Item {item_id} processed"
    print(f"✓ {result}")
    return result

@app.get("/")
@DBOS.workflow()
def parallel_workflow():
    print("Enqueueing 10 items for processing...")
    
    # Enqueue 10 items
    handles = []
    for i in range(10):
        handle = queue.enqueue(process_item, i)
        handles.append(handle)
    
    print("Waiting for all items to complete...")
    
    # Wait for all results
    results = []
    for handle in handles:
        result = handle.get_result()  # Blocks until this item completes
        results.append(result)
    
    print(f"✓ Successfully processed {len(results)} items")
    return {
        "status": "complete",
        "items_processed": len(results),
        "results": results
    }

if __name__ == "__main__":
    DBOS.launch()
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Understanding the Code

1. **`Queue("example-queue")`**: Creates a named queue
2. **`queue.enqueue(process_item, i)`**: Adds a task to the queue
   - Returns a handle you can use to track the task
   - The task runs asynchronously (doesn't block)
3. **`handle.get_result()`**: Waits for the task to complete and returns its result

### Run It

1. Start the server: `python main.py`
2. Visit http://localhost:8000
3. Watch the terminal

### What You Should See

Even though each item takes 2 seconds to process, all 10 items should complete in roughly 2 seconds total because they run **in parallel**:

```
Enqueueing 10 items for processing...
Waiting for all items to complete...
Processing item 0...
Processing item 1...
Processing item 2...
...
Processing item 9...
✓ Item 0 processed
✓ Item 1 processed
...
✓ Item 9 processed
✓ Successfully processed 10 items
```

### Queue Benefits

- **Parallelism**: Run many tasks concurrently
- **Flow Control**: Limit how many tasks run at once
- **Distributed Execution**: Tasks can run on any server connected to the same database
- **Durability**: Even if a server crashes, queued tasks are saved and will complete

---

## How DBOS Works Under the Hood

Understanding how DBOS works helps you use it effectively.

### Architecture

```
┌─────────────────┐
│  Your Python    │
│  Application    │
│                 │
│  @DBOS.workflow │
│  @DBOS.step     │
└────────┬────────┘
         │
         │ DBOS Library
         │ (checkpointing)
         │
         ▼
┌─────────────────┐
│   PostgreSQL    │
│   (or SQLite)   │
│                 │
│  - Workflows    │
│  - Steps        │
│  - Queue state  │
└─────────────────┘
```

**Key Points:**
- DBOS is just a library—no separate server needed
- All state is stored in the database
- Your application code stays the same

### Checkpointing Process

1. **Workflow starts**: DBOS saves the workflow inputs to the database
2. **Step executes**: The step function runs
3. **Step completes**: DBOS saves the step's output to the database
4. **Next step**: Process repeats
5. **Workflow completes**: Final result is saved

### Recovery Process

When a failure occurs:

1. **Detection**: DBOS scans for incomplete (PENDING) workflows
2. **Restart**: Each interrupted workflow is restarted with saved inputs
3. **Skip completed steps**: If a step's output exists in the database, DBOS returns that instead of re-running
4. **Continue**: Execution proceeds from the first step without a checkpoint

### Example: Recovery Flow

```
Original Execution:
[Workflow Start] → [Step 1 ✓] → [Step 2 ✓] → [Step 3 ✗ CRASH]

After Restart:
[Workflow Start] → [Step 1: Use saved output] → [Step 2: Use saved output] → [Step 3: Execute] → [Complete]
```

### Determinism Requirement

Workflows must be deterministic because DBOS might re-execute them during recovery. If the workflow makes different decisions each time, recovery could produce incorrect results.

**Why determinism matters:**
- During recovery, DBOS re-runs the workflow function
- It skips steps that already completed
- But the workflow logic itself runs again
- If it's non-deterministic, it might take a different path

**Solution**: Put non-deterministic operations in steps, which are only executed once.

---

## Common Use Cases

### 1. AI Agents

AI agents often need multiple iterations and tool calls. DBOS ensures they can recover from failures:

```python
@DBOS.workflow()
def agent_workflow(user_query: str, max_iterations: int = 10):
    context = []
    
    for i in range(max_iterations):
        # Research step
        research_result = research_step(user_query, context)
        context.append(research_result)
        
        # Decide if we should continue
        should_continue = decide_next_action(research_result, context)
        if not should_continue:
            break
        
        # Generate next query
        user_query = generate_next_query(research_result)
    
    # Synthesize final answer
    return synthesize_answer(context)

@DBOS.step()
def research_step(query: str, context: list):
    # Call AI API, search web, etc.
    return perform_research(query, context)
```

### 2. Data Pipelines

Process large datasets reliably:

```python
@DBOS.workflow()
def data_pipeline_workflow(source_url: str):
    # Download data
    raw_data = download_data(source_url)
    
    # Process in parallel
    queue = Queue("processing-queue")
    handles = []
    
    for chunk in split_into_chunks(raw_data):
        handle = queue.enqueue(process_chunk, chunk)
        handles.append(handle)
    
    # Collect results
    results = [h.get_result() for h in handles]
    
    # Aggregate
    final_result = aggregate_results(results)
    
    # Upload
    upload_result(final_result)
    return final_result
```

### 3. Payment Processing

Ensure payments complete reliably:

```python
@DBOS.workflow()
def payment_workflow(order_id: str, amount: float):
    # Validate payment
    payment_result = validate_payment(order_id, amount)
    
    # Reserve inventory
    inventory_result = reserve_inventory(order_id)
    
    # Process payment
    charge_result = charge_customer(order_id, amount)
    
    # Fulfill order
    fulfillment_result = fulfill_order(order_id)
    
    # Send confirmation
    send_confirmation(order_id)
    
    return {
        "order_id": order_id,
        "status": "completed",
        "payment": charge_result
    }
```

### 4. Background Jobs

Long-running background tasks:

```python
@DBOS.workflow()
def background_job_workflow(job_id: str):
    # Step 1: Prepare
    preparation_result = prepare_job(job_id)
    
    # Step 2: Execute (might take hours)
    execution_result = execute_long_running_job(job_id, preparation_result)
    
    # Step 3: Cleanup
    cleanup_result = cleanup_job(job_id, execution_result)
    
    # Step 4: Notify
    notify_completion(job_id, execution_result)
    
    return execution_result
```

---

## Best Practices

### 1. Keep Steps Focused

Each step should do one thing well:

```python
# ✅ Good: Focused steps
@DBOS.step()
def validate_input(data):
    return validate(data)

@DBOS.step()
def process_data(data):
    return process(data)

# ❌ Bad: Too much in one step
@DBOS.step()
def do_everything(data):
    validate(data)
    process(data)
    save(data)
    notify(data)
```

### 2. Avoid Large Return Values

Large outputs increase checkpoint size. Store large data externally:

```python
# ✅ Good: Return a reference
@DBOS.step()
def process_large_file(file_path: str):
    result = process_file(file_path)
    s3_url = upload_to_s3(result)  # Store in S3
    return s3_url  # Return just the URL

# ❌ Bad: Return large data
@DBOS.step()
def process_large_file(file_path: str):
    result = process_file(file_path)  # 1GB of data
    return result  # This gets checkpointed - slow!
```

### 3. Use Queues for Parallelism

Don't call steps sequentially when they can run in parallel:

```python
# ✅ Good: Parallel execution
@DBOS.workflow()
def parallel_workflow():
    queue = Queue("work-queue")
    handles = [queue.enqueue(process_item, i) for i in range(100)]
    return [h.get_result() for h in handles]

# ❌ Bad: Sequential execution
@DBOS.workflow()
def sequential_workflow():
    results = []
    for i in range(100):
        results.append(process_item(i))  # One at a time!
    return results
```

### 4. Handle Errors in Steps

Steps should handle their own errors:

```python
@DBOS.step()
def robust_step(data):
    try:
        result = risky_operation(data)
        return result
    except SpecificError as e:
        # Handle known errors
        return handle_error(e)
    except Exception as e:
        # Log and re-raise unexpected errors
        log_error(e)
        raise
```

### 5. Use Meaningful Names

Name your workflows and steps clearly:

```python
# ✅ Good: Clear names
@DBOS.workflow()
def process_customer_order_workflow(order_id: str):
    validate_order(order_id)
    charge_customer(order_id)
    ship_order(order_id)

# ❌ Bad: Unclear names
@DBOS.workflow()
def workflow1(id: str):
    step1(id)
    step2(id)
```

---

## Next Steps

Congratulations! You now understand the basics of DBOS. Here's what to explore next:

### 1. **Production Setup**
   - Connect to PostgreSQL instead of SQLite
   - Set up environment variables
   - Configure connection pooling

### 2. **Advanced Features**
   - **Conductor**: Self-hosted management service for distributed deployments
   - **DBOS Cloud**: Serverless platform for DBOS applications
   - **Workflow forking**: Restart workflows from specific steps
   - **Queue configuration**: Rate limits, concurrency, priorities

### 3. **Integration**
   - Add DBOS to existing applications
   - Use DBOS Client to interact with workflows from external code
   - Integrate with web frameworks (Flask, Django, etc.)

### 4. **Monitoring and Observability**
   - View workflow dashboards
   - Monitor queue status
   - Debug failed workflows

### 5. **Language Support**
   - DBOS is also available for TypeScript, Go, and Java
   - Check out language-specific documentation

### Resources

- **Official Documentation**: https://docs.dbos.dev
- **GitHub**: https://github.com/dbos-inc/dbos
- **Examples**: Check out the DBOS Toolbox for example applications
- **Community**: Join the DBOS community for support and discussions

---

## Summary

**Key Takeaways:**

1. **DBOS** makes your code fault-tolerant by checkpointing to a database
2. **Workflows** orchestrate operations and must be deterministic
3. **Steps** are individual operations that should be idempotent
4. **Queues** enable parallel and distributed execution
5. **Recovery** happens automatically—DBOS resumes from the last completed step

**Remember:**
- Keep workflows deterministic
- Keep steps idempotent
- Use queues for parallelism
- Avoid large return values in steps
- Let DBOS handle the complexity of recovery

Happy coding with DBOS! 🚀

