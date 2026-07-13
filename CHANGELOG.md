# Changelog

All notable changes to the AURIORA Firmware Style Guide are documented in this file. Released versions are tagged in version control.

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
