# DBOS Q&A Reference Guide

This document contains answers to common questions about DBOS and how it works in the Xyra Evaluation System. Use this as a quick reference guide.

---

## Table of Contents

1. [What Does DBOS Store in PostgreSQL?](#what-does-dbos-store-in-postgresql)
2. [Input/Output Data Storage](#inputoutput-data-storage)
3. [Scheduled Workflow Tracking](#scheduled-workflow-tracking)
4. [Execution History](#execution-history)
5. [scheduled_time vs actual_time](#scheduled_time-vs-actual_time)

---

## What Does DBOS Store in PostgreSQL?

### Overview

DBOS uses PostgreSQL as its state store to ensure reliable, exactly-once workflow execution with full history and recovery capabilities.

### What Gets Stored

#### 1. Workflow Status and Execution Data
- **Workflow Metadata**: IDs, names, status (running, completed, failed, etc.)
- **Execution History**: When workflows started, completed, or failed
- **Input/Output Data**: Workflow inputs and outputs (serialized as text/JSON)
- **Error Information**: Error messages and stack traces for failed workflows
- **Execution Context**: Executor ID, application version, timestamps

#### 2. Workflow State and Checkpoints
- **Checkpoint Data**: Intermediate state for recovery after failures
- **Step Outputs**: Results from individual steps within workflows
- **Recovery Information**: Retry attempts, recovery state

#### 3. Scheduled Workflow Tracking
- **Schedule Metadata**: Cron expressions, next run times
- **Execution Guarantees**: Ensures scheduled workflows run exactly once
- **Schedule History**: When scheduled workflows were triggered

#### 4. Events and Notifications
- **Workflow Events**: Custom events set via `DBOS.set_event()` (e.g., `batch_evaluation_status`)
- **Notifications**: Inter-workflow notifications
- **Event History**: All events with timestamps

#### 5. Transaction and Operation Logs
- **Operation Outputs**: Results from each step/operation
- **Transaction Logs**: For transactional operations
- **Deduplication Data**: Prevents duplicate workflow executions

#### 6. System Metadata
- **Application Version**: Tracks which version of your app created workflows
- **Executor Information**: Which executor/instance ran workflows
- **Schema Version**: DBOS migration version

### Database Schema

DBOS creates tables like:
- `workflow_status`: Main workflow execution records
- `operation_outputs`: Step/operation results
- `workflow_events`: Custom events (your `batch_evaluation_status`)
- `notifications`: Inter-workflow messaging
- `streams`: Stream data
- `event_dispatch_kv`: Event dispatch key-value store

### In Your Specific Case

For the Xyra Evaluation System, DBOS stores:
1. **Each scheduled trigger execution**: When `trigger_evaluation_batch_workflow()` runs
2. **Batch status events**: The `batch_evaluation_status` events with workflow status
3. **HTTP call results**: Success/failure of calls to Xyra Analytics API
4. **Retry information**: If a call fails and needs retry
5. **Schedule tracking**: Ensures the cron job runs exactly once per schedule

---

## Input/Output Data Storage

### What DBOS Stores Automatically

#### Workflow-Level Data
- **Workflow Function Parameters**: The parameters passed to your workflow function
  - In your case: `scheduled_time` and `actual_time` (datetime objects)
- **Workflow Return Value**: What your workflow function returns (if anything)
- **Step Inputs/Outputs**: Inputs and outputs of individual `@DBOS.step()` functions

### What You're Explicitly Storing

In `dbos_evaluation_workflow.py`, you're storing:

```python
DBOS.set_event(EVENT_BATCH_STATUS, {
    "workflow_id": workflow_id,
    "status": "triggered/completed/skipped/failed",
    "processed": processed,      # From API response
    "succeeded": succeeded,       # From API response  
    "failed": failed,            # From API response
    "scheduled_time": scheduled_time.isoformat()
})
```

**Stored Data:**
- ✅ Processed/succeeded/failed counts from API response
- ✅ Workflow status (triggered, completed, skipped, failed)
- ✅ Scheduled time

### What is NOT Automatically Stored

- ❌ The full HTTP request payload you send: `{"workflow_id": "...", "triggered_at": "...", "batch_size_limit": 30}`
- ❌ The full HTTP response body (unless you explicitly store it)
- ❌ The raw API response JSON

**Note**: If you want to store the full request/response, you'd need to explicitly add it to the event or workflow output.

---

## Scheduled Workflow Tracking

### What DBOS Stores

- ✅ **Cron Expression**: Your `"*/2 * * * *"` is stored in DBOS's internal configuration
- ✅ **Execution Records**: Each time the scheduled workflow runs, DBOS creates a record
- ✅ **Next Execution Tracking**: DBOS calculates and tracks when the next run should occur
- ✅ **Execution Guarantees**: Ensures it doesn't run twice for the same scheduled time

### How It Works

1. **Registration**: When you define `@DBOS.scheduled("*/2 * * * *")`, DBOS registers this schedule
2. **Calculation**: DBOS calculates "Next run should be at 10:00, 10:02, 10:04..."
3. **Execution**: After each run, DBOS marks that scheduled time as "executed" and calculates the next one
4. **Recovery**: On restart, DBOS checks what's been executed and what's pending

### What Gets Stored

- The schedule pattern (cron expression)
- Which scheduled times have been executed
- Which scheduled times are pending
- Prevents duplicate executions for the same scheduled time

### Example

If your cron is `"*/2 * * * *"` (every 2 minutes):
- 10:00:00 - Executed ✅
- 10:02:00 - Executed ✅
- 10:04:00 - Pending ⏳
- 10:06:00 - Pending ⏳

DBOS tracks all of this in PostgreSQL.

---

## Execution History

### Full History Storage

**Yes, DBOS stores the complete execution history.**

### What Gets Stored for Each Run

- **Workflow UUID**: Unique ID for each execution
- **Status**: triggered → running → completed/failed/skipped
- **Start Time**: When the workflow started
- **End Time**: When it completed/failed
- **Duration**: How long it took
- **Inputs**: Workflow parameters
- **Outputs**: Workflow return value
- **Error Details**: If it failed, the error message and stack trace
- **Executor ID**: Which instance/executor ran it
- **Application Version**: Which version of your code ran it

### History Retention

- **By Default**: DBOS keeps all history (no automatic cleanup)
- **Queryable**: All past executions are queryable
- **Complete Record**: Every single run, when it ran, what happened, success/failure

### Example History

For your evaluation system, you'll have records like:

```
Run 1: 2025-01-15 10:00:00
  Status: completed
  Processed: 30, Succeeded: 28, Failed: 2

Run 2: 2025-01-15 10:02:00
  Status: completed
  Processed: 25, Succeeded: 25, Failed: 0

Run 3: 2025-01-15 10:04:00
  Status: skipped
  Message: "Batch evaluation already in progress"

Run 4: 2025-01-15 10:06:00
  Status: failed
  Error: "Connection timeout"
```

All of this history is stored in PostgreSQL and queryable.

---

## scheduled_time vs actual_time

### Key Difference

| Parameter | Description | Example |
|-----------|-------------|---------|
| `scheduled_time` | When the workflow **SHOULD have run** according to cron schedule | `10:00:00` |
| `actual_time` | When the workflow **ACTUALLY started executing** | `10:00:03` |

### Why They Differ

1. **System Load**: Previous workflow still running, CPU/memory pressure
2. **Database Delays**: Slow queries, connection pool exhaustion
3. **Network Delays**: If scheduler is distributed
4. **Resource Contention**: Other processes competing for resources
5. **Previous Workflow Overrun**: If previous run hasn't finished

### Example Scenarios

#### Scenario 1: On-Time Execution
```
scheduled_time: 2025-01-15 10:00:00
actual_time:    2025-01-15 10:00:00
Difference:     0 seconds (perfect timing)
```

#### Scenario 2: Small Delay
```
scheduled_time: 2025-01-15 10:00:00
actual_time:    2025-01-15 10:00:02
Difference:     2 seconds (minor delay)
```

#### Scenario 3: Significant Delay
```
scheduled_time: 2025-01-15 10:00:00
actual_time:    2025-01-15 10:00:15
Difference:     15 seconds (previous workflow took longer)
```

### In Your Code

Your workflow function receives both:

```python
async def trigger_evaluation_batch_workflow(
    scheduled_time: datetime,  # When it SHOULD have run
    actual_time: datetime      # When it ACTUALLY started
):
```

You're currently using `scheduled_time` in your payload:

```python
payload = {
    "workflow_id": workflow_id,
    "triggered_at": scheduled_time.isoformat()  # Using scheduled_time
}
```

### Best Practices

#### Use `scheduled_time` for:
- ✅ Business logic (e.g., "this batch was scheduled for 10:00")
- ✅ Reporting and analytics
- ✅ API payloads (what you're doing now)

#### Use `actual_time` for:
- ✅ Performance monitoring
- ✅ Delay detection
- ✅ Debugging timing issues

### Why This Matters

1. **Monitoring**: Track how often and by how much runs are delayed
2. **Debugging**: Identify if delays are causing issues
3. **Reporting**: Use `scheduled_time` for consistent reporting, `actual_time` for performance analysis
4. **Scheduling**: If delays are consistent, you might need to adjust cron schedule

---

## Quick Reference Summary

### What Gets Stored
- ✅ Workflow execution records (every run)
- ✅ Custom events (via `DBOS.set_event()`)
- ✅ Schedule metadata and execution tracking
- ✅ Complete execution history
- ✅ Step/operation outputs
- ✅ Error details and stack traces

### What Doesn't Get Stored Automatically
- ❌ Full HTTP request/response bodies (unless you add them)
- ❌ Raw API response JSON (unless you store it)
- ❌ Temporary variables or intermediate calculations

### Key Concepts
- **scheduled_time**: When it should run (use for business logic)
- **actual_time**: When it actually ran (use for monitoring)
- **History**: Complete execution history is stored
- **Exactly-Once**: DBOS ensures scheduled workflows run exactly once

---

## Additional Notes

### Querying History

You can query DBOS tables directly in PostgreSQL to see:
- All workflow executions
- Event history
- Schedule execution records
- Error logs

### Monitoring

Track these metrics:
- Delay between `scheduled_time` and `actual_time`
- Success/failure rates
- Average execution duration
- Retry frequency

### Troubleshooting

If workflows are delayed:
1. Check `actual_time` vs `scheduled_time` difference
2. Look for long-running previous workflows
3. Check database connection pool
4. Monitor system resources

---

**Last Updated**: 2025-01-15  
**Version**: 1.0

