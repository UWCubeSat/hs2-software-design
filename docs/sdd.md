# HS2 Satellite Flight Software — Software Design Document
**Date:** 2026-04-20
**Framework:** F' (F Prime) — `nasa/fprime@devel`
**Platform:** 3U CubeSat, single flight computer

---

## 1. Mission Overview

HS2 is a 3U CubeSat scientific mission validating three optical navigation algorithms:
- **LOST** — lost-in-space star identification; runs on Camera 1
- **FOUND** — follow-up optical navigation; runs on Camera 2
- **SCOPE** — star catalog optical processing; runs LOST internally as a preprocessing stage; used for calibration experiments on Camera 1 + 2

All algorithms are included as external C++ libraries via CMake. Science results are always stored as F' data products. Raw images are stored to external flash when flagged by the science algorithms. The flight software is implemented in F' and organized into custom and pre-built subtopologies.

---

## 2. Hardware Inventory

| Hardware | Interface | Primary User |
|----------|-----------|--------------|
| Camera 1 | SPI/I2C | DataCollectionApplication (LOST, SCOPE) |
| Camera 2 | SPI/I2C | DataCollectionApplication (FOUND, SCOPE) |
| Star Tracker | UART | DataCollectionApplication (synchronized capture), AdcsApplication (precision pointing) |
| aGNSS Receiver | UART | DataCollectionApplication (synchronized capture), AdcsApplication (position/timing), SatStateMachine (orbital state) |
| IMU | SPI/I2C | AdcsApplication |
| Sun Sensors | I2C/GPIO | AdcsApplication, SatStateMachine (sun/eclipse detection) |
| Magnetorquers | PWM | AdcsApplication |
| EPS Board | I2C/UART | EPSApplication, MpptIcManager, SatStateMachine |
| EnduroSat S-band Radio | UART | CommsApplication |
| External Flash | SPI | FileHandling subtopology |
| Temperature Sensors | I2C | ThermalApplication (via TemperatureSensorManager) |
| Heater | PWM | ThermalApplication (via HeaterManager) |

---

## 3. Operational Modes

The satellite operates in two main modes managed by `SatStateMachine`. Uplink commands are accepted in all modes via the always-active omni communications link.

| Mode | Description |
|------|-------------|
| **Safe** | Initial and emergency mode. Minimal operations. Detumble runs when power allows. Omni comms always active. |
| **Standby** | Full autonomous operations. Entered after ground completes checkout. Submodes evaluated each 1 Hz tick. |

### Safe Mode

The satellite enters Safe mode on: deploy/reboot, critical battery condition reported by `EPSApplication` via `powerState` (vbatt below `CRITICAL_THRESHOLD`), or ground command `SAFE_MODE`. Exits to Standby on ground command `SAFE_EXIT` (only after checkout has been completed).

`AdcsApplication` runs detumble (magnetometer B-dot algorithm) when power allows. All other application components are inactive.

### Standby Mode Entry (Checkout)

Checkout is a one-time ground-commanded commissioning sequence performed before the satellite enters autonomous Standby operations. Ground commands subsystem verification and OKs the transition. After checkout completes the satellite enters Standby and does not return to a checkout mode.

### Standby Submodes

Evaluated each 1 Hz tick by `SatStateMachine` in priority order. The highest-priority condition that is met determines the active submode.

| Priority | Submode | Entry Condition |
|----------|---------|-----------------|
| 1 | **Downlink** | Over ground station AND downlink queue above `DOWNLINK_QUEUE_THRESHOLD` AND power OK |
| 2 | **Science** | Power OK AND `EXPERIMENT_ENABLED` parameter set AND not Downlink |
| 3 | **Charge** | In sun AND not Downlink AND not Science |
| 4 | **Eclipse** | Fallback — in eclipse, not Downlink, not Science |

