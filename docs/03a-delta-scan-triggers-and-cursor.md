# 03a Delta Scan Triggers & Cursor

## Table of Contents
1. [TL;DR](#tldr)
2. [Delta vs Full Scan](#delta-vs-full-scan)
3. [Trigger Mechanisms](#trigger-mechanisms)
4. [The Last-Seen Cursors](#the-last-seen-cursors)
5. [Cursor Implementations in Detail](#cursor-implementations-in-detail)
6. [Timeline & Lifecycle Diagrams](#timeline--lifecycle-diagrams)

## TL;DR
- **Delta Scan** is the default incremental run, relying on Resource Explorer (tags) and AWS Config (config drift) to only ingest changes. **Full Scan** simply represents the first iteration when no previous state/cursor exists.
- Delta Scans are triggered exclusively via **Celery Beat schedules** (no event-driven webhooks/SQS found).
- State and incremental progress are tracked via three main cursor concepts: `ConfigChangeCursor.last_polled_at`, Pagination `NextToken`, and `update_tag`/`last_seen` timestamps for cleanup.

## Delta vs Full Scan

- **Delta Scan:** Calculates differences between the current cloud state (using AWS Resource Explorer and AWS Config) and the locally stored hashes (`api_resourcestatehash`). Only resources categorized as `new` or `changed` are fetched via full API calls and evaluated by OPA.
- **Full Scan (Fallback):** The system does not have a dedicated "Full Scan" function for the primary inventory loop. Instead, if `load_hashes(account_id)` returns an empty state, the delta logic intrinsically treats every discovered resource as `new`, converting the run into a full scan. (Note: "Deep Scans" using Cartography are a separate exhaustive mapping process).

## Trigger Mechanisms

- **Schedules:** `worker.tasks.periodic_inventory_pulls` runs periodically via Celery Beat, enqueuing `perform_inventory_pull_task`.
- **Events/Webhooks:** **Not found in codebase**. (Searched for EventBridge, SQS, SNS, and explicit API Webhooks). All delta mechanisms rely on scheduled polling.

## The Last-Seen Cursors

The application utilizes several distinct cursor types to manage pagination and incremental state fetching.

### 1. `last_polled_at` (Config Drift Cursor)
- **What it tracks:** Tracks the last time AWS Config was queried for changes, per `account_id` and `region`.
- **Where it is stored:** PostgreSQL table `api_configchangecursor` (`app/models/orm.py:L169-L177`).
- **Initial value:** `utcnow() - 180 minutes` (from `lookback_minutes_default` setting).
- **Read/Write Lifecycle:** Read at the start of `collect_changed_arns_from_config` (`worker/jobs/inventory/config_drift.py:L41`). Written only upon successful fetching of all paginated results (`config_drift.py:L72`). It records the timestamp `now` taken at the *start* of the run.
- **Filtering:** Filters the AWS Config query via `configurationItemCaptureTime > '{since}'`.
- **Overlap Window:** Because `last_polled_at` is set to the *start time* of the current run, any AWS Config events captured after the run starts are safely picked up on the next run. However, if AWS Config experiences ingestion delays (event capture time is before `now`, but the event isn't queryable until after the run finishes), it might be missed.

### 2. `NextToken` / Pagination Cursors
- **What it tracks:** AWS API pagination states (e.g., `select_resource_config`).
- **Where it is stored:** Kept purely in memory during the execution of a Celery task.
- **Failure handling:** If a task crashes mid-pagination, the `NextToken` is lost. Because `last_polled_at` is only updated *after* full success, the entire query will seamlessly restart from the original `last_polled_at` during the retry.

### 3. `last_seen` / `update_tag` (State Cleanup Cursors)
- **What it tracks:** The staleness of resources and findings to handle deletions. 
- **Where it is stored:** PostgreSQL `api_finding.last_seen` and Neo4j node properties `last_updated` / `update_tag`.
- **Usage:** When a resource is discovered, its `last_seen` is updated to the current run timestamp. Later, any finding or graph node older than the current run's timestamp (`update_tag`) is identified as deleted and removed (e.g., `worker/jobs/deep_scan/findings.py:L86-93`).

## Timeline & Lifecycle Diagrams

### Worked Timeline (Delta Scan Cursor)

1. **Run 1 (Initial):** Starts at `10:00:00`.
   - Cursor: None. Initialized to `07:00:00` (180 min lookback).
   - Query: `configurationItemCaptureTime > '07:00:00'`.
   - Results: Fetches 50 changes.
   - Outcome: Success. `last_polled_at` updated to `10:00:00`.

2. **Run 2 (Failure):** Starts at `10:15:00`.
   - Cursor read: `10:00:00`.
   - Query: `configurationItemCaptureTime > '10:00:00'`.
   - Outcome: AWS API throws a `RateExceeded` error mid-way. The run crashes. `last_polled_at` is **not** updated and remains `10:00:00`.

3. **Run 3 (Retry):** Celery retries at `10:16:00`.
   - Cursor read: `10:00:00`.
   - Query: `configurationItemCaptureTime > '10:00:00'`.
   - Results: Re-fetches the changes from `10:00:00` to `10:16:00`. (Duplicate fetches are handled safely via the strict hash checks in `has_config_drifted`).
   - Outcome: Success. `last_polled_at` updated to `10:16:00`.

### Cursor Lifecycle Diagram

```mermaid
stateDiagram-v2
    [*] --> ReadCursor: Celery Task Starts
    ReadCursor --> CheckFallback: Read last_polled_at
    
    state CheckFallback {
        direction LR
        Valid --> QueryAWS
        None --> ApplyLookback (now - 180m) --> QueryAWS
    }
    
    CheckFallback --> Paginate: Execute AWS Query
    
    state Paginate {
        direction LR
        FetchPage --> ProcessRows
        ProcessRows --> HasNextToken?
        HasNextToken? --> FetchPage: NextToken
        HasNextToken? --> Done: No Token
    }
    
    Paginate --> Success: All Pages Fetched
    Paginate --> Crash: API Error / OOM
    
    Crash --> [*]: Task Fails (Cursor unchanged)
    
    Success --> UpdateCursor: write last_polled_at = task_start_time
    UpdateCursor --> [*]: Task Completes
```
