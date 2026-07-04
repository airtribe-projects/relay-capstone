# AI-First Software Engineering Capstone - Case Study

## Relay: AI Workflow Orchestrator

**Author:** Airtribe

## Background & Objective

Every operations team runs on workflows: when an order webhook arrives, check the amount, classify the request, notify the right channel, wait for a manager's approval, then act. Platforms like n8n, Temporal, and Zapier exist because wiring these flows by hand — and keeping them running when steps fail — is real infrastructure work.

The 2026 version of this problem is AI-first in two ways. First, the steps themselves now include machine judgment: an AI node reads the webhook payload and decides which branch the run takes. Second, the builder is AI too: users describe a workflow in plain English and the platform compiles it into a valid, runnable definition. Neither works without serious engineering underneath — runs must survive crashes, side effects must never fire twice, and an AI step's output must be structurally valid before anything downstream trusts it.

The objective is to build a reliable workflow orchestration platform that can:

- Define workflows as connected, typed nodes with branching, and manage them through a draft → publish lifecycle with versioning.
- Trigger runs from webhooks, schedules, or manual API calls.
- Execute runs durably on background workers: every step persisted, crash recovery without repeating side effects, retries with backoff.
- Run AI nodes whose output is enforced against a JSON schema, so downstream branching can trust it.
- Pause runs at human approval gates and resume on decision.
- Compile plain-English descriptions into valid workflows, and prove it against a labeled eval set.
- Enforce run-level limits — max steps, timeouts, token caps — so no run can loop or spend forever.
- Keep a per-step trace complete enough to debug any run after the fact.

## Language and Stack

This project is intentionally language agnostic. You may implement it in Node.js, Java, Python, Go, Ruby, or any stack of your choice.

You are free to choose your framework, database, and queue. You do not need a paid AI provider: free-tier APIs (for example Gemini or Groq) or locally hosted models (for example via Ollama) are acceptable, and the engine can be tested against a mocked model adapter. The pack includes a mock external-services server (the "mock world") so notify and action nodes have something real to call — it records every call it receives in a ledger, which is how exactly-once execution is verified. Your choices must be documented, and the system must satisfy the functional, durability, and verification requirements described below.

The provided Python scripts are optional local utilities. They are not part of the required implementation stack.

## Scope

Relay has a clear core scope. Build the Must Have features first. Good To Have and Stretch features should only be attempted after the core engine works end to end.

Relay is the distributed-systems-heavy option in the capstone set. Most of the effort goes into the execution engine — durability, idempotency, state machines, recovery — with the AI concentrated in two places: AI nodes making decisions inside runs, and the compiler turning natural language into workflow definitions. Pick it if orchestration is the kind of engineering you want to show.

There is no drag-and-drop builder to build. Workflows are created via API (as JSON) or via the natural-language compiler. The frontend is a read-and-operate console: workflow list, run history, run traces, and pending approvals.

## Recommended Demo Flow

Your final demo should be able to show this flow clearly:

1. Start the mock world server and your engine; show the console with the seeded workflows.
2. Trigger the seed workflow `wf_support_triage` with sample payload `pay_001` (a normal order complaint). Walk through the run trace: the AI node classifies the payload, the run branches on that output, and a notify step hits the mock world.
3. Trigger the same workflow with `pay_inject_001` — a payload containing a prompt injection that instructs the AI to skip approval and issue a large refund. Show that the sensitive branch still lands in the approval queue, because the gate is enforced by the engine, not the prompt.
4. Approve a pending action and show the run resume and complete; reject one and show the run end cleanly.
5. Kill your worker process in the middle of a multi-step run (use `wf_slow_fulfillment`, which includes a delay). Restart it. Show the run resume from the last completed step — and run `scripts/duplication_check.py` to show the mock world's ledger contains exactly one copy of each side effect.
6. Compile a workflow from plain English (`nl_demo_001` in the eval set is a good script), show the generated draft, publish it, and trigger it live.
7. Send a compile request the node catalog cannot express (`nl_trap_001`) and show the compiler refuse with an explanation instead of inventing nodes.
8. Trigger `wf_runaway`, the seeded workflow that polls an order that never ships, and show the engine stop it at the max-step cap.
9. Run the provided smoke test and your NL-compiler eval and show the summaries.

## Acceptable Simplifications

