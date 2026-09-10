# loci-coordination-bus

A secure, local-first coordination bus for multiple coding agents (Claude, Codex, or any other
tool-using LLM agent) working on the same codebase at once. It answers one narrow question:
**how do two or more independent agents hand off tasks, claim leases, request review, and pass
messages to each other without any of them being able to grant themselves permissions, run
commands, or trust unverified content as an instruction?**

It has no dependency on any specific project — no imports from a RAG pipeline, a memory engine,
or any other application code. It is pure coordination plumbing: an authenticated event log plus
a small task-lease/review state machine on top of it, backed by one local SQLite file.

## What it does

- **`SecureEventBus`** — an append-only, HMAC-signed event log. Every event (`task.created`,
  `task.claimed`, `review.approved`, `agent.message`, ...) is validated, signed, and checked for
  replay/tampering before it's accepted. Agents authenticate with a per-agent secret registered
  at startup; there is no way to impersonate another agent or widen your own scope from inside an
  event's payload.
- **`TaskBoard`** — a task lifecycle (`ready → claimed → completed → review → approved`) built on
  top of the bus. Claims are time-leased (so a crashed agent's task becomes claimable again),
  completion requires a real test result and a full Git commit SHA, and only an agent explicitly
  registered as a reviewer can approve.
- **`coordination_bus/mcp_server.py`** — an MCP (Model Context Protocol) stdio server exposing
  the bus as tools (`coordination_create_task`, `coordination_claim_task`,
  `coordination_send_message`, etc.), so any MCP-capable agent can use it without a custom
  integration. The process binds one agent identity at startup from environment variables —
  agents never supply their own identity or secret as a tool argument.

## What it deliberately does not do

- **It never executes anything.** Payloads are inert data (JSON), capped at 16 KB, and any field
  that looks authority-bearing (`command`, `shell`, `exec`, `credentials`, `secret`, `token`,
  `api_key`, `system_prompt`, ...) is rejected outright before the event is even signed.
- **It never grants permissions.** An agent's project/operation scope is fixed at registration
  time by whoever configures the process (a human, or a trusted setup script) — not by anything
  arriving over the bus.
- It is not a general message queue, not a distributed system, and not a replacement for your
  own review process — `TaskBoard.approve()` still requires a real reviewer to call it.

## Security model

Every event must pass, in order: schema validation → HMAC-SHA256 signature verification →
project/operation authorization → expiry/sequence/idempotency checks → relative-path validation
(no absolute paths, no `..`) → payload authority-field rejection. See
[`docs/SECURE-BUS.md`](docs/SECURE-BUS.md) for the full boundary description, the review/conflict
flow, and guidance for moving beyond a single-machine SQLite deployment.

## Install

```bash
pip install -e .              # core bus + task board (stdlib only, no dependencies)
pip install -e '.[mcp]'       # + the MCP server (adds the `mcp` package)
pip install -e '.[dev]'       # + pytest, for running the test suite
```

## Quick start (library usage)

```python
from coordination_bus import SecureEventBus, TaskBoard

bus = SecureEventBus("coordination.sqlite3")
bus.register_agent("planner", bus.new_secret(), projects={"my-project"},
                    operations={"create_task"})
bus.register_agent("worker", bus.new_secret(), projects={"my-project"},
                    operations={"claim_task", "complete_task", "request_review"})
bus.register_agent("reviewer", bus.new_secret(), projects={"my-project"},
                    operations={"approve_review"}, reviewer=True)

board = TaskBoard(bus)
task_id = board.create(agent="planner", project="my-project", title="Add feature X",
                        base_commit="a" * 40, allowed_paths=("src/",))
board.claim(agent="worker", task_id=task_id)
board.complete(agent="worker", task_id=task_id, commit_sha="a" * 40, tests_passed=True)
board.request_review(agent="worker", task_id=task_id)
board.approve(reviewer="reviewer", task_id=task_id)
```

Each `register_agent` secret should be generated once (`SecureEventBus.new_secret()`) and stored
outside version control — see the MCP section below for how the server process picks these up
from the environment instead of hardcoding them.

## Quick start (MCP server, for agent-to-agent use)

1. Generate a secret per agent identity and store them somewhere outside the repo (a local
   `.loci/agent.env` file, a secrets manager — never commit them).
2. Set the environment variables the server reads at startup:
   - `LOCI_AGENT_ID` — this process's own agent identity
   - `LOCI_AGENT_SECRET` — that identity's secret (32+ characters)
   - `LOCI_PROJECTS` — comma-separated project names this identity may act in
   - `LOCI_OPERATIONS` — comma-separated operations this identity may perform
   - `LOCI_COORDINATION_DB` — path to the shared SQLite file (defaults to
     `.loci/coordination.sqlite3`)
   - `LOCI_PEERS_JSON` — a JSON array registering the *other* agents this process needs to know
     about (each: `agent_id`, `secret`, `projects`, `operations`, optional `reviewer`), so
     message delivery and cross-agent authorization work
   - `LOCI_REVIEWER` — `true` if this identity is allowed to approve reviews
3. Run one process per agent identity, all pointed at the same `LOCI_COORDINATION_DB`:
   ```bash
   scripts/run_coordination_mcp.sh
   ```
4. Point your MCP-capable agent at the server — `.mcp.json` in this repo is a generic example
   (adjust the `command` for your platform; a WSL-hosted server invoked from Windows, for
   instance, needs a `wsl.exe -e bash -lc ...` wrapper instead of a bare `bash` command).

## Running the tests

```bash
pip install -e '.[dev,mcp]'
pytest -q
```

## Project status

This is the first coordination slice: local-first, single-SQLite-file, one bus per project. It
does not yet include a real-time transport (SSE/WebSockets), a durable multi-host event log, or
mTLS/short-lived-credential support for remote agents — see the "Real-time deployment" section of
[`docs/SECURE-BUS.md`](docs/SECURE-BUS.md) for what a production, multi-host deployment would
need to add on top of this foundation.
