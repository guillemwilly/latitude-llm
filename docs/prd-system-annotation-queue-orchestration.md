# PRD: System Annotation Queue Orchestration

## Overview

Implement the backend pipeline that automatically detects issues in LLM traces and creates draft annotations for human review. This is the orchestration layer on top of the existing annotation queue foundation (schemas, entities, constants, topic registry, domain event dispatch).

When a trace ends, the system screens it against 9 predefined issue categories, validates matches with a more capable model, and creates draft annotation scores queued for human review.

**Spec phase:** (LAT-474) Phase 17 - Annotation Queues Orchestration And Product Surface
**Depends on:** Phase 0 (Async Foundations), Phase 6 (Annotation Queues Foundations), Phase 9 (Scores Analytics), Phase 10 (Annotations Backend), Phase 16 (Keep-Monitoring Settings)
**Scope of this PRD:** Backend orchestration only. The frontend (queue management page, focused review screen, manual bulk-add from dashboards) is deferred to a follow-up.

## Non-goals

- **Frontend surfaces**: Queue management page, focused review screen, manual bulk-add from dashboards — all deferred to a follow-up PRD.
- **Custom/user-defined system queues**: Users cannot create new system queue categories; the 9 predefined categories are fixed.
- **Retroactive annotation of historical traces**: The pipeline only processes new traces via `TraceEnded`. Backfill tooling for existing traces is out of scope.
- **Real-time streaming of annotations**: Annotations are written asynchronously and discovered via polling/refresh, not pushed via WebSocket.
- **Prompt authoring**: The flagger and annotator prompts are owned by the prompt engineer and developed in parallel. This PRD defines the I/O contracts only.

## Current State

### What exists

- **Domain entities and schemas**: `AnnotationQueue`, `AnnotationQueueItem`, `AnnotationQueueSettings` in `packages/domain/annotation-queues/`
- **Constants**: 8 system queue definitions (Thrashing is in the spec but missing from the code — see note below), sampling defaults (10%), flagger context window (8 messages), outlier multiplier, deterministic queue names in `packages/domain/annotation-queues/src/constants.ts`
- **DB tables**: `annotation_queues` and `annotation_queue_items` with RLS, indexes, and unique constraints in `packages/platform/db-postgres/src/schema/annotation-queues.ts`
- **Score model**: Full annotation score support with `source = "annotation"`, `sourceId` = queue CUID, `draftedAt` draft semantics, metadata with anchor fields in `packages/domain/scores/`
- **Topic registry**: `system-annotation-queues` (flag, annotate), `live-annotation-queues` (curate), `annotation-scores` (publish) in `packages/domain/queue/src/topic-registry.ts`
- **Event dispatch**: `TraceEnded` handler in `apps/workers/src/workers/domain-events.ts` publishes to two annotation queue topics concurrently: `system-annotation-queues:flag` and `live-annotation-queues:curate` (alongside `live-evaluations:enqueue`). The `annotation-scores:publish` topic is triggered separately after human review, not on `TraceEnded`.
- **Worker stubs**: Three annotation-related workers registered in `apps/workers/src/server.ts` (`createSystemAnnotationQueuesWorker`, `createLiveAnnotationQueuesWorker`, `createAnnotationScoresWorker`), all currently log-only stubs
- **Seeds**: Representative manual, system, and live queue data with queue items

### What does NOT exist

- Worker handler implementations (flag, annotate, curate, publish)
- Flagger prompt (lightweight LLM screening)
- Annotator prompt (full-context validation + draft writing)
- Deterministic detection logic for Tool Call Errors and Resource Outliers
- Thrashing payload builder (span tree → `ThrashingFlaggerPayload`)
- Queue CRUD use-cases
- Project provisioning of default system queues
- Annotation queue repository
- Live queue filter materialization logic

## Architecture

### Pipeline Flow