- A single worker process is acceptable. Crash recovery must still work (kill it, restart it, runs resume), but multi-worker coordination is Stretch.
- Workflow shape is a list of nodes with if/else branching and backward jumps. A full arbitrary-DAG engine with parallel fan-out is Good To Have.
- Schedule triggers can have 1-minute granularity; catch-up for missed schedules is not required.
- AI nodes may use a free-tier or local model; engine tests may use a mocked model adapter. The NL-compiler eval needs a real (free-tier or local) model, but it is graded on structural assertions, not prose quality.
- Notify and action nodes only need to call the mock world. Real email/Slack integrations are Stretch.
- Authentication can be a simple demo token. Role-based access control is Good To Have.
- The console can be a simple page with tables and buttons. You may vibe-code or AI-assist it. UI polish is not graded.
- Approvals happen via API or a console button. Approval over email links is Stretch.

## Avoid These Mistakes

- Do not execute runs inside the HTTP request handler. Triggers enqueue; workers execute.
- Do not resume a crashed run by re-executing it from step 1. Completed steps must be skipped — the mock world's ledger will expose duplicate side effects, and `scripts/duplication_check.py` looks for exactly this.
- Do not lose queued or mid-flight runs on process restart. Run state lives in the database, not in memory.
- Do not treat webhook payloads as trusted input. `pay_inject_001` and `pay_inject_002` carry prompt injections aimed at your AI nodes.
- Do not enforce approval gates in the prompt. The engine must make sensitive nodes unreachable without an approval record, regardless of what any AI output says.
- Do not pass unvalidated AI output downstream. Enforce the node's JSON schema; retry once with the validation error; fail the step cleanly if it still does not conform.
- Do not let the NL compiler invent node types. Generated workflows must validate against the same rules as hand-written ones — the eval set contains impossible requests to test exactly this.
- Do not build Good To Have features before the Must Have workflow works end to end.

## Must Have

### 1. Load the Provided Data

- Load the provided node catalog, seed workflows, sample trigger payloads, and NL eval cases (`data/node_catalog.json`, `data/seed_workflows.json`, `data/sample_payloads.jsonl`, `data/nl_eval.jsonl`).
- Use any persistent store you prefer.
- Preserve workflow and payload IDs such as `wf_support_triage` and `pay_inject_001`, and the workflow-definition JSON shape — the demo flow and provided scripts reference them.

### 2. Workflow Definitions: Draft → Publish

- APIs to create, read, update, and list workflows. A workflow is a set of typed nodes from the catalog, connected by `next` pointers, with if/else branches on condition nodes.
- Two states: Draft (editable) and Published (frozen, runnable). Publishing creates an immutable version; every run records the version it executed.
- Publish-time validation rejects unknown node types, missing required parameters, and references to nonexistent nodes — each with a clear error.
- Backward jumps (loops) are legal — polling and retry patterns depend on them. The run caps in Must Have 7 are what bound them at execution time.

### 3. Triggers

- Webhook trigger: a per-workflow URL guarded by a secret (sent in the `X-Relay-Secret` header); requests with a missing or wrong secret are rejected. The payload becomes the run's input.
- Schedule trigger: cron-style recurring runs.
- Manual trigger API for testing and demos.

### 4. Deterministic Nodes and Data Flow

- Implement the deterministic node types from the catalog: `http_request`, `condition`, `delay`, and `notify` (which calls the mock world).
- Nodes reference earlier data with template expressions such as `{{trigger.body.email}}` and `{{nodes.classify.output.category}}`, resolved by the engine.
- Every node execution persists its resolved input and its output.

### 5. AI Nodes with Enforced Structure

- Implement the `ai` node: a prompt template plus a JSON schema for the output.
- The engine validates the model's output against the schema, retries once with the validation error included, and fails the step cleanly if the output still does not conform.
- Downstream `condition` nodes can branch on AI output fields — this is how AI decides a run's path.
- Record token usage per AI step in the run trace.
- Trigger payloads flow into AI prompts and are untrusted: the injection payloads must not produce unauthorized actions (see the guardrail rule in Must Have 7).

### 6. Durable Execution

- Runs execute asynchronously via a queue and worker; the API tier never blocks on a run.
- Every step is persisted as it completes. If the worker dies mid-run, a restarted worker resumes the run from the last completed step.
- Side-effect nodes (`notify`, `order_action`, and mutating `http_request` calls) send an idempotency key to the mock world; a resumed run must not duplicate a side effect. The mock world's ledger is the proof.