**Condition sources:**
- Power OK → `EPSApplication` (state of charge above `POWER_THRESHOLD` parameter)
- Over ground station → `GnssManager` (orbital position + ephemeris)
- In sun / in eclipse → sun sensors AND `GnssManager` orbital position calculation
- Downlink queue depth → `ComQueue` component
- `EXPERIMENT_ENABLED`, `POWER_THRESHOLD`, `DOWNLINK_QUEUE_THRESHOLD` → persisted via `PrmDb`

**Key parameters:**

| Parameter | Description |
|-----------|-------------|
| `POWER_THRESHOLD` | Minimum EPS state of charge (%) for Downlink or Science |
| `EXPERIMENT_ENABLED` | Ground-set boolean enabling the Science submode |
| `DOWNLINK_QUEUE_THRESHOLD` | Minimum queue depth (bytes) required to enter Downlink |

---

## 4. Architecture Overview

The flight software uses a **five-layer architecture**. Component names reflect their layer:

```
Layer 5 — System Infrastructure
    CdhCore (CmdDispatcher, EventManager, Health, Version, AssertFatalAdapter, fatalHandler)
    HardwareResetManager [future]

Layer 4 — Mission Orchestration
    SatStateMachine

Layer 3 — Application components (*Application)
    DataCollectionApplication | ScienceInferenceApplication
    AdcsApplication | CommsApplication | EPSApplication | ThermalApplication
    + pre-built subtopologies: ComCcsds | FileHandling | DataProducts

Layer 2 — Hardware Managers (*Manager)
    Camera1Manager | Camera2Manager | StarTrackerManager | GnssManager
    ImuManager | SunSensorManager | MagnetorquerManager
    MpptIcManager | WatchdogPinger | DeployPanelsManager
    TemperatureSensorManager | HeaterManager
    EnduroSatManager

Layer 1 — F' Native Bus Drivers (*Driver)
    LinuxI2cDriver | LinuxSpiDriver | LinuxUartDriver | LinuxGpioDriver
```

**System infrastructure** (Layer 5) provides the satellite-wide backbone: command routing (`CmdDispatcher`), event logging and FATAL escalation (`EventManager → fatalHandler`), component liveness monitoring (`Health`), and version reporting. All other layers depend on Layer 5 services. `HardwareResetManager` is reserved for future Layer 5 work alongside a general-purpose `FaultManager`.

**Mission orchestration** (Layer 4) is `SatStateMachine` — it evaluates submode conditions each 1 Hz tick and sends typed mode commands to all Layer 3 application components. It has no hardware knowledge and never talks to Layer 2 or below directly.

**Application components** (Layer 3) contain mission logic and dispatch work to hardware managers, never talking directly to drivers. Most receive mode-switch commands from `SatStateMachine` and use a hierarchical F' state machine (`Fw::Sm`) where mode is the top-level state and operational substates are nested inside. `EPSApplication` is the exception — it has no mode port and no hierarchical SM, running continuously and responding only to rate group ticks and commands. See §9 for the standard pattern and §5.6 for the EPS exception.

**Hardware managers** (Layer 2) are Active or Queued components with a single flat F' state machine following the startup→operational→recovery pattern: `RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN`. They have no satellite mode awareness. Reference implementation: `fprime-community/fprime-sensors` `ImuManager`.

**Drivers** (Layer 1) are passive bus drivers with no device knowledge.

`StarTrackerManager`, `GnssManager`, and `EnduroSatManager` are instantiated at the **top-level topology** because they are shared across multiple subtopologies. All other hardware managers are instantiated inside their primary subtopology.

---

## 5. Subtopology Decomposition

### 5.1 Layer 5 — CdhCore Subtopology

`CdhCore` is the pre-built system infrastructure subtopology. It is the entry and exit point for all external communication and provides satellite-wide services consumed by every other layer.

| Component | Purpose |
|-----------|---------|
| `cmdDisp` (Svc.CmdDispatcher) | Routes uplink commands to all registered components |
| `events` (Svc.EventManager) | Collects and downlinks events; routes `FATAL`-severity events via `FatalAnnounce → fatalHandler` |
| `$health` (Svc.Health) | Ping-based liveness monitoring for all critical active components |
| `fatalHandler` (Svc.FatalHandler) | Resets the system on `FATAL` event; satellite reboots into Safe mode |
| `fatalAdapter` (Svc.AssertFatalAdapter) | Converts C++ assert failures to `FATAL` events |
| `version` (Svc.Version) | Reports software version |