```
TraceEnded (domain event)
    │
    ├─► system-annotation-queues:flag
    │       │
    │       ├─ list system=true queues for project
    │       ├─ apply settings.sampling per queue (skip if not sampled)
    │       ├─ deterministic checks: Tool Call Errors, Resource Outliers
    │       ├─ flagger LLM: remaining queues (last 8 messages + queue metadata)
    │       │   └─ Thrashing queue gets structured ThrashingFlaggerPayload
    │       └─ for each flagged queue:
    │           └─► publish system-annotation-queues:annotate { queueId, traceId }
    │
    ├─► system-annotation-queues:annotate
    │       │
    │       ├─ fetch full conversation context via TraceRepository
    │       ├─ run annotator LLM (validates flag + writes draft annotation)
    │       └─ if confirmed:
    │           ├─ insert score (source="annotation", sourceId=queueCUID, draftedAt=now)
    │           └─ insert annotation_queue_items row (completedAt=null)
    │           (both in same transaction)
    │
    └─► live-annotation-queues:curate
            │
            ├─ list non-deleted live queues (settings.filter present)
            ├─ apply FilterSet against trace
            ├─ apply settings.sampling
            └─ batch insert annotation_queue_items (completedAt=null)
```

### Reference Pattern

The `live-evaluations` worker follows the same architectural pattern: topic subscription → trace fetch via `TraceRepository` → LLM execution → score write via `writeScoreUseCase` → `ScoreImmutable` event. The annotation queue handlers should mirror this structure.

**Important difference:** Unlike evaluation scores which are immediately immutable, system-created annotation scores are **drafts** (`draftedAt != null`). They do NOT emit `ScoreImmutable` at creation time. Draft annotations are excluded from issue discovery, aggregates, and ClickHouse analytics until a human reviews and finalizes them (clears `draftedAt`). Only after finalization does the score become immutable and trigger downstream processing via `annotation-scores:publish`.

## Components To Build

### 1. Annotation Queue Repository

**Package:** `packages/domain/annotation-queues/` (new use-cases and repository port)
**Platform:** `packages/platform/db-postgres/` (repository implementation)

Provides:
- `listSystemQueuesByProject(organizationId, projectId)` — returns all non-deleted `system = true` queues
- `listLiveQueuesByProject(organizationId, projectId)` — returns all non-deleted queues where `settings.filter` is present
- `insertQueueItem(organizationId, projectId, queueId, traceId)` — inserts one `annotation_queue_items` row, no-ops on unique constraint conflict
- `batchInsertQueueItems(items[])` — bulk insert for live queue materialization
- `incrementTotalItems(queueId)` — bumps `totalItems` counter on the queue

Standard CRUD for queue management (create, update, soft-delete, list with pagination) is also needed but is lower priority than the orchestration handlers.

### 2. `system-annotation-queues:flag` Handler

**File:** `apps/workers/src/workers/system-annotation-queues.ts`

**Input payload:**
```typescript
{ organizationId: string, projectId: string, traceId: string }
```

**Steps:**
1. Fetch all `system = true` queues for the project via repository
2. For each queue, apply probabilistic sampling check against `settings.sampling` (default 10%). Skip queue if not sampled.
3. Partition sampled queues into deterministic vs LLM-classified (using `DETERMINISTIC_SYSTEM_QUEUE_NAMES` constant)
4. **Deterministic checks:**
   - **Tool Call Errors**: Query trace spans for any span with error status or tool-type spans with error outcomes. Flag if found.
   - **Resource Outliers**: Compare trace latency/cost/tokens against project medians. Flag if any metric exceeds `median * RESOURCE_OUTLIER_MULTIPLIER (3)`. Requires a query for project median stats.
5. **LLM flagger** (for non-deterministic queues):
   - Fetch last `SYSTEM_QUEUE_FLAGGER_CONTEXT_WINDOW (8)` messages from trace
   - For Thrashing queue: build `ThrashingFlaggerPayload` from span tree instead
   - Call flagger prompt with conversation excerpt + array of queue `{ name, description, instructions }`
   - Parse boolean decision per queue
6. For each flagged queue, publish `system-annotation-queues:annotate` with `{ organizationId, projectId, queueId, traceId, flaggerReason }` (reason from flagger output is forwarded via the topic payload — see resolved decision below)