### 7. Approvals, Failures, and Limits

- Implement the `approval` node: the run pauses in a `waiting_approval` state; an approve API resumes it and a reject API ends it cleanly. The approval decision and decider are recorded.
- Sensitive nodes (marked `requires_approval` in the catalog, such as `order_action`) are unreachable without an approval record — enforced by the engine regardless of AI output.
- Per-node retries with exponential backoff for transient failures; a run that exhausts retries ends in `failed` with the error recorded.
- Run-level caps, enforced by the engine: max steps, wall-clock timeout, and a token budget across the run's AI nodes. `wf_runaway` must be stopped by the step cap.
- Runs use a documented status set: `queued`, `running`, `waiting_approval`, `succeeded`, `failed`, `cancelled`.

### 8. Natural-Language Workflow Compiler

- An endpoint that accepts a plain-English description and returns a Draft workflow using only catalog node types, with branches, parameters, and template wiring filled in.
- Generated drafts go through the same publish validation as hand-written ones.
- When a request needs a capability the catalog does not have, the compiler must refuse with an explanation instead of inventing node types — the eval set contains these traps.
- Build a one-command eval that runs every case in `data/nl_eval.jsonl` through the compiler and scores it on structural assertions (expected node types present, correct branching, valid publish, refusal on traps), reporting per-case results and overall accuracy.

### 9. Run Traces, Console, and Verification

- A trace API per run: every step with resolved input, output, attempts, timing, and token usage for AI steps — complete enough to debug a wrong branch decision after the fact.
- A simple console frontend: workflow list, run history, a run-detail view showing the step trace, and a pending-approvals view with approve/reject buttons.
- Your platform must pass `scripts/smoke_test.py`.
- Run `scripts/duplication_check.py` after the kill-and-resume demo and include its output (it inspects the mock world ledger for duplicate side effects).
- Provide automated tests for critical logic: publish validation, template resolution, schema enforcement, and idempotency.

## Good To Have

- An `agent` node: a bounded tool-using loop inside a single step (the model may call catalog-defined tools up to N iterations before returning a structured result).
- Parallel branches: fan out independent branches and join before a later node.
- Replay: re-run a failed run from its failed step with the original inputs.
- Cost accounting: price AI-node token usage from a price table, per-run and per-workflow spend, and a per-workflow monthly budget that blocks new runs when exhausted.
- AI failure triage: when a node fails repeatedly, an AI pass annotates the run with a probable cause and suggested fix.
- Live run view: stream step progress to the console over SSE or WebSocket instead of polling.
- Sub-workflows: a node that triggers another workflow and waits for its result.
- Role-based access control: builders, approvers, and viewers.
- Dead-letter view: failed runs with filters, bulk retry.

## Stretch

- Multiple concurrent workers with leases or locks so two workers never execute the same run, documented and demonstrated.
- Canary versions: route a percentage of triggers to a new workflow version and compare outcomes.
- NL editing: "add an approval step before the refund" compiles to a diff against an existing workflow rather than a new one.
- Approval over email: the mock world delivers an approval link that resumes the run.
- Real connector integrations (Slack, Gmail, Sheets) behind the same node interface.
- Organizations and multi-tenancy with isolated data and quotas.

## Technical Requirements

- Expose core functionality as RESTful APIs. The workflow-definition JSON shape and the core routes exercised by the smoke test must match `docs/API_CONTRACT.md`, because the provided scripts depend on them. Other routes may be reshaped if documented.
- Use a reliable persistent store for workflows, versions, runs, steps, approvals, and schedules. Run and step state must survive process restarts.
- Execution must be asynchronous: a queue (a database-backed queue is fine) and at least one worker process separate from the API's request path.
- Side-effect calls to the mock world must carry idempotency keys; recovery must be demonstrably exactly-once at the ledger.
- Every external call — model providers and the mock world — needs a timeout; a hung dependency must fail the step, not the engine.
- Keep the AI provider integration behind an adapter so it can be mocked in engine tests.
- Enforce approval gates and run caps in the engine, independent of model output.
- Provide automated tests for critical flows and guardrails.
- Do not depend on any specific programming language, framework, or hosted AI provider for the core design.

## Suggested API Surface

You may design your own API shape, but a complete solution should cover flows similar to these (the ones marked ✱ are exercised by the smoke test and defined in `docs/API_CONTRACT.md`):

