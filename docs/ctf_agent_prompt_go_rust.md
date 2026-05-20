# Prompt for Coding Agent: Build a Minimal CTF-Focused Sandbox Variant (Go/Rust)

You are a senior systems engineer. Implement a **minimal, production-usable CTF-oriented sandbox service** in **Golang or Rust** (pick one and state why), inspired by CAPE/Cuckoo execution flow, but intentionally narrow in scope.

## Objective
Build a lean service that only:
1. Accepts a Windows sample submission.
2. Executes it in a Windows VM via **KVM + libvirt only**.
3. Captures **periodic screenshots during execution**.
4. Captures **one final screenshot at teardown**.
5. Returns a compact run record (status + screenshot list + timings + basic errors).

No deep malware analytics, no signature engines, no heavy reporting pipeline.

---

## Hard Constraints (Must Follow)

### Platform / Backend
- Support **only Linux host + KVM/libvirt** backend.
- Reject any non-libvirt backend configuration at startup.
- Use libvirt snapshot restore/revert workflow before each run.

### Guest Scope
- Support **only Windows guests**.
- Reject non-Windows VM profiles at validation time.

### Functional Scope
- Implement only the minimal control-plane for:
  - task queueing
  - machine reservation/locking
  - run lifecycle
  - screenshot collection
  - result persistence
- Do **not** implement:
  - static analysis
  - behavioral parsing/signatures
  - network analytics
  - YARA/Suricata/AV integrations

### Screenshot Behavior
- Periodic host-side screenshot capture (libvirt domain screenshot API) during run.
- Optional simple dedupe (RMS or hash threshold) to reduce duplicates.
- Always attempt **one final screenshot** immediately before VM stop/release.
- Never fail the whole task solely because screenshot capture failed.

---

## Architecture Requirements

Create a small modular architecture with these components:

1. **API service**
   - `POST /tasks` (submit sample path/hash + timeout + vm profile)
   - `GET /tasks/{id}` (status)
   - `GET /tasks/{id}/screenshots` (ordered screenshot metadata)
   - `GET /tasks/{id}/artifacts/{name}` (optional image fetch)

2. **Scheduler/Worker loop**
   - Pull pending task
   - Select compatible Windows KVM machine
   - Lock machine
   - Execute task lifecycle
   - Unlock machine

3. **Libvirt VM controller**
   - Connect via DSN
   - Lookup domain
   - Revert snapshot
   - Start/ensure running
   - Screenshot capture
   - Stop/destroy on teardown

4. **Guest runner (minimal)**
   - Trigger sample execution in guest (assume existing guest agent endpoint or command channel)
   - Poll run status until complete/failed/timeout

5. **Artifact store**
   - Local filesystem layout:
     - `storage/tasks/<task_id>/meta.json`
     - `storage/tasks/<task_id>/shots/0001.jpg ...`
     - `storage/tasks/<task_id>/run.log`

6. **State store**
   - Lightweight SQLite (preferred) with tables:
     - `tasks`
     - `machines`
     - `task_events`
     - `screenshots`

---

## Data Model (Minimum)

### Task
- `id`
- `submitted_at`, `started_at`, `finished_at`
- `status` (pending|running|completed|failed|timeout)
- `sample_ref` (path/hash)
- `vm_name`
- `timeout_sec`
- `error_message`

### Screenshot
- `id`
- `task_id`
- `seq`
- `kind` (periodic|final)
- `path`
- `captured_at`
- `width`, `height` (if available)

### Machine
- `name`
- `platform` (must be windows)
- `locked` bool
- `snapshot`
- `last_heartbeat`

---

## Execution Flow (Required)

For each task:
1. Validate task input and VM compatibility.
2. Create task row `pending`.
3. Scheduler picks task and locks a matching machine.
4. Revert VM to configured snapshot.
5. Start VM and wait until reachable.
6. Start sample execution via guest runner.
7. While running:
   - every N seconds capture screenshot via libvirt
   - store metadata
8. On completion/timeout/error:
   - attempt final screenshot (`kind=final`)
   - stop VM
   - unlock machine
   - mark final task status

---

## Non-Functional Requirements

- Strong error handling with clear typed errors.
- Context cancellation/timeouts for all VM and guest operations.
- Idempotent teardown (safe to call twice).
- Structured logging (JSON logs).
- Minimal dependencies.
- Clear configuration file with strict validation.

---

## Security / Safety Guardrails

- Do not allow path traversal in sample/artifact paths.
- Enforce max file size and timeout limits.
- Sanitize all operator/user-supplied strings logged or stored.
- Restrict screenshot/artifact serving to task directory root.

---

## Deliverables

1. Working codebase in **Go or Rust**.
2. `README.md` with:
   - architecture overview
   - setup instructions (libvirt prerequisites)
   - run instructions
   - API examples (`curl`)
3. `config.example.{yaml|toml}`
4. DB schema migration/init script.
5. Basic tests:
   - unit tests for scheduler state transitions
   - unit tests for screenshot naming/order
   - integration test with mocked libvirt/guest runner
6. A short `CTF_SCOPE.md` explaining what is intentionally out-of-scope.

---

## Acceptance Criteria (Definition of Done)

- Can submit a Windows sample task and receive a terminal status.
- Task produces zero or more periodic screenshots plus one attempted final screenshot.
- Machine lock is always released even on failure/timeout.
- No references to non-KVM backends in runtime paths.
- No analytics pipeline code included.
- All tests pass.

---

## Implementation Guidance

If choosing **Go**:
- Suggested libs: `gin`/`chi`, `database/sql` + `sqlite`, libvirt Go bindings, `zap`/`zerolog`.

If choosing **Rust**:
- Suggested libs: `axum`, `sqlx` + SQLite, libvirt bindings/crate, `tracing`, `tokio`.

Prefer simple, explicit code over abstraction-heavy frameworks.

---

## Output Format Required from You (the coding agent)

When done, provide:
1. Summary of implemented modules.
2. Explicit file tree.
3. Config and startup commands.
4. Test commands + results.
5. Known limitations.