**Flagger LLM input contract (for prompt engineer):**

```typescript
// Standard queues (all except Thrashing)
type FlaggerInput = {
  conversation: Array<{ role: string; content: string }>  // last 8 messages
  queues: Array<{
    id: string
    name: string
    description: string
    instructions: string
  }>
}

// Thrashing queue only
type ThrashingFlaggerInput = {
  payload: ThrashingFlaggerPayload  // structured span-tree summary
  queue: {
    id: string
    name: string
    description: string
    instructions: string
  }
}
```

**Flagger LLM output contract:**

```typescript
type FlaggerOutput = {
  decisions: Array<{
    queueId: string
    flagged: boolean
    reason: string  // short justification, passed downstream to annotator
  }>
}
```

### 3. Thrashing Payload Builder

**Package:** `packages/domain/annotation-queues/` (new helper)

Builds `ThrashingFlaggerPayload` from trace span data:

```typescript
type ThrashingFlaggerPayload = {
  conversation_excerpt: Array<{ role: string; content: string }>
  system_prompt_excerpt: string
  turn_count: number
  tool_call_sequence: Array<{
    tool_name: string
    call_index: number
    outcome: "success" | "error" | "empty_result"
  }>
  tool_call_summary: {
    total_calls: number
    failed_calls: number
    unique_tools_called: number
    repeated_tool_calls: Array<{ tool_name: string; call_count: number }>
    tools_available: string[]
    tools_used: string[]
  }
}
```

**ClickHouse query:** Query the `spans` table filtered by `organization_id` and `trace_id`. Use `SpanRepositoryLive.findByTraceId()` which returns all spans for a trace with columns including `name`, `tool_name`, `tool_call_id`, `tool_input`, `tool_output`, `status_code` (0=UNSET, 1=OK, 2=ERROR), `start_time`, `end_time`, and `duration_ns`. Tool spans are identified by non-empty `tool_name` or `tool_call_id`. Order by `start_time` to reconstruct the call sequence. The conversation excerpt and system prompt can be extracted from the root span's message attributes or via `TraceRepositoryLive.findByTraceId()`.

### 4. `system-annotation-queues:annotate` Handler

**File:** `apps/workers/src/workers/system-annotation-queues.ts`

**Input payload:**
```typescript
{ organizationId: string, projectId: string, queueId: string, traceId: string, flaggerReason: string }
```

> **Resolved decision (flagger reason propagation):** The `flaggerReason` field is included directly in the `annotate` topic payload. This is the simplest approach — a short string (< 500 chars) adds negligible overhead to the message, avoids a transient store, and avoids re-deriving context in the annotator. The `annotate` task schema in `topic-registry.ts` must be updated to add this field.

**Steps:**
1. Fetch queue by ID (get name, description, instructions)
2. Fetch full conversation context via `TraceRepository.findByTraceId()`
3. Call annotator prompt with full trace + queue metadata + `flaggerReason` from payload
4. If `confirmed: false` → done, no annotation created
5. If `confirmed: true` → in a single transaction:
   a. Create score via `writeScoreUseCase`:
      - `source = "annotation"`
      - `sourceId = queue CUID`
      - `value` = severity from LLM (0-1)
      - `passed = false` (always false — a confirmed flag is a detected issue)
      - `feedback` = LLM-written annotation text (clusterable, describes the failure pattern)
      - `metadata` = `{ rawFeedback: feedback }` (copy feedback into `rawFeedback`; conversation-level, no anchor fields)
      - `error = null` (this is a detected issue, not an execution error)
      - `errored = false` (must be maintained by application code per spec)
      - `draftedAt = new Date()` (draft until human review)
      - `duration` = annotator LLM call duration in nanoseconds
      - `tokens` = annotator LLM token usage
      - `cost` = annotator LLM cost in microcents
   b. Insert `annotation_queue_items` row with `completedAt = null`
   c. Increment queue `totalItems`
   d. Do NOT emit `ScoreImmutable` — drafts are excluded from analytics and issue discovery until human review clears `draftedAt`

