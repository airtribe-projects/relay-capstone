# Relay Capstone Pack

This repository contains a self-contained capstone package for building **Relay**, an AI-first workflow orchestrator with durable execution, human approval gates, AI decision nodes, and a natural-language workflow compiler. The AI lives in two places: `ai` nodes make schema-validated decisions inside runs, and the compiler turns plain-English descriptions into valid workflow definitions. Everything else is the engineering that makes those safe: crash recovery, exactly-once side effects, and approval gates the model cannot talk its way past.

The capstone is language agnostic. You may implement it in Node.js, Java, Python, Go, Ruby, or any stack you are comfortable with, as long as you satisfy the API, durability, and verification requirements.

## Contents

- `RELAY_PROBLEM_STATEMENT.md` - capstone problem statement.
- `docs/IMPLEMENTATION_GUIDE.md` - suggested build order, demo scenarios, and FAQ.
- `docs/API_CONTRACT.md` - language-agnostic API contract, the fixed routes the smoke test drives, and error shapes.
- `docs/DATA_MODEL.md` - suggested entities, relationships, and the correctness guarantees (idempotency, crash windows).
- `docs/EVALUATION_GUIDE.md` - how your engine is verified: smoke test, kill-and-resume drill, NL eval, failure drills.
- `data/node_catalog.json` - the node and trigger type registry; the boundary for validation and the NL compiler.
- `data/seed_workflows.json` - four seed workflows: AI triage, expense approval, slow fulfillment (kill-and-resume), and a runaway poll loop (caps).
- `data/sample_payloads.jsonl` - annotated trigger payloads, including two prompt-injection cases.
- `data/nl_eval.jsonl` - 15 labeled compiler cases (12 buildable, 3 traps the catalog cannot express).
- `scripts/mock_world.py` - zero-dependency mock external world: email, chat, orders, refunds, shipments, an idempotency-aware call ledger, and live failure injection.
- `scripts/duplication_check.py` - reads the world's ledger and verifies exactly-once side effects after a crash drill.
- `scripts/smoke_test.py` - checks your platform against the required API contract and run lifecycle.
- `scripts/mock_provider.py` - zero-dependency OpenAI-compatible mock LLM (shared with the Prism pack) for engine tests.
- `scripts/validate_pack.py` - validates that the package data is parseable and internally consistent.

## Optional Local Utilities

All scripts use only Python standard library modules (Python 3.9+). From this folder:

```bash
python3 scripts/validate_pack.py
```

Start the mock world:

```bash
python3 scripts/mock_world.py --port 9210
```

Poke at it — this is what your notify and action nodes will call:

```bash
# a side effect with an idempotency key
curl -s -X POST http://localhost:9210/email/send \
  -H "Idempotency-Key: demo-1" -H "Content-Type: application/json" \
  -d '{"to": "maya@example.com", "subject": "hi", "message": "hello"}'

# send it again with the same key - replayed, not repeated
curl -si -X POST http://localhost:9210/email/send \
  -H "Idempotency-Key: demo-1" -H "Content-Type: application/json" \
  -d '{"to": "maya@example.com", "subject": "hi", "message": "hello"}' | grep -i replayed

# the ledger records both calls, one executed, one replayed
curl -s http://localhost:9210/admin/ledger

# business rules bite: refunds above the order total are rejected
curl -s -X POST http://localhost:9210/orders/ord_2001/refund -d '{"amount_usd": 5000}'
```

Inject failures live to test your retries and timeouts:

```bash
curl -s -X POST http://localhost:9210/admin/config -d '{"mode": "down"}'        # everything 503
curl -s -X POST http://localhost:9210/admin/config -d '{"fail_rate": 0.3}'      # 30% errors
curl -s -X POST http://localhost:9210/admin/config -d '{"latency_ms": 3000}'    # slow world
curl -s -X POST http://localhost:9210/admin/config -d '{"mode": "ok", "fail_rate": 0, "latency_ms": 0}'
```

Verify your platform once it is running (API, worker, and mock world up):

```bash
python3 scripts/smoke_test.py --url http://localhost:8080 --token <demo-token>
python3 scripts/duplication_check.py --url http://localhost:9210     # after the kill-and-resume drill
```

## Language-Agnostic Expectations

Your implementation should provide:

- Workflow CRUD with draft → publish versioning and catalog-based validation, matching the definition shape in `data/seed_workflows.json`.
- Webhook (secret-guarded), schedule, and manual triggers.
- Durable asynchronous execution: queue + worker, per-step persistence, crash resume, idempotency keys on side effects.
- `ai` nodes with JSON-schema-enforced output and per-step token accounting.
- `approval` nodes that pause runs, and engine-enforced gating of sensitive nodes.
- Run caps: max steps, wall-clock timeout, AI token budget.
- A natural-language compiler that emits valid drafts or refuses, with a one-command eval over `data/nl_eval.jsonl`.
- Full run traces and a lightweight console (workflows, runs, trace view, approvals).
- A model-provider adapter that can be faked in tests.

The console can be vibe-coded or AI-assisted. It does not need to be visually complex, but it should let you demonstrate a run trace, a pending approval, and the decision buttons.

## Recommended Reading Order

1. Read `RELAY_PROBLEM_STATEMENT.md`.
2. Follow `docs/IMPLEMENTATION_GUIDE.md` for the build path.
3. Use `docs/API_CONTRACT.md` and `docs/DATA_MODEL.md` while designing your implementation.
4. Use `docs/EVALUATION_GUIDE.md` before running the verification scripts and writing your report.

## Correctness Notes

The pack is designed to expose the classic orchestrator failure modes: engines that re-run completed side effects after a crash, side effects sent without idempotency keys, approval gates enforced only in the prompt, AI output trusted without schema validation, loops with no cap, and compilers that invent node types. The ledger, the injection payloads, the runaway workflow, and the trap eval cases exist so you can prove your implementation does not have them.