### 5.2 Layer 3 Pre-Built Subtopologies

| Subtopology | Source | Purpose |
|-------------|--------|---------|
| `ComCcsds` | `Svc/Subtopologies/ComCcsds` | CCSDS communications stack; Space Packet and TM/TC frame framing, uplink/downlink pipeline |
| `FileHandling` | `Svc/Subtopologies/FileHandling` | FileUplink, FileDownlink, FileManager, PrmDb |
| `DataProducts` | `Svc/Subtopologies/DataProducts` | DpManager, DpWriter, DpCatalog for science results |

### 5.3 DataCollection Subtopology

**Purpose:** Executes data collection experiments — powers on cameras, acquires synchronized images and navigation data, stores results to flash, and reports outcome to `SatStateMachine`.

**Components:**

| Component | Type | Purpose |
|-----------|------|---------|
| `DataCollectionApplication` | Active (high priority) | Hierarchical SM; receives mode from `SatStateMachine`; orchestrates experiment execution |
| `Camera1Manager` | Active (worker) | State machine: RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN / error→RESET |
| `Camera2Manager` | Active (worker) | State machine: RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN / error→RESET |

**DataCollectionApplication modes (received via `Sat.DataColModePort`):**

| Mode | Behavior |
|------|----------|
| `Off` | Inactive. Cameras powered off. |
| `HealthCheck` | Powers on cameras, verifies operation, powers off. One-time checkout step. |
| `RunExperiment` | Full experiment: power on → capture → store → power off → report result |

**DataCollectionApplication hierarchical SM:**

```
OFF
HEALTH_CHECK
  └─ CHECKING
RUN_EXPERIMENT
  ├─ POWERING_ON
  ├─ CAPTURING      (simultaneous: Camera1, Camera2, StarTracker attitude, GNSS position)
  ├─ STORING
  └─ POWERING_OFF
```

Top-level `switchMode` signal inherited by all leaf states — mode switch valid from any substate.

**Synchronized capture requirement:** Within `CAPTURING`, `DataCollectionApplication` simultaneously requests images from Camera1Manager and Camera2Manager, attitude from `StarTrackerManager`, and position from `GnssManager`. All four must be acquired within a 10ms window before dispatching to `ScienceInferenceApplication`.

**Ports consumed from outside subtopology:**
- `StarTrackerManager` attitude port (top-level)
- `GnssManager` position/time port (top-level)
- `DpManager` / `DpWriter` (DataProducts subtopology)
- `FileDownlink` (FileHandling subtopology)
- `modeIn: Sat.DataColModePort` (from `SatStateMachine`)

**Health monitoring:** `DataCollectionApplication` is health-monitored. Camera managers excluded.

### 5.4 ScienceInference Subtopology

**Purpose:** Processes raw images stored on flash by running LOST, FOUND, or SCOPE. Operates on a scheduled polling cycle. Schedule-driven; receives no ground commands.

**Components:**

| Component | Type | Purpose |
|-----------|------|---------|
| `ScienceInferenceApplication` | Active | Hierarchical SM; receives mode from `SatStateMachine`; polls flash, invokes algorithms, stores results |

**ScienceInferenceApplication modes (received via `Sat.ScienceInferenceModePort`):**

| Mode | Behavior |
|------|----------|
| `Off` | Inactive. No flash polling. |
| `ProcessImages` | Polls flash for unprocessed images; invokes LOST, FOUND, or SCOPE per experiment metadata; stores results; compresses flagged images |

**External libraries invoked directly from `ScienceInferenceApplication` C++ implementation:**

| Library | Algorithm | Camera | Experiment Type |
|---------|-----------|--------|----------------|
| LOST | Lost-in-space star identification | Camera 1 | L&F |
| FOUND | Follow-up optical navigation | Camera 2 | L&F |
| SCOPE | Star catalog optical processing (runs LOST internally) | Camera 1 + 2 | Calibration |