**Annotator LLM input contract (for prompt engineer):**

```typescript
type AnnotatorInput = {
  trace: {
    traceId: string
    messages: Array<{ role: string; content: string }>  // full conversation
    systemPrompt: string | null
    metadata: Record<string, unknown>
    spans: Array<{  // relevant span summary
      name: string
      type: string
      duration_ms: number
      status: string
      error: string | null
    }>
    summary: {
      total_tokens: number
      total_cost: number
      total_duration_ms: number
      tool_call_count: number
    }
  }
  queue: {
    name: string
    description: string
    instructions: string
  }
  flagger_reason: string  // from the flag step
}
```

**Annotator LLM output contract:**

```typescript
type AnnotatorOutput = {
  confirmed: boolean       // was the flagger correct? if false, no annotation is created
  value: number            // severity 0 (minor) to 1 (critical); only used when confirmed=true
  feedback: string         // human-readable annotation text describing the failure pattern;
                           // must be phrased so similar failures can cluster together cleanly
                           // (this becomes both score.feedback and score.metadata.rawFeedback)
}
```

> **Note on `passed`:** The handler always sets `passed = false` for confirmed system annotations since a confirmed flag means an issue was detected. The LLM does not output `passed` — it only decides `confirmed` (whether the flag was valid) and `value` (severity).
>
> **Note on `labels`:** The score model does not have a labels field. If issue categorization is needed, it should flow through the existing issue discovery pipeline which clusters scores by their `feedback` text, not through LLM-generated labels on individual annotations.

### 5. `live-annotation-queues:curate` Handler

**File:** `apps/workers/src/workers/live-annotation-queues.ts`

**Input payload:**
```typescript
{ organizationId: string, projectId: string, traceId: string }
```

**Steps:**
1. List all non-deleted live queues for the project (queues with `settings.filter`)
2. Fetch trace metadata needed for filter evaluation
3. For each live queue:
   a. Apply `settings.filter` (FilterSet) against the trace using shared filter semantics (reuse `buildClickHouseWhere` or equivalent in-memory matcher)
   b. If filter matches, apply probabilistic `settings.sampling` check (default `LIVE_QUEUE_DEFAULT_SAMPLING = 10`)
   c. If both pass, include in batch insert
4. Batch insert all matching `annotation_queue_items` rows with `completedAt = null`
5. Unique constraint handles deduplication on retry

### 6. `annotation-scores:publish` Handler

**File:** `apps/workers/src/workers/annotation-scores.ts`

**Input payload:**
```typescript
{ organizationId: string, projectId: string, scoreId: string }
```

This handler processes **finalized** annotation scores (after human review clears `draftedAt`) for downstream analytics. It is NOT triggered at draft creation time. It is triggered when a human reviews a draft annotation in the queue review screen and finalizes it.

**Steps:**
1. Fetch the score by ID
2. Verify `draftedAt = null` (score has been finalized)
3. Sync to ClickHouse analytics table (mirrors `analytic-scores:save` pattern)
4. Trigger issue discovery refresh if `passed = false` and `issueId = null` (the score is now eligible for issue assignment)

### 7. Project Queue Provisioning

**Package:** `packages/domain/annotation-queues/` (new use-case)

When a project is created (or on first trace if lazy), provision the 9 default system queues using `SYSTEM_QUEUE_DEFINITIONS` from constants:

- Insert one `annotation_queues` row per definition
- `system = true`, `settings.sampling = SYSTEM_QUEUE_DEFAULT_SAMPLING (10)`
- No `settings.filter` (system queues are manual)
- Idempotent: unique constraint on `(orgId, projectId, name, deletedAt)` prevents duplicates

**Note:** The Thrashing queue definition exists in the spec but is NOT yet in `SYSTEM_QUEUE_DEFINITIONS` constant (which only has 8 entries). It needs to be added.

### 8. Deterministic Detection Logic

**Package:** `packages/domain/annotation-queues/` (new helpers)

