# AURIORA Firmware Style Guide

The AURIORA Firmware Style Guide (AFSG) is the official firmware development guide for all AURIORA embedded systems. It complements the AURIORA Engineering Standard (AES): AES defines *what* must be true about AURIORA engineering work, AFSG defines *how* firmware should be designed, structured, implemented, documented and maintained across all AURIORA hardware platforms.

Requirements scale with the AES maturity level: safety rules apply always (safe outputs at boot and fault, validated external input), release-grade obligations (reproducible builds, update integrity, watchdog, production diagnostics) bind Released firmware, and the architecture, structure, testing and CI guidance is the recommended default that firmware converges on as it matures — an experimental spike may honestly be a single `main.c` with a README.

Read the guide: [GUIDE.md](./GUIDE.md)

## Scope

- Design philosophy and a common firmware architecture: layers, modules, active objects, policy/mechanism separation
- Project structure, coding style, API design, error handling, state machines
- Interrupt design, RTOS guidelines (when an RTOS is used), memory management, concurrency
- Logging and diagnostics, configuration management, communication interfaces, power management
- Firmware update, security, testing scaled to maturity, documentation
- A compact code review checklist with maturity-scaled depth

The guide is platform-independent: it avoids references to specific programming languages, compilers, SDKs and vendors, and applies to RP-series, ESP32, STM32, Nordic, Microchip and future embedded platforms alike.

## Requirement Language

Aligned with AES:

- `MUST` — mandatory when applicable (equivalent to `SHALL` in AES)
- `SHOULD` — recommended default; engineering judgment may justify another approach, no documented exception needed
- `MAY` — optional improvement

Where AFSG and AES conflict, AES prevails.

## Versioning and Changes

Released versions are recorded in the [CHANGELOG](./CHANGELOG.md) and tagged in version control.

## License

This documentation is licensed under the [Creative Commons Attribution-ShareAlike 4.0 International License (CC BY-SA 4.0)](./LICENSE).