**Health monitoring:** `ScienceInferenceApplication` is health-monitored.

### 5.5 ADCS Subtopology

**Purpose:** Attitude determination and control across all ADCS operating modes.

**Components:**

| Component | Type | Purpose |
|-----------|------|---------|
| `AdcsApplication` | Active (high priority) | Hierarchical SM; receives mode from `SatStateMachine`; runs attitude control loop |
| `ImuManager` | Queued (worker) | State machine: RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN / error→RESET |
| `SunSensorManager` | Queued (worker) | State machine: RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN / error→RESET |
| `MagnetorquerManager` | Queued (worker) | State machine: RESET → WAIT_RESET → CONFIGURE → RUN / error→RESET; drives six `LinuxPwmDriver` channels (two per axis) |

**AdcsApplication modes (received via `Sat.AdcsModePort`):**

| Mode | Sensors | Actuators | Use |
|------|---------|-----------|-----|
| `Off` | None | None | Inactive |
| `Detumble` | IMU, Magnetorquers | Magnetorquers | Safe mode — B-dot algorithm |
| `SunPointing` | IMU, Sun Sensors | Magnetorquers | Standby/Charge — point panels at sun |
| `AntennaPointing` | IMU, GNSS | Magnetorquers | Standby/Downlink — point antenna at ground station |
| `EarthLimbPointing` | IMU, Star Tracker | Magnetorquers | Standby/Science — point FOUND camera at lit Earth limb, optimize solar |
| `AttitudeHold` | IMU | Magnetorquers | Standby/Eclipse — hold current attitude |

**AdcsApplication hierarchical SM:**

```
OFF
DETUMBLE
  └─ RUNNING
SUN_POINTING
  ├─ ACQUIRING
  └─ TRACKING
ANTENNA_POINTING
  ├─ ACQUIRING
  └─ TRACKING
EARTH_LIMB_POINTING
  ├─ ACQUIRING
  └─ TRACKING
ATTITUDE_HOLD
  └─ HOLDING
```

Top-level `switchMode: Adcs.Mode` signal inherited by all leaf states.

**Ports consumed from outside subtopology:**
- `StarTrackerManager` attitude port (top-level, precision pointing)
- `GnssManager` position/time port (top-level, timing reference + antenna pointing)
- `modeIn: Sat.AdcsModePort` (from `SatStateMachine`)

**Health monitoring:** `AdcsApplication` is health-monitored. Hardware managers excluded.

### 5.6 Comms Subtopology

**Purpose:** Manages the EnduroSat S-band radio for omni telemetry (always active) and high-gain downlink (Standby/Downlink mode only).

**Components:**

| Component | Type | Purpose |
|-----------|------|---------|
| `CommsApplication` | Active | Hierarchical SM; receives mode from `SatStateMachine`; manages radio operating mode |
| `EnduroSatManager` | Active (worker) | State machine: RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN / error→RESET. Bridges ComCcsds to the S-band radio. |

**CommsApplication modes (received via `Sat.CommsModePort`):**

| Mode | Behavior |
|------|----------|
| `OmniOnly` | Low-rate omni telemetry only. Small packets. Always available. |
| `HighGainDownlink` | Full high-rate downlink. Requires `AntennaPointing` from `AdcsApplication`. |

**Health monitoring:** `CommsApplication` is health-monitored. `EnduroSatManager` excluded.

### 5.7 EPS Subtopology

**Purpose:** Monitors battery and power system health, accepts power configuration and panel deployment commands from ground and `SatStateMachine`, and publishes power state to `SatStateMachine` for submode decisions. Runs continuously independent of satellite mode.

**Components:**

