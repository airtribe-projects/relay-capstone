# Relay Data Model

This is a language-agnostic data model. You may use SQL, NoSQL, or a mix, as long as you preserve the core entities and the correctness guarantees. A single relational database is a perfectly good answer for everything below, including the queue.

## Source Data Files

- `data/node_catalog.json` — the node/trigger type registry. Publish validation and the NL compiler validate against it.
- `data/seed_workflows.json` — four workflows to load as Published version 1 at startup. The JSON shape is the canonical definition contract.
- `data/sample_payloads.jsonl` — trigger bodies for demos and tests (inputs only, not entities to store).
- `data/nl_eval.jsonl` — labeled compiler test cases. For evaluation only; do not feed the assertions to the compiler.

## Entities

### Workflow

- `id` (e.g. `wf_support_triage`)
- `name`, `description`
- `status` (`draft` / `published`)
- `current_version`
- `created_at`, `updated_at`

### Workflow Version

Created on publish; immutable.

- `workflow_id`, `version`
- `definition` (the full JSON: trigger, entry, nodes, limits)
- `published_at`

Runs reference a version, not the live workflow — editing and republishing must not change what an in-flight or historical run executed.

### Run

- `run_id`
- `workflow_id`, `workflow_version`
- `status` (`queued`, `running`, `waiting_approval`, `succeeded`, `failed`, `cancelled`)
- `trigger_type` and `input` (the trigger body)
- `current_node_id` (nullable)
- `steps_executed` (for the max_steps cap)
- `ai_tokens_used` (for the token cap)
- `error` (nullable: message + which cap or node failed)
- `started_at`, `finished_at`

### Step

One row per node execution attempt-group; the trace.

- `run_id`, `node_id`, `node_type`
- `sequence` (execution order — node ids repeat when the run loops)
- `status` (`succeeded`, `failed`, `waiting`, ...)
- `attempt` (final attempt number)
- `resolved_input` (params after template resolution)
- `output`
- `tokens_prompt`, `tokens_completion` (ai nodes; null otherwise)
- `idempotency_key` (side-effect nodes)
- `started_at`, `duration_ms`

### Approval

- `id`, `run_id`, `node_id`
- `message` (resolved)
- `status` (`pending`, `approved`, `rejected`)
- `decided_by`, `decided_at` (nullable)

### Schedule

For cron triggers: `workflow_id`, `cron`, `next_fire_at`, `enabled`. Firing a schedule creates a normal Run.

### Queue

Any durable mechanism: a `queue_jobs` table with `SELECT ... FOR UPDATE SKIP LOCKED`, a Redis list, or a proper broker. Requirements: jobs survive restart, and a job picked up by a crashed worker becomes available again (visibility timeout, lease, or heartbeat).

## Correctness Requirements

These are where the grading pressure sits:

- **Persist, then acknowledge.** A step's completion (including its output and idempotency key) must be persisted before the engine moves to the next node. Resume logic derives entirely from persisted steps.
- **Stable idempotency keys.** `{run_id}:{node_id}` per side-effect node. A retry after a timeout and a resume after a crash must both send the same key the first attempt sent. If the first attempt reached the mock world, the replay is absorbed; if it never arrived, the call executes normally. Either way: exactly once.
- **The crash window.** The dangerous moment is after the side effect fired but before the step was persisted. On resume you cannot know which happened — so you re-send with the same key and let replay detection resolve it. This is why keys must not include the attempt number.
- **Loop accounting.** `steps_executed` increments on every node execution, including repeats of the same node. `wf_runaway` (max_steps 12) exists to verify this.
- **Approval gate as data.** "Has this run an approved approval?" is a query over Approval rows, checked by the engine before executing any `requires_approval` node. Nothing the AI outputs can write an Approval row.
- **Caps checked in the engine loop.** Before each node execution: steps, tokens, and elapsed time against `limits`. Exceeding one ends the run `failed` with a reason naming the cap.

## Relationships

- A Workflow has many Versions; a Version has many Runs.
- A Run has many Steps and zero or more Approvals.
- Schedules belong to a Workflow and create Runs.

## Storage Expectations

- Workflows, versions, runs, steps, approvals, and schedules survive a full process restart — the kill-and-resume drill depends on it.
- Run status transitions should be atomic (no run simultaneously `running` on two workers; a single worker makes this trivial, leases make it Stretch-ready).
- Traces are append-mostly and read by the console; index steps by `run_id, sequence`.
- Prompt/response body logging for `ai` steps is a deliberate design decision — if you store full prompts, document retention and privacy implications; storing token counts and the validated output is the Must Have.
