# AURIORA Firmware Style Guide

**Document ID:** AFSG
**Version:** 0.3.0
**Status:** Normative
**Complements:** AURIORA Engineering Standard (AES)
**Language:** English

## About This Guide

This is the official firmware development guide for all AURIORA embedded systems. It complements the AURIORA Engineering Standard: AES defines *what* must be true about AURIORA engineering work (identity, interfaces, maturity levels, releases), while this guide defines *how* firmware should be designed, structured, implemented, documented and maintained across all AURIORA hardware platforms.

It is an engineering handbook, not a programming tutorial. It is deliberately independent of specific microcontrollers, SDKs, compilers and vendors: it applies equally to RP-series, ESP32, STM32, Nordic, Microchip and future platforms. Architecture and discipline age well; toolchains do not.

Requirement language (aligned with AES):

- **MUST** — mandatory when applicable. Equivalent to `SHALL` in AES.
- **SHOULD** — recommended default; engineering judgment may justify another approach. Skipping a SHOULD requires no documented exception.
- **MAY** — optional improvement.

Where this guide and AES conflict, AES prevails. Deviating from an applicable MUST needs a concise note in the design notes; formal exception records are reserved for Released artifacts and platform-wide deviations ([AES-GOV-003](https://github.com/auriora-org/auriora-engineering-standard/blob/main/STANDARD.md#aes-gov-003-honest-deviations)).

### Maturity Scaling

Requirements scale with the [AES maturity level](https://github.com/auriora-org/auriora-engineering-standard/blob/main/STANDARD.md#3-maturity-model):

- **Always (all maturity levels):** the safety rules — safe output states at boot and fault, validated external input on anything that can energize hardware, Module-level limits enforced regardless of what peripherals or Units claim.
- **Released only:** release-grade obligations — reproducible builds with provenance, update integrity and rollback for field-updatable devices, watchdog in production images, production diagnostics, complete interface documentation, hardware testing of release candidates.
- **Everything else** is the recommended default (SHOULD). Experimental firmware — a sensor spike, a test rig, a throwaway bring-up image — may be a single `main.c` with a README; it just says so honestly. The architecture in Sections 2–3 describes where firmware should *converge* as it moves toward Release, not a template every repository must instantiate on day one. Scale test and CI expectations to maturity and risk.

---

## 1. Design Philosophy

AURIORA firmware is written to be read, debugged and extended for years — usually by someone other than the original author, often without the original hardware on the desk.

- **Simplicity.** The simplest design that meets requirements is the correct design. Complexity MUST be justified by a real requirement, not an anticipated one. If a mechanism needs a diagram to explain, first ask whether it needs to exist.
- **Readability.** Code is read far more often than written. Optimize for the reader: clear names, small functions, obvious control flow. Cleverness that saves lines but costs comprehension MUST be avoided.
- **Reliability.** Firmware MUST behave correctly under worst case: full queues, failed peripherals, corrupted input, power loss at any instant. Every failure path is part of the design, not an afterthought.
- **Maintainability.** A module MUST be understandable from its public header and module documentation alone, without reading its internals. Non-obvious decisions MUST be documented where they apply.
- **Deterministic behaviour.** Prefer bounded loops, bounded queues, fixed-size buffers and static allocation. Timing-critical paths MUST have known worst-case execution characteristics. Avoid designs whose behaviour depends on unbounded input or heap state.
- **Explicitness over cleverness.** State, dependencies and side effects MUST be visible in the code: explicit state machines, explicit initialization order, explicit ownership. No hidden coupling through globals, singletons or magic macros.
- **Predictability.** The same input in the same state MUST produce the same behaviour. Uniform module structure and API shape across the codebase means a developer who has read one module can navigate all of them.
- **Long-term support.** Firmware outlives its toolchain. Isolate vendor and SDK dependencies behind project-owned interfaces so a platform migration changes one layer, not the application.
- **Open source philosophy.** Firmware MUST be buildable from its published repository with documented, preferably free toolchains. Write code as if strangers will read it — because they will.

---

## 2. Firmware Architecture

Architecture matters more than style. Most firmware becomes unmaintainable not through bad formatting but through missing structure: no layers, no module boundaries, no clear responsibilities. AURIORA firmware converges on one architectural model so that developers can move between projects years apart and still know where everything lives.

Scaling: firmware of real complexity — anything with concurrent subsystems, multiple drivers or a maintained lifespan — SHOULD follow this model, and Released firmware is expected to. Simple firmware (a test image, a single-sensor logger) may be a handful of files; do not manufacture layers that have nothing in them.

### 2.1 Layered Model

Organize firmware in layers with dependencies pointing strictly downward:

| Layer | Responsibility | Examples |
|---|---|---|
| **Application** | Product behaviour: coordination, workflows, protocol handling | agents/tasks, command dispatch, state machines |
| **Services** | Reusable, hardware-independent building blocks | protocol codecs, calibration store, logging, executors |
| **Drivers** | One external device or peripheral each, behind a clean API | sensor drivers, LED drivers, flash drivers |
| **HAL / Port** | Thin project-owned wrapper over MCU peripherals, RTOS config, board pinout | pin/bus configuration, RTOS port, clock setup |
| **Platform** | Vendor SDK, RTOS kernel, third-party libraries — vendored, never modified in place | SDK, kernel, vendor libs |

- **Downward dependencies only.** A layer MUST NOT depend on a layer above it, and SHOULD NOT skip layers. The application SHOULD NOT touch registers or SDK calls directly; that is what drivers and the HAL are for.
- **Vendor isolation.** Vendor SDK and RTOS APIs MUST NOT leak into the application layer. If the application needs a platform service, wrap it in a project-owned interface. Vendored third-party code lives in its own directory, unmodified; local changes go into the port layer.
- **Upward communication** happens through return values, callbacks registered at initialization, or messages — never by a lower layer calling application code directly.

### 2.2 Modules and Responsibilities

- **One module, one responsibility.** A module is a directory with a public header, its sources and its build definition. Everything not in the public header is private. If a module's purpose cannot be stated in one sentence, split it.
- **Active objects.** Concurrent subsystems SHOULD be structured as active objects: a module that owns a task (or main-loop slot), its own state and a message queue. Other modules interact by sending messages or calling its thread-safe API — never by reaching into its data.
- **Policy vs. mechanism.** Decision logic (validation, timeout policy, health rules, state transitions) SHOULD be separated from hardware-touching code into pure functions with no hardware dependencies. Pure policy code is testable on a host PC without target hardware — this separation is what makes the test strategy of Section 18 possible.
- **Large modules** SHOULD be split by aspect into multiple source files behind one public header (e.g. `*_runtime`, `*_state`, `*_commands`) rather than growing thousand-line files.
- **Reusable modules.** Anything useful to more than one project (drivers, protocol code, utilities) SHOULD NOT depend on application specifics and SHOULD be written as if it will be extracted into a shared library — because eventually it will. Not every module needs to be independently extractable; invest where reuse is plausible.

### 2.3 Dependencies, Initialization and Lifecycle

- **Dependency injection.** Modules SHOULD receive their dependencies (bus handles, configuration, callbacks) at initialization rather than discovering them through globals. This makes dependencies visible and modules testable.
- **Explicit initialization order.** Startup MUST initialize in dependency order: platform → HAL → drivers → services → application. Startup logic SHOULD live in a dedicated startup module, not scattered through `main`. `main` itself stays short: initialize, start the scheduler or main loop, never return.
- **Fail loudly at boot.** If a required component fails to initialize, the firmware MUST NOT limp on silently — report it (LED pattern, log, health state) and enter a defined degraded or safe state.
- **Defined lifecycle.** Every module with state SHOULD have a defined lifecycle (`init` → operational → optional `deinit`) and SHOULD reject use before initialization in debug builds.
- **Architecture documentation.** Non-trivial firmware in Active Development or beyond SHOULD document its architecture — layers, modules, tasks, data flow — and Released firmware MUST (see Section 19). A section in `docs/design-notes.md` is a perfectly good home; a separate architecture document is needed only when the project outgrows it. If the code and the document disagree, one of them is wrong — fix it in the same change.

---

## 3. Project Structure

AURIORA firmware repositories of real size SHOULD use one recognizable layout. Names may vary per build system; keep the structure recognizable. Small firmware may collapse this to `src/` + `README.md` — the full tree is where projects grow *to*, not a requirement to start from.

```
project/
├── src/                  # all project source code
│   ├── app/              # application layer (tasks/agents, workflows)
│   ├── services/         # hardware-independent services
│   ├── drivers/          # device drivers, one directory per device
│   ├── hal/              # board pinout, bus setup, hardware config
│   ├── config/           # compile-time configuration headers
│   └── main.c(pp)        # entry point: init + start scheduler
├── port/                 # RTOS/SDK port and configuration files
├── lib/                  # vendored third-party code (SDK, kernel) — unmodified
├── test/                 # host-based unit tests and mocks
├── tools/                # developer and CI scripts
├── docs/                 # architecture and module documentation
├── README.md
└── CHANGELOG.md
```

- Each module directory SHOULD contain its public header(s), sources and build definition, so a module can be understood as one unit.
- Public headers define the API; internal headers (`*_internal.h`) MUST NOT be included from outside the module.
- Build outputs MUST stay out of version control. The build MUST be reproducible from a clean checkout with documented steps for anything beyond Experimental.
- Board support (pin assignments, bus mappings, clocks) MUST be concentrated in the HAL/config area, never scattered through application code. Supporting a board variant means changing one place.
- Tests mirror the source structure: one test file per module, mocks for the layers below (Section 18).

---

## 4. Coding Style

The goal is consistency, not a particular aesthetic. Uniform code reads as one codebase, not an anthology.

- **One style per codebase.** Each repository SHOULD have a recorded code style — a committed formatter configuration is the cheapest mechanism — and code follows it consistently. Which style is chosen matters less than applying it everywhere. Style debates end at the config file.
- **File naming.** Files and directories MUST use `snake_case`, named after the module they implement (`sensor_agent.c`, `protocol_parser.h`). One module's files share its name as prefix.
- **Naming conventions.** Names MUST be English, pronounceable and specific. Choose one convention per identifier class and keep it project-wide:
  - Functions and variables: consistent case (e.g. `snake_case`); public API functions carry their module prefix (`protocol_parse()`, `led_driver_set()`).
  - Constants and macros: `UPPER_SNAKE_CASE`, prefixed by module (`PROTOCOL_MAX_FRAME_LEN`). Unprefixed short macros pollute the global namespace and MUST be avoided in headers.
  - Types: consistent scheme (e.g. `snake_case_t` for C typedefs, `PascalCase` for classes); enum members carry the enum's prefix.
  - Booleans read as assertions (`is_ready`, `has_frame`); functions read as verb phrases (`start_sampling()`), except pure predicates.
- **No magic numbers.** Every non-obvious literal MUST be a named constant with its unit in the name or documentation (`TIMEOUT_MS`, `VBUS_MV`).
- **Formatting.** Keep lines to a stated limit (100 columns SHOULD be the default), indentation uniform, braces always used — even for single-statement bodies. Whitespace separates logical steps within a function.
- **Functions.** Short, one job, few parameters. A function that needs a scroll wheel SHOULD be decomposed. Deep nesting SHOULD be flattened with early returns.
- **Macros.** Prefer typed constants, inline functions and enums over function-like macros. Macros are acceptable for conditional compilation and genuinely token-level work; anything with control flow inside SHOULD NOT be a macro.
- **Comments** explain *why*, not *what*. Commented-out code MUST NOT be committed — version control remembers. Every public API element gets a documentation comment (Section 19); internals get comments only where the code cannot speak for itself.

---

## 5. API Design

A module's public header is a contract. Design it deliberately — it is the hardest thing to change later.

- **Minimal surface.** Expose only what callers need. Everything else is private by default: `static` in C, private members or internal headers otherwise. Growing an API later is easy; shrinking one is a breaking change.
- **Opaque state.** Module state SHOULD be hidden behind opaque handles or accessor functions. Callers that can reach into a struct will, and every field they touch becomes API.
- **Consistent shape.** APIs across the codebase follow one pattern: `module_init(config)` → operations → optional `module_deinit()`. Configuration goes in a config struct with sane defaults, not a growing parameter list. The same concept has the same name everywhere (`init`, `read`, `write`, `start`, `stop`).
- **One responsibility per function.** A function either performs an action or answers a question; avoid functions whose behaviour is steered by mode flags.
- **Error reporting.** Every fallible public function MUST report failure through a status return (project-wide status/error enum SHOULD be used). Output values go through out-parameters or result structs — never through the status channel, and never half-written on failure.
- **No hidden side effects.** An API call MUST NOT silently reconfigure shared resources (bus speed, clocks, pin modes) that other modules depend on. Shared-resource ownership is declared, not assumed.
- **Compatibility.** Public APIs used across repositories MUST be versioned. Within a repository, changing an API means updating every caller in the same change — no deprecated corpses left behind.

---

## 6. Error Handling

Errors are part of the normal flow of an embedded system, not an exceptional one. Every failure path is code you shipped.

- **Check what can fail.** Every call that returns a status MUST be checked and handled or explicitly propagated. Ignoring a return value is a review defect; deliberately ignoring one is documented at the call site.
- **Fail fast, recover deliberately.** Detect errors at the point of occurrence, propagate them to the level that has enough context to decide, and take a deliberate action: retry with bound, degrade, reset the subsystem, or escalate.
- **Assertions** guard programmer errors — broken invariants, impossible states, contract violations by callers. They MUST NOT be used for runtime conditions that can legitimately occur (bus timeouts, bad input). Assertions SHOULD stay enabled in production builds with a handler that records the location and resets safely.
- **Runtime checks** guard the outside world: all input from communication interfaces, storage and sensors MUST be validated (range, length, CRC) before use. The firmware trusts nothing it did not compute itself.
- **Defensive programming** with judgment: validate at trust boundaries (public APIs, external input), don't re-validate the same data at every internal layer — that hides the actual contract.
- **Fatal errors.** Unrecoverable conditions MUST route through a single fatal-error handler that brings hardware outputs to a safe state, records the cause (persistent register/log where available), and resets. It MUST NOT attempt complex work — it may itself be running from a corrupted state.
- **Watchdog.** A hardware watchdog MUST be enabled in Released production firmware, and SHOULD be brought up early in development — retrofitting one finds every blocking bug you wrote in the meantime. It is fed from a point that proves the system is actually alive (e.g. a health monitor that checks all critical tasks) — never from a timer ISR that keeps ticking while everything else is dead. Reset cause MUST be read at boot, logged and reported.

---

## 7. State Machines

Stateful behaviour that is not an explicit state machine is an implicit one — with undocumented states and untested transitions.

- **Explicit states.** Non-trivial stateful behaviour MUST be modelled as an explicit state machine: a named state enum, a stored current state, and transitions in one place (a dispatch function or table) — not as a web of boolean flags whose combinations nobody enumerated.
- **Events drive transitions.** State changes happen in response to defined events, through the transition logic — never by scattered writes to the state variable from random call sites.
- **Complete handling.** Every state MUST define its response to every relevant event, including timeout and error events. Unhandled event–state pairs are explicit no-ops or explicit errors, not accidents. Every state that waits SHOULD have a timeout.
- **No hidden states.** If behaviour depends on a combination of flags, that combination *is* a state — name it. Transition side effects (entry/exit actions) live with the transition, not sprinkled around the codebase.
- **Documentation.** Module documentation MUST include the state machine: states, events, transitions — as a diagram or table. Transitions SHOULD be observable in logs at debug level.

---

## 8. Interrupt Design

Interrupt context is the most expensive place to run code and the easiest place to create unreproducible bugs. Keep it minimal.

- **ISRs do the minimum:** capture the hardware event, clear the source, hand the data off (queue, buffer, notification), request deferred processing. Real work — parsing, decisions, I/O — happens in task context. ISR execution time SHOULD be measured, not guessed.
- **No blocking, no allocation.** ISRs MUST NOT block, wait, allocate memory or call any API not explicitly ISR-safe. RTOS calls from ISRs MUST use their ISR-safe variants.
- **Deferred processing** is the default pattern: ISR signals, task processes. High-rate data moves via DMA and ring buffers rather than per-byte interrupts.
- **Shared data** between ISR and task context MUST be protected: atomics or lock-free single-producer/single-consumer structures where possible, briefest possible interrupt masking otherwise. Every variable shared with an ISR is documented as such.
- **Priorities.** Interrupt priorities MUST be assigned deliberately and documented in one place, with the reasoning. Priorities interacting with RTOS API-call thresholds MUST respect the kernel's constraints. Nested interrupt scenarios SHOULD be avoided unless a latency requirement demands them.

---

## 9. RTOS Guidelines

This section applies when an RTOS is used; firmware without an RTOS needs no RTOS-oriented architecture and no RTOS documentation. The discipline is still worth reading: a bare-metal main loop with ISRs has the same concurrency, priority and starvation questions — it just answers them implicitly. Answer them explicitly, whichever way you build.

- **Task design.** One task per ongoing responsibility (an active object per Section 2), not per function call. Every task has a stated purpose, owned resources, message interface, priority and stack size — recorded in the architecture document. Task count stays small; tasks are structural, not a convenience.
- **Priorities** are assigned by deadline, not importance, and documented with reasoning. Equal-priority time-slicing between cooperating tasks is acceptable; starvation is not — every task must get its turn under worst-case load.
- **Queues are the default** inter-task mechanism: they transfer both data and ownership and impose bounded buffering. Design queue depths for worst-case burst and define the overflow behaviour (drop-oldest, drop-newest, backpressure) explicitly.
- **Mutexes** protect shared resources that cannot be handed off — held briefly, never across blocking calls, always with priority inheritance where the kernel supports it. Any lock acquisition SHOULD use a timeout with a defined failure path rather than waiting forever.
- **Semaphores** signal events; mutexes protect data. Do not swap their roles.
- **Timers.** Callbacks in timer/service context follow ISR-like discipline: short, non-blocking, hand off real work to a task.
- **Deadlock avoidance** by construction: prefer message passing over shared state; when multiple locks are unavoidable, define and document a global lock ordering and always acquire in that order.
- **Priority inversion** is avoided by priority-inheritance mutexes and short critical sections — never by hand-tuning priorities until the symptom moves.
- **Stack sizing.** Every task's stack MUST be sized deliberately and verified by high-water-mark measurement under worst-case operation, with margin. Stack overflow checking MUST be enabled where the kernel provides it.

---

## 10. Memory Management

Allocate statically where determinism and longevity matter; let the linker prove it fits.

- **Static allocation is the default.** Tasks, queues, buffers and module state SHOULD be statically allocated so total memory use is known at link time.
- **Dynamic allocation is not banned — it is placed.** It is reasonable at startup for configuration-dependent sizing, and in non-critical paths of resource-rich targets (embedded Linux, large-RAM SoCs) where the platform is built for it. It is unsafe in steady-state operation of long-running MCU firmware — repeated allocate/free cycles fragment the heap and fail unpredictably months in — and has no place in ISRs, control loops or anything with a deadline. Where a pool of variable objects is genuinely needed in such paths, use fixed-size block pools with defined exhaustion behaviour. State in the design notes which pattern the project uses.
- **No unbounded buffers.** Every buffer, queue and table has a fixed capacity chosen for worst case, and code handles the full condition explicitly.
- **Ownership.** Every dynamically created object and every buffer passed between contexts has exactly one owner at any time, and module documentation states who allocates, who frees, and when. Passing a pointer through a queue transfers ownership — the sender MUST NOT touch it afterwards.
- **Lifetime.** References MUST NOT outlive what they point to: no returning pointers to stack data, no storing caller-owned pointers beyond the call unless the API contract explicitly says so.
- **Monitoring.** Firmware SHOULD track and expose memory health in operation: task stack high-water marks, heap remaining (if a heap exists), queue peak depths. "It fits today" is a measurement, not a guarantee.

---

## 11. Concurrency

Concurrency bugs are the most expensive bugs in firmware: rare, unreproducible, and found in the field. Prevent them by design, not by debugging.

- **Single ownership.** Every piece of mutable data is owned by exactly one task or ISR context. Others interact through messages or the owner's thread-safe API. Data owned by everyone is protected by no one.
- **Message passing over shared state.** Where data must cross contexts, prefer handing it off (queues, ownership transfer) to sharing it under a lock.
- **Atomics** for simple shared flags and counters — with the correct memory ordering, documented if anything beyond the conservative default is used. Read-modify-write on a "simple" shared variable without atomics is a race, full stop.
- **Critical sections** are a last resort: as short as possible, no function calls with unknown cost inside, never nested with locks in inconsistent order.
- **Race conditions** hide in check-then-act sequences (`if (ready) use();`) and in lazy initialization. Any such sequence on shared state MUST be made atomic or restructured.
- **Lock ordering.** If more than one lock exists in the system, the acquisition order is documented globally and enforced in review (see Section 9).
- **Document the contract.** Every public API states its concurrency contract: thread-safe, ISR-safe, owner-context-only. Absence of a statement means owner-context-only.

---

## 12. Logging and Diagnostics

You will not have a debugger attached when the interesting bug happens. Logging and diagnostics are how firmware testifies about its own behaviour.

- **Log levels** are used consistently: `ERROR` (functionality lost), `WARNING` (degraded, recovered), `INFO` (state changes, lifecycle events — low rate), `DEBUG` (development detail, compiled out or disabled in production). Levels MUST be configurable per module, at least at compile time, so one noisy module cannot drown the system.
- **Structured content.** Every message states which module speaks (a module tag), what happened and the relevant values with units. Log messages are written for the maintainer reading them years later, not the author who already knows the context.
- **Logging must not distort the system.** The logging path in operational code MUST be non-blocking (buffered/deferred output); a full log path drops messages (and counts the drops) rather than stalling tasks. Logging MUST NOT be called from ISRs unless the mechanism is explicitly ISR-safe.
- **Tracing.** Timing-critical work SHOULD be observable with cheap mechanisms (GPIO toggles, cycle counters, trace hooks) that can be enabled without restructuring code.
- **Runtime diagnostics.** Firmware SHOULD maintain health state per subsystem and expose it (Section 6's health monitoring, status LEDs, host-readable status). Diagnostic counters (errors, retries, overflows, drops) are cheap; maintain them in every module where things can go wrong.
- **Production diagnostics.** Released firmware MUST retain a usable diagnostic surface: reset cause reporting, version reporting, error counters, health status — accessible over a host interface without a debug probe.

---

## 13. Configuration Management

Configuration is part of the firmware's contract. Every configurable value is a decision — make it visible, validated and versioned.

- **Compile-time configuration** lives in dedicated config headers (per project and per module), not scattered `#define`s. Every option has a documented default and its valid range enforced with compile-time checks where possible. A build configuration MUST be reproducible: same sources + same config = same binary.
- **Feature flags** SHOULD be additive and few. Every flag doubles the configuration space; regularly used combinations SHOULD be built and tested regularly — in CI where it exists (Section 18). Dead flags are removed, not accumulated.
- **Runtime configuration** (calibration, user settings) MUST be stored with a version field and integrity check (CRC), validated on load, with defined behaviour for missing, corrupt or newer-version data: fall back to safe defaults and report — never crash, never silently half-apply.
- **Version reporting.** Released firmware MUST report its exact identity at boot (log) and on request (host interface): version per the AES versioning standard, plus build traceability (VCS revision). Do it from the first prototype anyway — a binary of unknown origin is undebuggable.
- **Hardware compatibility.** Firmware that supports multiple hardware revisions MUST detect or be told the revision explicitly and refuse to run on unsupported hardware, rather than misbehaving on it. Board differences are absorbed in the HAL/config layer (Section 3).

---

## 14. Communication Interfaces

Interfaces to the outside world share one discipline regardless of transport — UART, SPI, I²C, USB, CAN, BLE or Ethernet. The transport varies; the API shape and the paranoia do not.

- **Uniform driver API.** Every interface driver follows the standard module shape (Section 5): configuration struct, `init`/`deinit`, status-returning transfer operations. Application code written against one bus driver should feel familiar on every other.
- **Timeouts everywhere.** Every blocking bus operation MUST have a timeout with a defined failure return. No transaction may hang a task indefinitely because a wire broke or a peripheral wedged — bus fault recovery (e.g. I²C bus clear) SHOULD be part of the driver.
- **Validate all input.** Everything received is untrusted until it passes framing, length, address and integrity checks (Section 6). Malformed input is counted and discarded — it MUST NOT be able to crash, stall or desynchronize the firmware. Parsers MUST tolerate arbitrary byte streams (truncation, garbage, restart mid-frame) and resynchronize.
- **Framing and integrity.** Message-based protocols over raw transports MUST define explicit framing with a CRC. Protocols MUST be versioned, and implementations MUST reject or safely ignore versions and message types they do not understand.
- **Shared buses.** Bus ownership and arbitration between modules MUST be explicit (one owning driver or a documented locking scheme) — two modules independently driving one bus is a design defect.
- **Flow and errors.** Define what happens under load and loss: bounded buffers, backpressure or documented drop policy, retry limits, and error counters visible in diagnostics (Section 12).

### 14.1 Managed Unit API

These rules apply to firmware in a **Managed Unit** — an AURIORA Unit that contains a programmable controller and exposes a versioned, high-level Unit API to its host Module ([AES Architecture](https://github.com/auriora-org/auriora-engineering-standard/blob/main/docs/03-architecture.md), [AES-UNIT-006](https://github.com/auriora-org/auriora-engineering-standard/blob/main/docs/03-architecture.md#aes-unit-006-declared-execution-model)). They add to, and do not replace, the communication discipline above.

- **High-level contract.** The public Unit API MUST expose capabilities, operations, status and errors at a high level. Internal component types and register-level protocols (radio, GNSS, sensor registers) MUST NOT be part of the host contract — the host operates the Unit through its API, not its internals. SPI (or whichever bus the profile defines) is a transport, not the semantic API definition.
- **Versioned protocol.** The Unit API MUST be versioned. Messages MUST have defined framing, length, command/message identifiers, status/error codes, timeout behavior and integrity checking (Section 14 framing rules apply). Unknown commands and unsupported API versions MUST fail safely (defined error, no side effects), never by undefined behavior.
- **Readable identity.** The Managed Unit MUST expose a readable firmware identity and Unit API version (Section 13 version reporting), so the host can validate API compatibility before relying on the Unit.
- **Lifecycle and `UIF_READY`.** The Unit MUST implement at least the lifecycle states *disabled → starting → ready → fault*, as an explicit state machine (Section 7). `UIF_READY` MUST remain LOW until the Unit can accept valid API transactions, and MUST be deasserted before shutdown or on entering an unrecoverable fault state. Bring outputs to a safe state on fault (Section 6 fatal-error handling).
- **Hardware sync over software timing.** Where the Unit Interface Profile provides a dedicated hardware synchronization signal, the host contract MUST NOT depend on timing derived only from software message latency; drive and document the hardware signal instead. This is a Unit-to-host signal within one Module; the Module-to-Module SYNC interface is a different thing (§14.2).

### 14.2 Module Synchronization Interface (SYNC)

These rules apply to firmware in a Module that provides a SYNC IN or SYNC OUT port. What SYNC is and the behavior AES requires — a single-meaning event edge, armed execution, ignore-while-running by default, an explicit post-completion mode, a configured output source and a logged event record — are defined by AES ([Interfaces and Versioning §4](https://github.com/auriora-org/auriora-engineering-standard/blob/main/docs/05-interfaces-and-versioning.md#4-module-synchronization-interface), `AES-SYNC-001` to `AES-SYNC-004`); this section states how firmware implements them. SYNC is Module-to-Module and is unrelated to the Unit-to-host synchronization signal of §14.1.

- **Capture the edge in hardware.** Timestamp the SYNC IN rising edge with a hardware mechanism — timer input capture, an edge interrupt reading a free-running counter, DMA or a programmable-I/O peripheral — never by polling from a task. The timestamp is taken at capture, not when the event is processed. Where the Module has a sample clock (audio frames, ADC samples), convert the capture to that time base and record the event as a frame or sample index as well as in local time; the sample index is what later correlation uses.
- **ISR does the minimum.** The capture ISR records the timestamp, increments the local RX counter and hands off (Section 8). The decision — armed or not, valid in this state or not, which action — runs in the owning task through the state machine.
- **Armed gate as an explicit state machine.** Model SYNC behavior with named states (baseline `IDLE → CONFIGURED → ARMED → RUNNING → COMPLETE`, Section 7) and put the SYNC event into the transition table of every state. An event in a state that does not accept it is an explicit no-op that logs an ignored event with a reason (`NOT_ARMED`, `RUNNING`, `INVALID_STATE`) — never a silent drop and never an implicit restart, stop or queue. `COMPLETE → IDLE` (one-shot) versus `COMPLETE → ARMED` (repeat-armed) is a configuration value, readable over the Host Interface.
- **Binding is validated configuration.** The SYNC IN action, SYNC OUT source, delay and post-completion mode are runtime configuration (Section 13): stored with version and integrity check, validated against the actions and sources this Module actually supports, rejected with a defined error otherwise, and reported in the Module's status so a host can record them with the experiment. Support only the actions the Module needs — `NONE` plus one start action is a complete first implementation.
- **Deterministic delay.** A configured delay between the event and the action is realized on a hardware timer, or scheduled against the sample clock from the captured timestamp, so that its jitter is the capture jitter and not task latency. The delay counts from the captured edge, not from when the ISR ran or the task woke.
- **SYNC OUT marks the real event.** Generate the output pulse from the internal event it is bound to — the block in which the first stimulus sample actually leaves the converter, the first acquired sample, the protocol end — through a hardware-timed output, compensating known fixed pipeline latency where it has been characterized. Do not pulse on command receipt or on SYNC IN reception unless that is the configured source. One source per output by default; where several sources may be active at once, the configuration says so explicitly and the documentation says that a receiver cannot tell the pulses apart.
- **Never encode meaning in the pulse.** Pulse width, count and spacing carry no information: firmware MUST NOT emit patterns and MUST NOT interpret them. Anything that needs a type or a parameter goes over the Host Interface.
- **Log and count.** Record received, transmitted and ignored SYNC events (Section 12) with local timestamp and sample index, current state, related run or protocol identifier, ignore reason and the local per-direction counter. The counters are diagnostics readable over the Host Interface, reset only explicitly, and local: never present them as synchronized with another Module.
- **Test it.** Host-test the state machine against every state × event pair, including unarmed, mid-run and repeat-armed cases (Section 18); on hardware, measure edge-to-action latency and jitter over a run of events and record the figures in the project documentation — they are part of the Module's timing contract.

---

## 15. Power Management

Power management is an architecture concern, not an optimization pass at the end.

- **Central policy.** Sleep decisions belong to one power-management module that knows the system state — not to individual modules independently deciding to sleep. Modules express readiness (idle / busy / wake-latency constraints); the policy decides.
- **Low-power modes.** The power states used, their entry/exit conditions and their wake latencies MUST be documented. State retention across each sleep mode (RAM, peripherals, pin states) is verified, not assumed.
- **Wake-up sources** are configured deliberately and documented. Every wake path is handled — including the unexpected-wake case — and pins are put in defined states before sleep so nothing floats or back-powers.
- **Peripheral shutdown.** Peripherals, clocks and power rails that are not in use SHOULD be disabled, and drivers SHOULD support suspend/resume so the power policy can drive them.
- **Energy efficiency by design:** event-driven over polling, batched work over frequent wakeups, DMA over CPU busy-loops. Timer periods are chosen for the requirement, not for round numbers.
- **Measure.** Power consumption in each operating state SHOULD be measured on real hardware and recorded in project documentation. Sleep-mode regressions are invisible in code review; only measurement catches them.

---

## 16. Firmware Update

Update is a first-class feature of the firmware, not an afterthought — it is the mechanism by which every future bug you don't know about yet gets fixed.

- **Bootloader.** Field-updatable devices MUST have a bootloader that is small, independent from application code and treated as immutable once released — a bootloader bug can brick every deployed unit. The bootloader's job is verification and boot decision, nothing more.
- **Integrity.** Every image MUST carry integrity metadata (length, CRC or digest, version, target hardware compatibility). The bootloader MUST verify integrity and compatibility before every boot, not only after update.
- **Power-loss safety.** An interrupted update MUST leave the device bootable. Dual-slot (A/B) update SHOULD be the default scheme; the new image is written and fully verified before the boot choice is switched, and switching is atomic.
- **Rollback.** If the new image fails to boot or fails its self-check, the bootloader MUST fall back to the previous working image automatically. The application confirms an update as good ("commit") only after reaching verified operation; watchdog resets before commit count as failure.
- **Version compatibility.** Updates MUST respect versioned compatibility: configuration data (Section 13) is migrated or safely defaulted across versions; downgrade behaviour is defined and enforced, not accidental.
- **Verification of delivery.** The update transport verifies each transferred block (Section 14 rules apply) and the complete image before activation. Reported firmware version (Section 13) is the ground truth for what actually runs.

---

## 17. Security

Security measures are proportional to the threat model — but the threat model MUST be written down, and "none" is a conclusion, not a default.

- **Threat model first.** Each product documents what it protects against: physical access, network attackers, malicious updates, counterfeit hardware. Controls follow from that, not from fashion.
- **Secure boot.** Where the platform supports it and the threat model requires it, the boot chain SHOULD verify signatures from ROM upward. At minimum, update images for field-updatable devices SHOULD be cryptographically signed and verified before activation — CRC catches accidents, signatures catch attackers.
- **Cryptography.** Use established, published algorithms and audited implementations. Inventing or hand-implementing cryptographic primitives is prohibited. Protocol-level security relies on standard constructions, not homemade obfuscation.
- **Key management.** Private keys MUST never appear in repositories, build artifacts or logs. Signing happens in controlled infrastructure; device keys are provisioned per device where the design requires them, stored in hardware-protected storage where available, and revocable/rotatable by design.
- **Random numbers.** Anything security-relevant MUST use a hardware entropy source or a CSPRNG properly seeded from one, with defined behaviour when entropy is unavailable (fail or degrade explicitly — never silently fall back to a weak source).
- **Secure communication.** Where the threat model includes the transport, communication MUST be authenticated (and encrypted if confidentiality matters) using standard mechanisms. Debug and diagnostic interfaces are part of the attack surface: production builds disable or protect them deliberately.

---

## 18. Testing

Untested firmware is untested regardless of how carefully it was written. The architecture of Section 2 exists partly to make testing cheap — use that. Scale expectations to maturity and risk: Experimental firmware needs no test suite; Released firmware earns its trust with one.

- **Host-based unit tests.** Hardware-independent code (services, policies, protocol logic, state machines) SHOULD be unit-tested on the host PC with mocked lower layers, and for Released firmware the critical logic — parsers of external input, safety-relevant state machines, calibration math — MUST be. This is where most logic bugs die: milliseconds per run, no hardware. Early prototypes need not test every hardware-independent function; test what would hurt.
- **Mocks at layer boundaries.** Tests mock the layer below the unit under test (HAL, drivers, RTOS primitives), following the project structure of Section 3. If a module is hard to mock, that is architecture feedback.
- **What to test:** normal paths, error paths, boundaries and malformed input. Every state machine's transitions, every parser against garbage (fuzz-style tests SHOULD be applied to anything parsing external input), every policy against its edge cases. A bug fixed without a regression test will return.
- **Integration tests** verify modules against each other and against real peripherals on target hardware: bus communication, storage, timing-dependent behaviour that host tests cannot represent.
- **Hardware testing.** Release candidates MUST be tested on real hardware under realistic conditions — including power cycling, communication faults, watchdog recovery and update/rollback paths (Section 16). Hardware-in-the-loop automation SHOULD be used where volume justifies it.
- **Continuous integration.** Repositories in Active Development or beyond SHOULD run CI that builds the supported configurations and runs host tests; Experimental projects need none. Without CI, run the build and tests locally before merging — the obligation is that changes are built and tested, not that a server does it. Build warnings are errors — warnings that scroll by unread are bugs with a timestamp.
- **Simulation** (peripheral simulators, board emulators) MAY be used to extend automated coverage where host tests can't reach and hardware doesn't scale.

---

## 19. Documentation

Documentation is written for the developer who arrives after everyone who understood the system has left. Documentation minimums are set by the [AES maturity model](https://github.com/auriora-org/auriora-engineering-standard/blob/main/docs/07-maturity-and-release.md) and the [AURIORA Documentation Standard](https://github.com/auriora-org/auriora-documentation-standard); the firmware-specific shape:

- **README.** Every firmware repository MUST have a README covering: what the firmware does, its maturity level, target hardware, how to build and flash (and run tests, where tests exist). A newcomer with the README and a clean checkout must reach a running build without asking anyone. For an Experimental repository this may be the entire documentation.
- **Architecture documentation.** Per Section 2.3: SHOULD for non-trivial Active Development firmware, MUST for Released firmware — layers, modules and responsibilities, tasks with priorities and communication paths, startup sequence, key decisions with reasoning. A section in `docs/design-notes.md` is a fine home. Update it in the same change that changes the architecture.
- **Module documentation.** Modules with non-obvious contracts state their purpose, API contract, concurrency contract (Section 11) and state machine (Section 7) where applicable. Critical subsystems SHOULD document their invariants explicitly — the properties that must always hold.
- **API documentation.** Public API that others consume SHOULD carry structured documentation comments (Doxygen-style): what it does, parameters with units and valid ranges, return values including errors, side effects and contracts. For Released firmware consumed as a library or protocol endpoint this is a MUST — the header alone must suffice to use the module correctly.
- **Inline comments** follow Section 4: explain *why* — the workaround's reason, the datasheet erratum, the non-obvious ordering constraint — with references where they exist.
- **Changelog.** Repositories with tagged releases MUST keep a changelog per released version, recording behaviour changes, fixed defects and compatibility impacts, aligned with the AES versioning standard. Unreleased repositories need none.

---

## 20. Code Review Checklist

Compact checklist for firmware changes. Self-review is valid ([AES-QA-001](https://github.com/auriora-org/auriora-engineering-standard/blob/main/docs/07-maturity-and-release.md#aes-qa-001-proportionate-review)) — preferably after a short time away from the code. Skip items that plainly don't apply; no N/A bookkeeping. The core prototype-grade check is the first block; the rest bears down as maturity rises.

**Every change**
- [ ] Clean build from documented steps, warning-free
- [ ] Critical error paths considered: fallible calls checked, external input validated, no waits without timeout
- [ ] Outputs stay safe at boot and on fault paths touched by this change
- [ ] Interfaces still match their documentation (commands, framing, units)
- [ ] Obvious secrets, build outputs and debug leftovers excluded
- [ ] Known limitations and non-obvious decisions recorded (design notes or comments)
- [ ] Relevant basic tests pass

**Concurrency and resources (where touched)**
- [ ] Every shared datum has one owner or explicit, correct protection; no check-then-act races
- [ ] ISR code minimal, non-blocking, ISR-safe APIs only; locks held briefly, ordering respected
- [ ] Buffer/queue sizes justified for worst case; no steady-state allocation in deterministic paths
- [ ] Stack and timing impact of new code considered

**Toward Release (additionally)**
- [ ] Critical logic host-tested, including error paths and boundaries; fixed bugs have regression tests
- [ ] Policy logic separated from hardware access; layer boundaries respected
- [ ] Architecture/module/API documentation and changelog updated
- [ ] All supported configurations build and pass tests (in CI where it exists)

---

## 21. Best Practices

The principles above, condensed to what experienced firmware engineers check by reflex:

- **Avoid global state.** Every global mutable variable is an undocumented API with unlimited callers. Own state inside modules; inject dependencies at init.
- **Avoid hidden dependencies.** If a module needs something, it says so in its header or init signature — not by silently reading a global, a magic section of flash, or the state another module happened to leave behind.
- **Minimize coupling, maximize cohesion.** Things that change together live together; things that change separately talk through small, stable interfaces. The test is refactoring cost: touching one module should not ripple.
- **Deterministic timing.** Bounded loops, bounded queues, timeouts on every wait, static allocation. If you cannot state the worst case, you have not designed the fast path — you've sampled it.
- **Defensive at the boundaries, confident inside.** Validate everything that crosses a trust boundary; then trust your own invariants and assert them.
- **Portability is layering.** Firmware is portable exactly to the degree that platform knowledge is confined to the HAL and drivers. Every SDK call in application code is a tax on the next platform.
- **Make it observable.** Version, health, counters, logs, reset cause. Firmware that cannot explain what it did will be debugged by superstition.
- **Boring is a feature.** The best firmware is unsurprising: uniform structure, uniform APIs, explicit state, no cleverness. Surprise is where the bugs live.

---

*AURIORA Firmware Style Guide 0.3.0 — complements the AURIORA Engineering Standard. Licensed under CC BY-SA 4.0.*
