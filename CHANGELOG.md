# Changelog

All notable changes to the AURIORA Firmware Style Guide are documented in this file. Released versions are tagged in version control.

## 0.4.0 - 2026-09-12

- Module Control Interface (MCI) firmware rules (§14.3): implement the operations once behind an internal interface and let transport adapters do framing only, so no handler branches on which transport a request arrived over; answer identity before any configuration, from immutable storage, independent of transport and cable; keep the declared capability set in one place and generate dispatch checks from it, returning a defined "unsupported" error rather than silent success; model `IDLE → CONFIGURED → ARMED → RUNNING → COMPLETE` as an explicit state machine that reports *why* a transition was refused; make entering `ARMED` mean every precondition was actually checked, and report state rather than acknowledgment; never provide a control operation documented as the way to start several Modules together — the deterministic instant belongs to SYNC (§14.2); stage, verify against a whole-object integrity value, then commit, leaving a previously committed object untouched on failure; expose the stored inventory with content identity so a host can skip unnecessary transfers; keep configuration, SYNC binding and versions readable; bound every host-startable operation; and host-test every state × operation pair, not the happy path. §14.3 also notes that MCI is the Module-to-host control path, unrelated to the Managed Unit API of §14.1. The interface itself is defined by AES 0.7.0 (`AES-MCI-001` to `AES-MCI-005`).

## 0.3.0 - 2026-09-11

- Module Synchronization Interface (SYNC) firmware rules (§14.2): capture the SYNC IN edge with a hardware timestamp (input capture, edge interrupt with free-running counter, DMA or programmable I/O — never polling) and convert it to the Module's sample clock where one exists; keep the ISR minimal and decide in the owning task; model arming as an explicit state machine in which every unaccepted event is a logged ignored event with a reason, never an implicit restart, stop or queue; treat the SYNC IN action, SYNC OUT source, delay and post-completion mode as validated, versioned, readable runtime configuration; realize a configured delay on a hardware timer from the captured edge; generate SYNC OUT from the bound internal event through a hardware-timed output with characterized pipeline latency compensated, one source per output by default; never emit or interpret pulse patterns; log received, transmitted and ignored events with timestamp, sample index, state, run identifier, reason and local counters; host-test every state × event pair and measure latency and jitter on hardware. §14.1 clarifies that its hardware synchronization signal is Unit-to-host within one Module and unrelated to SYNC. The interface itself and its behavioral rules are defined by AES 0.6.0 (`AES-SYNC-001` to `AES-SYNC-004`, `docs/interfaces/sync.md`).

## 0.2.0 - 2026-07-14

- Managed Unit API rules (§14.1): a Managed Unit's public API must expose capabilities/operations/status/errors at a high level (no register-level internals in the host contract); the API must be versioned with defined framing, identifiers, status/error codes, timeouts and integrity checking; unknown commands and unsupported versions must fail safely; firmware identity and API version must be readable; the Unit must implement a disabled→starting→ready→fault lifecycle with `UIF_READY` gating valid transactions; and the host contract must not rely on software-latency timing where a hardware sync signal exists.

## 0.1.0 - 2026-07-13

First release.

- Design philosophy and a maturity-scaling model: safety rules always; reproducible builds, update integrity, watchdog and production diagnostics bind Released firmware; architecture, project layout, host tests and CI are the model firmware converges on as it matures.
- Firmware architecture: layered model, module responsibilities, active objects, policy/mechanism separation, initialization and lifecycle.
- Project structure, coding style, API design and error handling guidance.
- State machines, interrupt design, RTOS guidelines (for firmware that uses an RTOS), memory management (placement guidance for dynamic allocation) and concurrency rules.
- Logging and diagnostics, configuration management, communication interfaces and power management guidance.
- Firmware update, security and testing requirements scaled to maturity and risk.
- Documentation requirements aligned with the AURIORA Documentation Standard.
- A compact code review checklist (every change / concurrency and resources / toward Release); self-review explicitly valid.
- Requirement language aligned with AES (MUST/SHOULD/MAY; a skipped SHOULD needs no documented exception).
- Platform-independent policy: no references to specific languages, compilers, SDKs or vendors.
- CC BY-SA 4.0 license.