**Tool Call Errors:**
- Query `spans` table via `SpanRepositoryLive.findByTraceId()` for the given trace
- Filter spans where `tool_name != ''` (tool spans) AND `status_code = 2` (ERROR), or any span with `status_code = 2` and non-empty `error_type`
- Flag if count > 0
- Relevant columns: `tool_name`, `tool_call_id`, `status_code`, `error_type`, `status_message`

**Resource Outliers:**
- **Trace metrics** come from `traces` table via `TraceRepositoryLive.findByTraceId()`: `duration_ns` (alias), total tokens (sum of `input_token_count` + `output_token_count`), total cost
- **Project medians**: Compute on-the-fly from `traces` table with a 7-day rolling window using ClickHouse `quantile(0.5)`:
  ```sql
  SELECT
    quantile(0.5)(reinterpretAsInt64(max_end_time) - reinterpretAsInt64(min_start_time)) AS median_duration_ns,
    quantile(0.5)(sum_total_tokens) AS median_tokens,
    quantile(0.5)(sum_total_cost) AS median_cost
  FROM traces
  WHERE organization_id = {organizationId:String}
    AND project_id = {projectId:String}
    AND min_start_time >= now() - INTERVAL 7 DAY
  ```
  This avoids materialized views for the initial implementation. If query latency becomes a problem at scale, promote to a materialized view later.
- Compare each trace metric against `median * RESOURCE_OUTLIER_MULTIPLIER (3)`
- Flag if any metric exceeds threshold
- Edge case: if fewer than 30 traces exist in the window, skip outlier detection (insufficient data for meaningful medians)

### 9. Finalize Annotation Use-Case

**Package:** `packages/domain/annotation-queues/` (new use-case)

This is the missing link between human review and the `annotation-scores:publish` handler. When a human reviewer finalizes a draft annotation in the queue review screen:

**Steps:**
1. Clear `draftedAt` on the score (set to `null`) — score becomes immutable
2. Set `completedAt` on the corresponding `annotation_queue_items` row
3. Increment `completedItems` counter on the queue
4. Publish to `annotation-scores:publish` topic with `{ organizationId, projectId, scoreId }`

This use-case is also the place to handle **draft rejection**: if the reviewer discards the annotation, delete the draft score and still set `completedAt` on the queue item (the item was reviewed, just not approved).

**Note:** This use-case is called from the frontend review screen (HTTP boundary), not from a worker. It is included here because the `annotation-scores:publish` handler depends on it to be triggered.

## Error Handling and Retry Semantics

All annotation queue workers are **at-least-once** with idempotent writes:

### Flag handler (`system-annotation-queues:flag`)
- **LLM failure**: Log error and skip the trace. Do not retry LLM calls inline — the queue infrastructure will redeliver the message per its retry policy. Since sampling is probabilistic, a missed trace is acceptable.
- **Repository failure**: Let the error propagate; the message will be retried by the queue.
- **Partial failure**: If deterministic checks succeed but the LLM flagger fails, still publish annotate messages for deterministically-flagged queues.

### Annotate handler (`system-annotation-queues:annotate`)
- **LLM failure**: Log error and let the message retry. The annotator is a single LLM call; retrying the full message is appropriate.
- **Score insert conflict**: The `writeScoreUseCase` should handle upsert/conflict semantics. If a score already exists for this `(traceId, source, sourceId)` combination, the retry is a no-op.
- **Queue item insert conflict**: The unique constraint on `(org, project, queue, trace)` in `annotation_queue_items` handles deduplication naturally — insert with `ON CONFLICT DO NOTHING`.

### Curate handler (`live-annotation-queues:curate`)
- **Filter evaluation failure**: Log and skip the individual queue; continue processing remaining queues.
- **Batch insert**: Uses `ON CONFLICT DO NOTHING` for natural deduplication on retry.

### Publish handler (`annotation-scores:publish`)
- **ClickHouse sync failure**: Let the message retry. ClickHouse inserts are naturally idempotent (same row re-inserted is a no-op with ReplacingMergeTree).
- **Stale `draftedAt` guard**: If the score still has `draftedAt != null` when the handler runs, log a warning and discard the message (should not happen in normal flow).

