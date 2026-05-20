# CAPEv2 VM Execution Path and Lightweight CTF Fork Feasibility (KVM/libvirt + Windows-only)

## 1) Current code path for executing a sample on a VM

1. **Scheduler picks a pending task and (if needed) reserves a machine** via `Scheduler.find_pending_task_to_service()`, then creates an `AnalysisManager` thread and marks task running. (`lib/cuckoo/core/scheduler.py`)
2. **AnalysisManager prepares storage and task artifacts** (`init_storage`, `category_checks`, `store_file`) and validates sample integrity. (`lib/cuckoo/core/analysis_manager.py`)
3. **Machine lifecycle** happens inside `machine_running()` context:
   - Start VM via machinery manager (`start_machine`)
   - Run analysis
   - Optionally dump memory
   - Stop and release machine. (`lib/cuckoo/core/analysis_manager.py`)
4. **Guest orchestration** (`run_analysis_on_guest`):
   - Build options and auxiliary toggles
   - Instantiate `GuestManager`
   - Set guest status in DB
   - Upload analyzer/config, trigger guest execution, poll until completion. (`lib/cuckoo/core/analysis_manager.py`, `lib/cuckoo/core/guest.py`)
5. **Result collection** is handled by ResultServer (uploads from guest analyzer for logs/files/shots/etc). (`lib/cuckoo/core/resultserver.py`)

### Screenshot-specific behavior in the current flow

- If `cuckoo.machinery_screenshots` is enabled, **host-side screenshots are pulled periodically** by `GuestManager.wait_for_completion()` calling `AnalysisManager.screenshot_machine()`. (`lib/cuckoo/core/guest.py`, `lib/cuckoo/core/analysis_manager.py`)
- Machinery screenshot uses backend implementation; for libvirt-based backends this is implemented in `LibVirtMachinery.screenshot()`. (`lib/cuckoo/common/abstracts.py`)
- When machinery screenshots are enabled, ResultServer **discards guest-uploaded `shots/*`**, preventing duplicate screenshot sources. (`lib/cuckoo/core/resultserver.py`)

## 2) KVM/libvirt backend specifics

- `modules/machinery/kvm.py` is a thin specialization over `LibVirtMachinery` and sets `module_name = "kvm"`.
- It validates DSN and relies on shared libvirt start/stop/snapshot/screenshot/dump logic in `LibVirtMachinery`.
- Startup includes explicit snapshot/arch checking for KVM/QEMU via `check_snapshot_state()`. (`lib/cuckoo/core/startup.py`)

## 3) Feasibility: Lightweight CTF-oriented version

## Proposed target
- Support **only KVM + libvirt** backend.
- Support **only Windows guest analysis**.
- Keep minimal execution capability (run sample in VM).
- Disable/omit deep analytics/reporting pipeline.
- Keep screenshot capture during execution + one final screenshot.

## Feasibility assessment: **High**

Reasoning:
- VM execution path is already modular and backend-abstracted; KVM can be made the only loaded machinery with small startup/config gating.
- Host-side screenshot capability is already production-grade for libvirt (`LibVirtMachinery.screenshot`) and integrated at runtime through `machinery_screenshots` flow.
- Screenshot storage path (`storage/analyses/<id>/shots`) is already standardized and consumed by UI/report modules.

### What can be removed or constrained with low risk

1. **Backend reduction to KVM only**
   - Restrict machinery plugin loading / configuration to `kvm`.
   - Remove or stop importing non-kvm machinery modules.
2. **Windows-only simplification**
   - Enforce `machine.platform == windows` in scheduling or task admission.
   - Skip Linux analyzer paths and Linux-specific auxiliary modules.
3. **Analytics minimization**
   - Keep ResultServer and bare minimum artifact handling.
   - Disable processing/reporting chains and expensive integrations.
4. **Screenshot-focused output**
   - Keep `machinery_screenshots = yes` so host-side captures are canonical.
   - Save periodic screenshots from existing loop.
   - Add guaranteed final screenshot on task teardown (if VM still reachable before stop).

### Key engineering changes likely needed

1. **Admission/scheduling guardrails**
   - Reject non-file/non-url categories if desired (CTF mode policy).
   - Reject any task that cannot map to Windows KVM machine.
2. **Config profile (`ctf_mode`)**
   - New compact config preset that disables heavy modules and keeps only essentials:
     - scheduler, kvm machinery, resultserver, guest orchestration, screenshot capture.
3. **Final screenshot guarantee**
   - Add one last `screenshot_machine()` call in `AnalysisManager.perform_analysis()` teardown path (or just before `stop_machine`) guarded by exception handling.
4. **Output contract**
   - Define CTF-friendly output: task status + screenshot list + final screenshot pointer + optional raw dropped files.

## 4) Risks / caveats

- **Guest agent dependency remains**: even “no analytics” still depends on guest agent for task execution lifecycle unless you redesign execution method.
- **Windows analyzer imports many auxiliary components** by default when enabled in config; keep config hard-minimal.
- **Snapshot quality matters**: KVM path assumes valid libvirt snapshots and proper VM state; startup checks help but operator errors will still break runs.
- **Resultserver still required** for many artifacts and control-plane expectations unless you rework the architecture.

## 5) Practical implementation plan (incremental)

1. Add `ctf_mode` config and enforce:
   - machinery=`kvm`
   - platform=`windows` only
   - `machinery_screenshots=yes`
2. Disable processing/reporting modules by default under `ctf_mode`.
3. Add explicit final screenshot call in teardown.
4. Add minimal API endpoint/view returning screenshot timeline.
5. Smoke test with one known Windows sample and verify:
   - VM boot/revert
   - sample execution completion
   - periodic screenshots produced
   - final screenshot produced

## 6) Bottom line

A lightweight CTF-focused CAPE variant is very feasible with moderate code/config changes because the existing architecture already isolates machine orchestration, guest execution, and screenshot capture in reusable components. The biggest win is treating this as a **profiled runtime mode** rather than a hard fork: preserve core scheduler + analysis manager + kvm machinery + resultserver, and strip everything else by configuration and a few targeted guards.
