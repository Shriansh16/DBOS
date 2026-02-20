# DBOS, Workers, and Queues - Complete Workflow Explanation

## Overview

The Predictive Audience system uses **DBOS** (Database-Oriented System) for workflow orchestration. DBOS provides a queue-based architecture where:
- **DBOS** = The orchestration framework that manages workflow execution
- **Queues** = Message queues that hold pending workflows
- **Workers** = Services that dequeue and execute workflows

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    FastAPI API Server                        │
│  (pa/api/pipeline_endpoints.py)                             │
│                                                              │
│  POST /pipeline/run/{client_id}                             │
│    ↓                                                         │
│  dbos_client.enqueue_async()                                │
│    ↓                                                         │
│  Creates workflow entry in PostgreSQL                       │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       │ (Workflow enqueued to queue)
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              PostgreSQL (DBOS System Database)              │
│                                                              │
│  workflow_status table:                                      │
│  - workflow_uuid (unique ID)                                │
│  - workflow_name (e.g., "Predictive_Audience_pipeline")     │
│  - status (PENDING → RUNNING → COMPLETED/FAILED)            │
│  - queue_name (e.g., "pa-workflows")                        │
│  - parameters (serialized workflow arguments)                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       │ (Worker polls queue)
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  DBOS Worker Service                         │
│  (pa/workers/dbos_worker.py)                                │
│                                                              │
│  1. Initializes DBOS                                         │
│  2. Registers queue: "pa-workflows"                         │
│  3. Imports workflows (auto-registers them)                 │
│  4. Launches DBOS runtime                                    │
│  5. Continuously dequeues workflows                          │
│  6. Executes workflows step-by-step                         │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       │ (Executes workflow steps)
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Workflow Execution (DBOS Steps)                 │
│  (pa/flows/channel_pipeline.py)                             │
│                                                              │
│  @dbos.workflow("Predictive_Audience_pipeline")             │
│    ↓                                                         │
│  Step 1: @dbos.step("compute_email_metrics")                │
│  Step 2: @dbos.step("generate_email_features")              │
│  Step 3: @dbos.step("segment_email_features")               │
│  Step 4: @dbos.step("calculate_ltv")                         │
│                                                              │
│  Each step:                                                  │
│  - Checkpoints state to PostgreSQL                          │
│  - Supports retries (retries_allowed=2)                     │
│  - Updates workflow_status table                            │
└──────────────────────────────────────────────────────────────┘
```

---

## Components Explained

### 1. DBOS (Database-Oriented System)

**What it is:**
- A workflow orchestration framework that uses PostgreSQL as its system database
- Provides checkpointing, retries, and state management for workflows
- Manages workflow execution lifecycle (PENDING → RUNNING → COMPLETED/FAILED)

**Key Features:**
- **Checkpointing**: Each workflow step saves its state to PostgreSQL
- **Retries**: Failed steps can automatically retry (configurable)
- **Queue Management**: Distributes workflows across multiple workers
- **State Persistence**: Workflow state survives worker restarts

**Configuration:**
- System database: PostgreSQL (configured via `DBOS_DATABASE_URL`)
- Application name: `pa-api-{environment}` (e.g., `pa-api-development`)
- Queue name: `pa-workflows` (default, configurable via `DBOS_QUEUE_NAME`)

**Files:**
- `pa/db/dbos_config.py` - DBOS configuration management
- `pa/flows/dbos_app.py` - DBOS instance initialization
- `dbos-config.yaml` - DBOS configuration file (optional)

---

### 2. Queues

**What it is:**
- A named queue that holds pending workflows
- Implemented using PostgreSQL tables (managed by DBOS)
- Workers poll the queue to find work

**How it works:**
1. **Queue Definition**: Worker defines the queue name (e.g., `"pa-workflows"`)
   ```python
   Queue("pa-workflows")  # Registers queue with DBOS
   ```

2. **Enqueueing**: API enqueues workflows to the queue
   ```python
   await dbos_client.enqueue_async(
       options={"queue_name": "pa-workflows", "workflow_name": "Predictive_Audience_pipeline"},
       workflow_function,
       client_id=70132,
       ...
   )
   ```

3. **Dequeueing**: Worker automatically dequeues workflows from the queue
   - DBOS runtime polls the queue
   - When a workflow is found, it's dequeued and executed
   - Multiple workers can process the same queue (load balancing)

**Queue Name:**
- Default: `"pa-workflows"`
- Configurable via `DBOS_QUEUE_NAME` environment variable
- Must match between API (enqueueing) and Worker (dequeueing)

**Storage:**
- Queue state stored in PostgreSQL `workflow_status` table
- Each workflow has a `queue_name` column
- Status: `PENDING` (in queue) → `RUNNING` (being processed) → `COMPLETED`/`FAILED`

---

### 3. Workers

**What it is:**
- A long-running service that processes workflows from the queue
- Runs independently from the API server
- Can scale horizontally (multiple workers = more parallelism)

**Worker Lifecycle:**

1. **Initialization** (`pa/workers/dbos_worker.py`):
   ```python
   # 1. Load configuration
   config = get_dbos_config()
   
   # 2. Initialize DBOS instance
   DBOS(config=config)
   
   # 3. Define the queue (registers it)
   Queue("pa-workflows")
   
   # 4. Import workflows (they auto-register via @dbos.workflow decorators)
   from pa.flows.channel_pipeline import client_pipeline_workflow
   
   # 5. Launch DBOS runtime (starts background threads)
   DBOS.launch()
   
   # 6. Wait indefinitely (DBOS handles dequeuing automatically)
   threading.Event().wait()
   ```

2. **Workflow Processing**:
   - DBOS runtime polls the queue in the background
   - When a workflow is found, it:
     - Updates status to `RUNNING`
     - Executes workflow steps sequentially
     - Checkpoints state after each step
     - Updates status to `COMPLETED` or `FAILED`

3. **Step Execution**:
   - Each `@dbos.step` is executed in order
   - State is checkpointed to PostgreSQL after each step
   - If a step fails, it can retry (up to `retries_allowed` times)
   - If all retries fail, workflow status becomes `FAILED`

**Running a Worker:**
```bash
# From PredictiveAudience root directory
python -m pa.workers.dbos_worker
```

**Expected Output:**
```
Initializing DBOS queue worker...
Initializing DBOS application: pa-api-development
DBOS instance initialized
Defining queue: pa-workflows
Queue 'pa-workflows' registered with DBOS
Importing workflows (will register with DBOS)...
Workflows imported and registered with DBOS
Launching DBOS runtime...
DBOS runtime launched. Worker is running and processing workflows...
Press Ctrl+C to stop the worker
```

---

## Complete Workflow: From API Request to Execution

### Step-by-Step Flow

1. **API Receives Request**
   ```
   POST /pipeline/run/70132
   {
     "channels": ["Email"],
     "baseline_date": "2025-01-15"
   }
   ```

2. **API Enqueues Workflow**
   ```python
   # In pa/api/pipeline_endpoints.py
   dbos_client = get_dbos_client()
   handle = await dbos_client.enqueue_async(
       options={
           "queue_name": "pa-workflows",
           "workflow_name": "Predictive_Audience_pipeline"
       },
       client_pipeline_workflow,
       client_id=70132,
       channels=["Email"],
       baseline_date="2025-01-15"
   )
   ```

3. **DBOS Creates Workflow Entry**
   - Inserts row into PostgreSQL `workflow_status` table
   - Status: `PENDING`
   - Queue: `pa-workflows`
   - Parameters: Serialized workflow arguments

4. **Worker Polls Queue**
   - DBOS runtime (in worker) polls `workflow_status` table
   - Finds workflow with status `PENDING` and queue `pa-workflows`
   - Dequeues it (updates status to `RUNNING`)

5. **Worker Executes Workflow**
   ```python
   # In pa/flows/channel_pipeline.py
   @dbos.workflow(name="Predictive_Audience_pipeline")
   async def client_pipeline_workflow(...):
       # Step 1: Compute metrics
       metrics_result = compute_email_metrics_step(...)
       
       # Step 2: Generate features
       features_result = await generate_email_features_step(...)
       
       # Step 3: Generate segments
       segments_result = await segment_email_features_step(...)
       
       # Step 4: Calculate LTV
       ltv_result = await calculate_ltv_step(...)
   ```

6. **Step Execution with Checkpointing**
   - Each `@dbos.step` executes and checkpoints state
   - If step fails, retries up to `retries_allowed` times
   - State persisted to PostgreSQL after each step

7. **Workflow Completion**
   - Status updated to `COMPLETED` or `FAILED`
   - Results stored in `workflow_status` table
   - API can query status via `GET /pipeline/status/{workflow_id}`

---

## Workflow Definitions

### Workflow Types

1. **Client Pipeline Workflow** (`client_pipeline_workflow`)
   - Executes pipeline for all active channels
   - Workflow name: `"Predictive_Audience_pipeline"`
   - Handles multiple channels (Email, LP)

2. **Email Pipeline Workflow** (`email_pipeline_workflow`)
   - Executes email-specific pipeline
   - Workflow name: `"Predictive_Audience_email_pipeline"`
   - Steps: metrics → features → segments → LTV

3. **LP Pipeline Workflow** (`lp_pipeline_workflow`)
   - Executes landing page-specific pipeline
   - Workflow name: `"Predictive_Audience_lp_pipeline"`
   - Steps: metrics → features → segments → LTV

### Step Decorators

Each step is decorated with `@dbos.step`:

```python
@dbos.step(name="compute_email_metrics", retries_allowed=2)
def compute_email_metrics_step(client_id, baseline_date, skip_if_recent=True):
    # Step implementation
    pass