## Concurrency and Rate-Limiting

### Trace volume considerations
For a project with high trace volume, each `TraceEnded` event triggers both the flag and curate handlers. The flag handler may then fan out to multiple annotate messages (one per flagged queue).

### Recommended controls
- **Worker concurrency**: Set per-worker concurrency limits in the queue subscription config (e.g., 5 concurrent flag handlers, 3 concurrent annotate handlers per worker instance). The annotate handler is the most expensive (larger LLM call) and should have the lowest concurrency.
- **Sampling as natural throttle**: The default 10% sampling rate means ~90% of traces are skipped at the flag stage. This is the primary cost control mechanism.
- **No per-queue rate limiting in v1**: Rely on sampling + worker concurrency for now. If a specific queue generates excessive annotations, users can reduce its sampling rate via settings.
- **LLM call budget**: Consider adding a per-project daily budget for annotation LLM calls (tracked via a simple counter in Redis). This is a v2 concern but worth noting.

## Prompt Engineering Deliverables

These are owned by the prompt engineer and developed in parallel:

### Flagger Prompt
- **Model**: Small/fast (e.g., Haiku or similar)
- **Input**: Last 8 conversation messages + queue definitions (batch of queues in one call)
- **Output**: Structured JSON with boolean flag + short reason per queue
- **Key constraint**: Must handle batch of queues in single call for efficiency
- **Thrashing variant**: Receives `ThrashingFlaggerPayload` instead of raw messages

### Annotator Prompt
- **Model**: Larger/more capable (e.g., Sonnet) — spec says "larger validator/drafter LLM"
- **Input**: Full conversation context + queue metadata + flagger reason
- **Output**: Structured JSON with `confirmed`, `value` (0-1 severity), `feedback` (clusterable failure description)
- **Key constraint**: Must be able to reject flagger's decision (`confirmed: false`); `feedback` must describe the failure pattern in a way that clusters cleanly for issue discovery
- **Type**: Single-shot structured output (`type: chat`), not agent

## Draft Annotation Lifecycle

System-created annotations follow the spec's score lifecycle states:

1. **Draft** (`draftedAt != null`): Created by `system-annotation-queues:annotate` when the validator/drafter confirms a flag. Excluded from:
   - Default score listings and aggregates
   - Issue discovery
   - Evaluation alignment
   - ClickHouse analytics sync
   - Visible only in draft-aware surfaces (queue review screen, in-progress annotation editing)

2. **Finalized** (`draftedAt = null`): After human review in the queue review screen clears `draftedAt`. The score becomes immutable and triggers `annotation-scores:publish` for downstream processing.

3. **Rejected draft**: If the human reviewer determines the system annotation is incorrect, the draft score can be deleted or edited before finalization. The queue item's `completedAt` is set regardless.

**Key invariant from spec:** "Writers must never emit a passed score with a non-empty `error`." System annotations set `error = null`, `errored = false`, `passed = false`.

## Score Provenance Rules