- `POST /workflows` ✱
- `GET /workflows` ✱ / `GET /workflows/{workflowId}` ✱
- `POST /workflows/{workflowId}/publish` ✱
- `POST /workflows/compile`
- `POST /hooks/{workflowId}` (webhook trigger) ✱
- `POST /workflows/{workflowId}/trigger` (manual) ✱
- `GET /runs?workflowId=...`
- `GET /runs/{runId}` (full step trace) ✱
- `POST /runs/{runId}/cancel`
- `GET /approvals?status=pending` ✱
- `POST /approvals/{approvalId}/approve` ✱ / `POST /approvals/{approvalId}/reject`
- `POST /eval-runs` (NL compiler eval) or a documented command

## Suggested Milestones

You may plan your own schedule, but this staging keeps scope manageable:

1. **Core platform:** Data loading, workflow CRUD, publish validation, versioning.
2. **Engine v1:** Queue and worker, manual trigger, deterministic nodes, template resolution, per-step persistence, run traces.
3. **Durability and gates:** Crash resume, idempotency keys against the mock world, retries and backoff, run caps, approval nodes, webhook and schedule triggers.
4. **AI:** The `ai` node with schema enforcement and injection-safe handling of trigger payloads, then the NL compiler and its eval — they can share prompt and validation machinery.
5. **Console and verification:** Console, smoke test green, duplication check after a live kill-and-resume, NL eval accuracy, documentation, demo video.

The Must Have section above defines the minimum viable submission.

For a more detailed build path and FAQ, see `docs/IMPLEMENTATION_GUIDE.md`.

## Provided Starter Dataset

This capstone pack includes:

- A node catalog (`data/node_catalog.json`): every node type with its parameters, output shape, and flags such as `requires_approval`.
- Seed workflow definitions (`data/seed_workflows.json`): `wf_support_triage` (webhook → AI classify → branch → approval → refund → notify), `wf_expense_approval` (a deterministic approval flow the smoke test drives), `wf_slow_fulfillment` (for the kill-and-resume demo), and `wf_runaway` (a poll loop that never completes, for the caps demo).
- Sample trigger payloads (`data/sample_payloads.jsonl`), including two prompt-injection payloads aimed at AI nodes.
- A labeled NL eval set (`data/nl_eval.jsonl`): plain-English workflow descriptions with structural assertions, including trap cases the node catalog cannot express.
- A zero-dependency mock world server (`scripts/mock_world.py`): endpoints for email, chat messages, orders, refunds, and shipments, with a queryable ledger of every call received (the exactly-once proof), idempotency-key replay detection, and live failure injection for retry testing.
- A zero-dependency mock LLM provider (`scripts/mock_provider.py`, shared with the Prism pack) for deterministic engine tests.
- A smoke test (`scripts/smoke_test.py`) that checks the required API surface and run lifecycle.
- A duplication check (`scripts/duplication_check.py`) that inspects the mock world ledger after crash recovery.

You may extend the dataset, but you must document any added data and how it affects verification.

## Assessment Criteria

- **Must Have workflow - 60%:** Does the platform load the provided data, manage draft/published workflow versions, trigger from webhooks and schedules, execute deterministic and AI nodes durably with traces, pause and resume on approvals, enforce run caps, compile natural language to valid workflows, and demo through a simple console?
- **Durability and AI correctness under pressure - 25%:** Does a killed worker resume without duplicate side effects (ledger-verified)? Do injection payloads fail to bypass approval gates? Does schema enforcement catch malformed AI output? Does the compiler refuse impossible requests and pass the eval set? Is the runaway workflow stopped by the caps?
- **Engineering quality, documentation, and demo - 15%:** Is the code maintainable, is setup clear, are design decisions documented, and does the demo clearly show the core workflow?
- Good To Have and Stretch work can strengthen the project, but it should not compensate for missing Must Have functionality.

## Deliverables

1. Final functional product: the orchestration engine and a simple console.
2. README with setup instructions, API documentation, architecture, and design decisions (especially: the run state machine and how exactly-once recovery works).
3. Public GitHub repository link.
4. Seeded demo workflows and instructions to reproduce the demo with the mock world.
5. Verification report: smoke test output, the duplication check result after a live kill-and-resume, and NL compiler eval accuracy with per-case results.
6. Explainer video demonstrating the project, including a live worker kill and resume, and an injection payload handled safely.