| Component | Type | Purpose |
|-----------|------|---------|
| `EPSApplication` | Active | Command-driven orchestrator; reads battery state from `MpptIcManager`; exposes `powerStateGet` synchronous get port read by `SatStateMachine`; forwards `SET_IC_REGISTER` commands to `MpptIcManager` via `setRegister` port; forwards deploy command to `DeployPanelsManager`. No mode interface. |
| `MpptIcManager` | Active (worker) | Sole owner of BQ25756 IC over I2C; custom two-state SM: UNINITIALIZED → RUNNING; reads measurements each tick; handles IC fault recovery via INT interrupt |
| `WatchdogPinger` | Passive | Toggles hardware watchdog GPIO pin on each rate group tick |
| `DeployPanelsManager` | Active | Two-state SM: NOT_DEPLOYED → DEPLOYED; executes burn wire sequence in both states; emits WARNING_HI on re-attempt in DEPLOYED state |

`SatStateMachine` retrieves the latest power state from `EPSApplication` each 1 Hz tick by invoking the `powerStateGet` synchronous get port; the returned struct drives submode activation decisions.

**Health monitoring:** `EPSApplication` is health-monitored. Hardware managers excluded.

### 5.8 Thermal Subtopology

| Component | Type | Purpose |
|-----------|------|---------|
| `ThermalApplication` | Active | Hierarchical SM (NoHeating / ActiveHeating); receives mode from `SatStateMachine`; reads all temperature sensor data each tick; runs PID control loop in ActiveHeating; commands duty cycle to `HeaterManager` |
| `TemperatureSensorManager` | Queued (worker) | State machine: RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN / error→RESET; owns all temperature sensors via `LinuxI2cDriver` |
| `HeaterManager` | Queued (worker) | State machine: RESET → CONFIGURE → RUN / error→RESET; owns the PWM heater channel via `LinuxPwmDriver` |

---

## 6. Top-Level Standalone Components

`SatStateMachine` (Layer 4) is described in §8. The components below are instantiated at the top-level topology because they are shared across multiple subtopologies or provide satellite-wide scheduling and bus infrastructure.

| Component | Layer | Type | Purpose |
|-----------|-------|------|---------|
| `StarTrackerManager` | 2 | Active (worker) | Shared by DataCollectionApplication and AdcsApplication |
| `GnssManager` | 2 | Active (worker) | Shared by DataCollectionApplication, AdcsApplication, and SatStateMachine |
| `RateGroupDriver` | — | Passive | Divides hardware timer interrupt into multiple rate signals |
| `RateGroup1` | — | Active | 10 Hz scheduling |
| `RateGroup2` | — | Active | 1 Hz scheduling |
| `RateGroup3` | — | Active | 0.1 Hz scheduling |
| `LinuxI2cDriver` (×N) | 1 | Passive | One instance per I2C bus |
| `LinuxSpiDriver` (×N) | 1 | Passive | One instance per SPI bus |
| `LinuxUartDriver` (×N) | 1 | Passive | One instance per UART (star tracker, GNSS, radio) |
| `LinuxGpioDriver` (×N) | 1 | Passive | One instance per GPIO group (watchdog, deploy panels) |

---

## 7. Rate Group Scheduling

| Rate Group | Frequency | Scheduled Components |
|------------|-----------|---------------------|
| `RateGroup1` | 10 Hz | `AdcsApplication`, `ImuManager`, `SunSensorManager`, `MagnetorquerManager`, `WatchdogPinger` |
| `RateGroup2` | 1 Hz | `SatStateMachine`, `EPSApplication`, `MpptIcManager`, `ThermalApplication`, `TemperatureSensorManager`, `HeaterManager`, `DataCollectionApplication` (availability check), `Health` |
| `RateGroup3` | 0.1 Hz | `StarTrackerManager`, `GnssManager`, `ScienceInferenceApplication`, `SystemResources`, `FileDownlink` |

---

## 8. SatStateMachine Design

`SatStateMachine` is an Active component. It evaluates all submode conditions each 1 Hz tick and sends the resulting mode to every application component via dedicated typed ports.

### Mode Output Ports

One typed output port per application component. Each port carries that application's own mode enum. `SatStateMachine` owns the translation table.