Per the spec, annotation `source_id` values:
- **Queue-created annotations**: `sourceId = queue CUID` (the annotation queue's ID)
- **UI-created annotations**: `sourceId = "UI"` (sentinel)
- **API-created annotations**: `sourceId = "API"` (sentinel)

This lets downstream systems distinguish how each annotation was created. System queue annotations always use the queue CUID.

## Implementation Order

### Phase A: Repository + Queue Provisioning (days 1-2)
1. Annotation queue repository (list system queues, list live queues, insert items)
2. Project queue provisioning use-case
3. Add Thrashing to `SYSTEM_QUEUE_DEFINITIONS`

### Phase B: Flag Handler (days 2-4)
1. Sampling utility
2. Deterministic checks (tool call errors, resource outliers)
3. Thrashing payload builder
4. Flagger prompt integration
5. `system-annotation-queues:flag` handler wiring

### Phase C: Annotate Handler (days 4-5)
1. Annotator prompt integration
2. Score + queue item creation in transaction
3. `system-annotation-queues:annotate` handler wiring

### Phase D: Live Curate Handler (days 5-6)
1. Filter evaluation against trace
2. Sampling + batch insert
3. `live-annotation-queues:curate` handler wiring

### Phase E: Publish + Finalization + Tests (days 6-7)
1. Finalize annotation use-case (clears `draftedAt`, sets `completedAt`, publishes to `annotation-scores:publish`)
2. `annotation-scores:publish` handler (ClickHouse sync + issue discovery)
3. Integration tests for full pipeline

## Key Files To Modify

| File | Change |
|------|--------|
| `apps/workers/src/workers/system-annotation-queues.ts` | Replace stubs with flag + annotate handlers |
| `apps/workers/src/workers/live-annotation-queues.ts` | Replace stub with curate handler |
| `apps/workers/src/workers/annotation-scores.ts` | Replace stub with publish handler |
| `packages/domain/annotation-queues/src/constants.ts` | Add Thrashing queue definition |
| `packages/domain/annotation-queues/src/` | New: repository port, use-cases, helpers (thrashing payload, deterministic checks, sampling) |
| `packages/domain/queue/src/topic-registry.ts` | Add `flaggerReason: string` to the `annotate` task schema in `system-annotation-queues` topic |
| `packages/platform/db-postgres/src/` | New: annotation queue repository implementation |
| `packages/platform/db-clickhouse/src/repositories/trace-repository.ts` | New: `getProjectMedianStats()` query for outlier detection (7-day rolling window) |

## New Files To Create

| File | Purpose |
|------|---------|
| `packages/domain/annotation-queues/src/ports/annotation-queue-repository.ts` | Repository port interface |
| `packages/domain/annotation-queues/src/use-cases/provision-system-queues.ts` | Default queue provisioning |
| `packages/domain/annotation-queues/src/use-cases/flag-trace.ts` | Core flagging logic |
| `packages/domain/annotation-queues/src/use-cases/annotate-trace.ts` | Core annotation logic |
| `packages/domain/annotation-queues/src/use-cases/curate-live-queues.ts` | Live queue materialization |
| `packages/domain/annotation-queues/src/use-cases/finalize-annotation.ts` | Human review finalization (clears draftedAt, publishes to annotation-scores:publish) |
| `packages/domain/annotation-queues/src/helpers/thrashing-payload.ts` | Span tree → ThrashingFlaggerPayload |
| `packages/domain/annotation-queues/src/helpers/deterministic-checks.ts` | Tool error + outlier detection |
| `packages/domain/annotation-queues/src/helpers/sampling.ts` | Probabilistic sampling utility |
| `packages/platform/db-postgres/src/repositories/annotation-queue-repository.ts` | Repository implementation |
| Prompt files (location TBD) | Flagger + annotator PromptL files |

## Resolved Decisions

1. **Flagger reason propagation**: Add a `flaggerReason` field directly to the `annotate` topic payload in `topic-registry.ts`. Simple, no transient store needed, negligible message size overhead. See Section 4 for details.
2. **Project median stats**: Compute on-the-fly with a 7-day rolling window using `quantile(0.5)` on the `traces` table. Skip outlier detection when fewer than 30 traces exist. Promote to a materialized view later if query latency becomes a problem. See Section 8 for the query.

## Open Questions

1. **Flagger batching**: Should the flagger LLM evaluate all sampled queues in one call or one queue per call? Single call is cheaper but couples queue decisions.
2. **Provisioning trigger**: Provision system queues on project creation, or lazily on first `TraceEnded`? Lazy avoids unused queues but adds latency to first trace.
3. **Prompt file location**: Where should PromptL files for the flagger and annotator live? Under `packages/domain/annotation-queues/src/prompts/` or elsewhere?
4. **System queue editability**: The spec says `system = true` queues keep `name`, `description`, `instructions`, and `settings.filter` non-editable, but `settings.sampling` and `assignees` are editable. The CRUD layer needs to enforce this constraint.