```

**Step Properties:**
- `name`: Unique identifier for the step
- `retries_allowed`: Number of automatic retries on failure (default: 2)
- State checkpointed after each step execution

---

## Database Schema

### workflow_status Table (PostgreSQL)

DBOS creates and manages this table automatically:

```sql
CREATE TABLE workflow_status (
    workflow_uuid UUID PRIMARY KEY,
    workflow_name VARCHAR(255),
    status VARCHAR(50),  -- PENDING, RUNNING, COMPLETED, FAILED
    queue_name VARCHAR(255),
    parameters JSONB,  -- Serialized workflow arguments
    result JSONB,     -- Workflow execution results
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

**Status Values:**
- `PENDING`: Workflow is in queue, waiting for worker
- `RUNNING`: Workflow is being executed by a worker
- `COMPLETED`: Workflow finished successfully
- `FAILED`: Workflow failed (after all retries)

---

## Configuration

### Environment Variables

**Required:**
- `DBOS_DATABASE_URL`: PostgreSQL connection string
  ```
  postgresql://dbos_user:dbos_password@localhost:5432/dbos
  ```

**Optional:**
- `DBOS_QUEUE_NAME`: Queue name (default: `"pa-workflows"`)
- `ENVIRONMENT`: Environment name (default: `"development"`)

### Configuration Files

**`pa/db/dbos_config.py`:**
- Reads environment variables
- Constructs DBOS configuration dictionary
- Returns queue name and database URL

**`dbos-config.yaml`:**
- Optional DBOS configuration file
- Used by DBOS CLI tools
- Can override environment variables

---

## Scaling and Deployment

### Horizontal Scaling

**Multiple Workers:**
- Run multiple worker instances
- All workers poll the same queue
- DBOS distributes workflows across workers
- Load balancing is automatic

**Example:**
```bash
# Terminal 1
python -m pa.workers.dbos_worker

# Terminal 2 (another worker)
python -m pa.workers.dbos_worker

# Both workers process from the same queue
```

### Deployment Architecture

```
┌─────────────┐
│   FastAPI   │  (1 instance, handles API requests)
│   API       │
└──────┬──────┘
       │
       │ enqueue_async()
       │
       ▼
┌─────────────┐
│ PostgreSQL  │  (DBOS system database)
│  (Queue)    │
└──────┬──────┘
       │
       ├──────────────┬──────────────┐
       │              │              │
       ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Worker 1 │  │ Worker 2 │  │ Worker 3 │  (N workers, process workflows)
└──────────┘  └──────────┘  └──────────┘
```

---

## Key Benefits

1. **Reliability**
   - State checkpointing: Workflows survive worker crashes
   - Automatic retries: Transient failures are handled
   - PostgreSQL ACID guarantees: No lost workflows

2. **Scalability**
   - Horizontal scaling: Add more workers for more throughput
   - Queue-based: Decouples API from execution
   - Load balancing: Automatic distribution across workers

3. **Observability**
   - Workflow status tracking in PostgreSQL
   - Step-level checkpointing
   - Retry history

4. **Separation of Concerns**
   - API server: Handles HTTP requests
   - Workers: Execute long-running workflows
   - Database: Stores workflow state

---

## Troubleshooting

### Workflow Stays in "PENDING"
- **Cause**: No worker running
- **Solution**: Start worker with `python -m pa.workers.dbos_worker`

### "Queue not found" Error
- **Cause**: Queue name mismatch between API and Worker
- **Solution**: Ensure `DBOS_QUEUE_NAME` is the same in both

### "Database connection failed"
- **Cause**: PostgreSQL not running or wrong credentials
- **Solution**: Check `DBOS_DATABASE_URL` and PostgreSQL status

### Workflow Fails Immediately
- **Cause**: Workflow function not registered
- **Solution**: Ensure worker imports workflows (they auto-register)

---

## Summary

**DBOS** = Workflow orchestration framework using PostgreSQL
**Queues** = Named queues that hold pending workflows (stored in PostgreSQL)
**Workers** = Services that dequeue and execute workflows from queues

**Flow:**
1. API enqueues workflow → PostgreSQL `workflow_status` table
2. Worker polls queue → Finds `PENDING` workflow
3. Worker executes workflow → Runs `@dbos.step` functions sequentially
4. State checkpointed → After each step to PostgreSQL
5. Workflow completes → Status updated to `COMPLETED`/`FAILED`

This architecture provides reliable, scalable workflow execution with automatic retries, state persistence, and horizontal scaling capabilities.