```fpp
module Sat {
    port AdcsModePort(mode: Adcs.Mode)
    port DataColModePort(mode: DataCollection.Mode)
    port ScienceInferenceModePort(mode: ScienceInference.Mode)
    port CommsModePort(mode: Comms.Mode)
}
```

Application components have no knowledge of `Sat::Mode` or `Sat::StandbySubmode`. They only receive and act on their own mode enum.

### Condition Inputs

| Port | Direction | Source | Data |
|------|-----------|--------|------|
| `powerStateGet` | Output (sync get into `EPSApplication`) | `EPSApplication` | State of charge + above/below threshold (returned struct) |
| `sunEclipseIn` | Input | Sun sensors + `GnssManager` | In sun / in eclipse |
| `orbitStateIn` | Input | `GnssManager` | Over ground station flag |
| `downlinkQueueDepthIn` | Input | `ComQueue` | Current queue depth (bytes) |

### Translation Table

| Satellite State | `AdcsApplication` | `DataCollectionApplication` | `ScienceInferenceApplication` | `CommsApplication` |
|----------------|-------------------|----------------------------|-------------------------------|-------------------|
| Safe | Detumble | Off | Off | OmniOnly |
| Standby/Downlink | AntennaPointing | Off | Off | HighGainDownlink |
| Standby/Science | EarthLimbPointing | RunExperiment | ProcessImages | OmniOnly |
| Standby/Charge | SunPointing | Off | Off | OmniOnly |
| Standby/Eclipse | AttitudeHold | Off | Off | OmniOnly |

### Mode Transitions

| From | To | Trigger |
|------|----|---------|
| Safe | Standby | Ground command `SAFE_EXIT` (only after checkout completed) |
| Any | Safe | Ground command `SAFE_MODE`; vbatt below `CRITICAL_THRESHOLD` in `powerState` from `EPSApplication` evaluated by `SatStateMachine` each 1 Hz tick. `EPSApplication` does not emit `FATAL`; the transition is a normal `SatStateMachine` mode change, not a `fatalHandler` reboot. Any `FATAL` from elsewhere (e.g. `AssertFatalAdapter`, health timeout) still routes through `EventManager.FatalAnnounce → fatalHandler` and reboots into Safe. |
| Standby | submode | Condition evaluation each 1 Hz tick |

**Events emitted:** mode and submode entry/exit events for every transition.

**Health checked:** Yes.

---

## 9. Application Component State Machine Pattern

All application components use **hierarchical F' state machines** (`Fw::Sm`) where:

