# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **documentation-only repository** — Software Design Documents for the HS2 3U CubeSat flight software. There is no code to build, lint, or test. The deliverable is markdown documentation.

## Document Structure

```
docs/sdd.md                                      # Top-level satellite SDD (architecture, modes, topology)
docs/Adcs/AdcsApplication.md                     # ADCS application component SDD
docs/DataCollection/DataCollectionApplication.md
docs/ScienceInference/ScienceInferenceApplication.md
docs/Comms/CommsApplication.md
fprime-repo-summary.md                           # F' framework reference guide (read this first)
```

## Architecture Context

The satellite uses **F' (F Prime)** (`nasa/fprime@devel`), a NASA JPL component-driven embedded framework. The design follows a strict **three-layer App-Manager-Driver** architecture:

- **Layer 3 — Application (`*Application`):** `AdcsApplication`, `DataCollectionApplication`, `ScienceInferenceApplication`, `CommsApplication`; packaged inside custom subtopologies (ADCS, DataCollection, ScienceInference, Comms) plus pre-built CdhCore, ComCcsds, FileHandling, DataProducts. Each has a **hierarchical F' state machine** where mode is the top-level state and operational substates are nested inside. Receive mode commands from `SatStateMachine` via typed ports.
- **Layer 2 — Managers (`*Manager`):** Hardware device managers — `ImuManager`, `Camera1Manager`, `StarTrackerManager`, etc. Single flat SM: `RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN`. No satellite mode awareness. Reference: `fprime-community/fprime-sensors` ImuManager.
- **Layer 1 — Drivers (`*Driver`):** F' native bus drivers with no device knowledge (LinuxI2cDriver, LinuxSpiDriver, LinuxUartDriver, LinuxGpioDriver)

Three Layer 2 managers are shared across subtopologies and instantiated at the top-level topology: `StarTrackerManager`, `GnssManager`, `EnduroSatManager`.

`SatStateMachine` manages two main modes: **Safe** and **Standby**. Standby has four autonomous submodes (Downlink, Science, Charge, Eclipse) evaluated each 1 Hz tick in priority order. `SatStateMachine` sends each application component its target mode via a dedicated typed port — it owns the translation table.

## F' Framework Conventions (critical for writing accurate SDDs)

- **Application components** use hierarchical F' state machines — mode is top-level state, operational substates nested inside; top-level `switchMode` signal inherited by all leaf states (see `nasa/fpp` `Defining-State-Machines.adoc#inherited-transitions`)
- **Hardware managers** use a single flat SM: `RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN`; self-healing on error (→ RESET); driven by rate group tick
- **Callback port pattern** used for parallel async requests (e.g., DataCollectionApplication simultaneous camera + attitude + position capture)
- Health monitoring (`Svc::Health`) applies to application components only — hardware managers excluded
- Parameters persisted via `Svc::PrmDb`; science results via `DataProducts` subtopology

See `fprime-repo-summary.md` for F' component types, port kinds, FPP syntax, pre-built service components, and GitHub raw URLs to fetch F' source docs.

## SDD Format Conventions

Follow the structure established in `docs/DataCollection/DataCollectionApplication.md`:
1. **Overview** — component role, subtopology position, layer
2. **Requirements** — table with ID (`HS2-<ABV>-NNN`), requirement, verification method
3. **Design** — component type, mode interface + enum, ports table, commands table
4. **State Machine** — hierarchical SM diagram with mode top-level states and nested substates
5. **Notes** — cross-subtopology wiring, constraints, deferred items

Port types use F' framework types (`Svc.Sched`, `Fw.Cmd`, `Fw.Dp`, `Fw.Log`, `Fw.Tlm`, `Svc.Ping`, `Fw.PrmGet`).

## Architecture Status (as of 2026-04-21)

### Established Decisions

- Three-layer architecture is locked: Layer 3 `*Application`, Layer 2 `*Manager`, Layer 1 `*Driver`.
- `SatStateMachine` owns top-level mode logic; sends typed mode commands to each Layer 3 app via dedicated ports.
- Layer 3 hierarchical SMs are established for `AdcsApplication`, `DataCollectionApplication`, `ScienceInferenceApplication`, `CommsApplication`.
- Layer 2 flat SM pattern (`RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN`) is established for all hardware managers.
- Top-level modes: `Safe` and `Standby`; Standby submodes: `Downlink`, `Science`, `Charge`, `Eclipse`.

### Open Questions

- **EPS:** `EpsManager` as standalone Layer 2, or introduce a Layer 3 `EpsApplication` for mode-aware power/fault logic?
- **CDH decomposition:** Map command handling, storage, parameters, health, and downlink from the PDF into F' pre-built components and any required custom pieces.
- **Ground software docs:** Translate ground-software chapter from the PDF into markdown, or mark out of scope.
- **Mode-switch behavior:** Define mid-operation mode-change handling for `DataCollectionApplication` and `AdcsApplication`.

### Immediate Next Step

Decide EPS architecture first — smallest unresolved question that determines whether additional Layer 3 app docs are needed.