- **Mode is the top-level state** — each mode is a parent state in the SM
- **Operational substates are nested inside** each mode
- **A single `switchMode` signal** is defined once at the top level and inherited by all leaf states (per FPP inherited transitions — `nasa/fpp` [`Defining-State-Machines.adoc#inherited-transitions`](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#inherited-transitions))
- **Entry/exit actions** follow the FPP Least Common Ancestor rule automatically — mode switches correctly unwind and re-enter
- **Each mode re-entry always starts from its `initial` substate** — no history

Each parent state requires one `initial` specifier per FPP rules ([`#substates`](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#substates)).

**Idempotency:** Application components ignore mode-switch calls for the mode already active.

**Exception:** `EPSApplication` does not follow this pattern. It has no mode port and no hierarchical SM. It operates continuously as a command-driven Active component with no satellite-mode-driven state transitions. See §5.6.

---

## 10. Hardware Manager State Machine Pattern

All hardware managers use a single flat F' state machine following the `fprime-community/fprime-sensors` `ImuManager` pattern:

```
RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN
  ↑_____________ error from any state ___________|
```

- Driven by rate group tick (`schedIn`)
- Each state action calls a helper function returning bus status (`Drv::I2cStatus` or equivalent)
- On error: log `WARNING_HI` (throttled), send `error` signal → back to RESET (self-healing)
- Configuration via F' parameters — `parameterUpdated()` sends `reconfigure` signal → RUN → CONFIGURE
- No satellite mode awareness

Reference: [`fprime-community/fprime-sensors/ImuManager`](https://github.com/fprime-community/fprime-sensors/tree/devel/fprime-sensors/MpuImu/Components/ImuManager)

---

## 11. Key Cross-Subtopology Wiring

| Source | Destination | Data |
|--------|-------------|------|
| `StarTrackerManager` | `DataCollectionApplication` | Attitude reading (synchronized capture) |
| `StarTrackerManager` | `AdcsApplication` | Attitude reading (precision pointing) |
| `GnssManager` | `DataCollectionApplication` | Position + time (synchronized capture) |
| `GnssManager` | `AdcsApplication` | Position + timing reference |
| `GnssManager` | `SatStateMachine` | Orbital state (over ground station, sun/eclipse) |
| `GnssManager` | Time services | PPS timing signal |
| `EPSApplication.powerStateGet` (sync get) | `SatStateMachine` | Battery state of charge (pulled by `SatStateMachine` each 1 Hz tick) |
| `SatStateMachine.adcsModeOut` | `AdcsApplication` | Mode command (`Adcs.Mode`) |
| `SatStateMachine.dataColModeOut` | `DataCollectionApplication` | Mode command (`DataCollection.Mode`) |
| `SatStateMachine.scienceInferenceModeOut` | `ScienceInferenceApplication` | Mode command (`ScienceInference.Mode`) |
| `SatStateMachine.commsModeOut` | `CommsApplication` | Mode command (`Comms.Mode`) |
| `EnduroSatManager` | `ComCcsds` | Uplink/downlink byte stream |
| `DataCollection` | `DataProducts` | Science result data products |
| `DataCollection` | `FileHandling` | Flagged image files |

---

## 12. Health Monitoring Summary

| Component | Subtopology |
|-----------|-------------|
| `SatStateMachine` | Top-level |
| `DataCollectionApplication` | DataCollection |
| `ScienceInferenceApplication` | ScienceInference |
| `AdcsApplication` | ADCS |
| `CommsApplication` | Comms |
| `EPSApplication` | EPS |
| `ThermalApplication` | Thermal |
| `cmdDisp` | CdhCore |
| `events` | CdhCore |

All hardware managers and workers excluded from health monitoring.

---

## 13. Design Pattern References

| Pattern | Applied To | F' Documentation |
|---------|-----------|-----------------|
| App-Manager-Driver | All subsystems | `docs/user-manual/design-patterns/app-man-drv.md` |
| Hierarchical State Machine | All application components | [`nasa/fpp Defining-State-Machines.adoc#substates`](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#substates) |
| Hardware Manager SM (flat) | All hardware managers | [`fprime-sensors/ImuManager`](https://github.com/fprime-community/fprime-sensors/tree/devel/fprime-sensors/MpuImu/Components/ImuManager) |
| Subtopologies | DataCollection, ScienceInference, ADCS, Comms, EPS (Layer 3) + CdhCore (Layer 5) + 3 pre-built Layer 3 | `docs/user-manual/design-patterns/subtopologies.md` |
| Rate Groups | RateGroup1/2/3 | `docs/user-manual/design-patterns/rate-group.md` |
| Health Checking | All application-level components | `docs/user-manual/design-patterns/health-checking.md` |
| Callback Ports | Synchronized capture in DataCollectionApplication | `docs/user-manual/design-patterns/common-port-patterns.md` |
| Data Products | Science algorithm results | `docs/user-manual/framework/data-products.md` |

---

## 14. External Library Integration

| Library | Algorithm | Camera | Experiment Type | Integration |
|---------|-----------|--------|----------------|-------------|
| LOST | Optical navigation | Camera 1 | L&F | CMake; called from `ScienceInferenceApplication` |
| FOUND | Optical navigation | Camera 2 | L&F | CMake; called from `ScienceInferenceApplication` |
| SCOPE | Star catalog processing (runs LOST internally) | Camera 1 + 2 | Calibration | CMake; called from `ScienceInferenceApplication` |

All libraries are C++ and included as CMake dependencies. `ScienceInferenceApplication` selects the algorithm based on experiment type metadata stored with each image.
