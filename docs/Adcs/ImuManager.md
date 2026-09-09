# ImuManager SDD

## 1. Overview

`ImuManager` is the Layer 2 hardware manager for the IMU within the ADCS subtopology. It owns all bus communication with the IMU (resetting, enabling, configuring, and reading the device) and publishes raw angular rate and linear acceleration samples to `AdcsApplication` each rate-group tick.

`ImuManager` is a **Queued** component driven by `RateGroup1` (10 Hz). It has no satellite mode awareness; it runs its startup and read loop unconditionally once initialized, driven entirely by the rate group tick. `AdcsApplication` (Layer 3) is responsible for interpreting failures and, if necessary, commanding a power cut to the IMU via EPS.

The IMU connects to the flight computer via SPI or I2C (bus assignment fixed at integration time). All bus access goes through the Layer 1 `LinuxSpiDriver` or `LinuxI2cDriver`. `ImuManager` has no direct hardware knowledge beyond register addresses.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-IMU-001 | ImuManager shall reset the IMU to a known default state before any configuration is applied | Inspection |
| HS2-IMU-002 | ImuManager shall wait for the IMU reset period to complete before proceeding to enable | Inspection |
| HS2-IMU-003 | ImuManager shall enable the IMU gyroscope and accelerometer via the power management register | Inspection |
| HS2-IMU-004 | ImuManager shall configure the IMU measurement range, digital low-pass filter, and sample rate divider from F' parameters before entering the read loop | Inspection |
| HS2-IMU-005 | ImuManager shall read angular rate and linear acceleration from the IMU data registers each rate-group tick while in RUN | Inspection |
| HS2-IMU-006 | ImuManager shall publish a valid IMU data sample to AdcsApplication after each successful read | Inspection |
| HS2-IMU-007 | ImuManager shall log a WARNING_HI event and transition to RESET on any bus read or write failure | Inspection |
| HS2-IMU-008 | ImuManager shall track consecutive failure count and report it as a telemetry channel | Inspection |
| HS2-IMU-009 | ImuManager shall apply updated F' parameter values by transitioning from RUN to CONFIGURE without a full reset | Inspection |
| HS2-IMU-010 | ImuManager shall not perform any bus operations while in WAIT_RESET | Inspection |
| HS2-IMU-011 | ImuManager shall transition to OFF from any state on receipt of an OFF command via controlIn | Inspection |
| HS2-IMU-012 | ImuManager shall attempt a software power-down of the IMU on OFF entry, with hardware reset via bus as fallback if the software command fails | Inspection |
| HS2-IMU-013 | ImuManager shall perform no bus operations while in OFF | Inspection |
| HS2-IMU-014 | ImuManager shall re-enter RESET on receipt of an ON command via controlIn while in OFF | Inspection |

---

## 3. Design

### 3.1 Component Type

Queued component with internal flat F' state machine (`Fw::Sm`). Has a message queue but no dedicated thread. It executes on the `RateGroup1` caller thread each 10 Hz tick.

### 3.2 Parameters

`ImuManager` is not satellite-mode-aware and receives no mode commands. Configuration is entirely parameter-driven. Parameters are loaded from `Svc::PrmDb` in the `CONFIGURE` state. A parameter update at runtime sends a `reconfigure` signal, causing `ImuManager` to re-enter `CONFIGURE` from `RUN` without a full reset.

| Parameter | Type | Description |
|-----------|------|-------------|
| `GYRO_RANGE` | `U8` | Gyroscope full-scale range selection (device-specific register value) |
| `ACCEL_RANGE` | `U8` | Accelerometer full-scale range selection |
| `DLPF_CONFIG` | `U8` | Digital low-pass filter bandwidth setting |
| `SAMPLE_RATE_DIV` | `U8` | Sample rate divider register value |
| `RESET_WAIT_TICKS` | `U32` | Number of rate-group ticks to wait after issuing reset before proceeding |

### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | 10 Hz rate group tick that drives the SM step |
| `busWrite` | Output | `Drv.I2cWrite` | Write to IMU register (reset, enable, configure) |
| `busWriteRead` | Output | `Drv.I2cWriteRead` | Write register address then read data (register reads in RUN) |
| `imuDataOut` | Output | `Adcs.ImuDataPort` | Publish raw angular rate + linear acceleration sample to AttitudeFilter (queried by AdcsApplication via its `filter*Get` ports, not read directly from ImuManager) |
| `controlIn` | Input (async) | `Adcs.ManagerControlPort` | ON/OFF command from AdcsApplication |
| `prmGet` | Output | `Fw.PrmGet` | Load parameters from PrmDb during CONFIGURE |
| `logOut` | Output | `Fw.Log` | Event logging (state transitions, errors) |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (SM state, consecutive failure count, last sample values) |

> **Bus type:** If the IMU is wired to SPI, `busWrite` and `busWriteRead` map to the corresponding `Drv.SpiReadWrite` port on `LinuxSpiDriver`. Port names remain the same; the driver type is determined at topology assembly.

### 3.4 Commands

`ImuManager` accepts no ground commands. It is not health-monitored. All recovery is handled autonomously by the self-healing SM or escalated via telemetry to `AdcsApplication`.

---

## 4. State Machine

`ImuManager` uses a single flat F' state machine following the `fprime-community/fprime-sensors` `ImuManager` reference pattern. All states are peers with no nesting. The SM is stepped by `schedIn` each tick.

```
OFF
  entry: attempt software power-down - write sleep bit to IMU power management register
         if busWrite OK: device is in low-power mode
         if busWrite fails: attempt hardware reset via bus (assert reset line or write hard-reset register)
  on tick: no action (no bus operations)
  on controlIn(ON) → RESET

RESET
  entry: write reset command to IMU power management register
         reset consecutive-failure counter
         reset wait-tick counter
  on tick → WAIT_RESET

WAIT_RESET
  on tick: increment wait-tick counter
           if counter >= RESET_WAIT_TICKS → ENABLE
           (no bus operations performed in this state)

ENABLE
  entry: write power management register to enable gyroscope + accelerometer
         (clear sleep bit, select internal oscillator)
  on tick:
    if busWrite OK → CONFIGURE
    if busWrite error → log WARNING_HI, increment failure count → RESET

CONFIGURE
  entry: load GYRO_RANGE, ACCEL_RANGE, DLPF_CONFIG, SAMPLE_RATE_DIV from PrmDb
         write gyroscope configuration register
         write accelerometer configuration register
         write DLPF configuration register
         write sample rate divider register
  on tick:
    if all writes OK → RUN
    if any write error → log WARNING_HI, increment failure count → RESET

RUN
  on tick: write data register address, read 12 bytes (gyro XYZ + accel XYZ)
           if busWriteRead OK:
             publish imuDataOut (angular rate, linear acceleration, timestamp)
             reset consecutive-failure counter
           if busWriteRead error:
             log WARNING_HI (throttled), increment failure count → RESET

# Global signals - reachable from any state:
on error signal       → RESET       # existing - self-healing fallback
on reconfigure signal → CONFIGURE   # parameter update path from RUN
on controlIn(OFF)     → OFF         # new - immediate from any state, including mid-startup
on controlIn(ON)      → RESET       # new - only meaningful from OFF; no-op from all other states
```

```mermaid
stateDiagram-v2
    [*] --> RESET
    RESET --> WAIT_RESET: on tick
    WAIT_RESET --> ENABLE: wait elapsed
    ENABLE --> CONFIGURE: busWrite OK
    CONFIGURE --> RUN: writes OK
    RUN --> CONFIGURE: reconfigure
```

**Error recovery:** any of `ENABLE`, `CONFIGURE`, or `RUN` return directly to `RESET` on a bus error.

```mermaid
stateDiagram-v2
    ENABLE --> RESET: error
    CONFIGURE --> RESET: error
    RUN --> RESET: error
```

**OFF power control:**

```mermaid
stateDiagram-v2
    RESET --> OFF: controlIn(OFF)
    WAIT_RESET --> OFF: controlIn(OFF)
    ENABLE --> OFF: controlIn(OFF)
    CONFIGURE --> OFF: controlIn(OFF)
    RUN --> OFF: controlIn(OFF)
```

**Powering back on:** `OFF --> RESET` on `controlIn(ON)`, from any state.

**Error self-healing:** Any bus error from `ENABLE`, `CONFIGURE`, or `RUN` emits a `WARNING_HI` event, increments the `consecutiveFailures` telemetry channel, and re-enters `RESET`. The SM will retry the full startup sequence automatically on the next tick cycle.

**Repeated failure escalation:** `ImuManager` does not cut power to itself. If `consecutiveFailures` exceeds a threshold observable by `AdcsApplication` via telemetry, `AdcsApplication` is responsible for commanding an EPS power rail cut. `ImuManager` continues attempting self-healing until then.

**Parameter reconfiguration:** When `parameterUpdated()` is called (F' framework callback), `ImuManager` sends a `reconfigure` signal. If currently in `RUN`, this transitions directly to `CONFIGURE` without a reset, preserving uptime. From any other state the signal is ignored (configuration will apply on next natural entry to `CONFIGURE`).

Reference: [`fprime-community/fprime-sensors/ImuManager`](https://github.com/fprime-community/fprime-sensors/tree/devel/fprime-sensors/MpuImu/Components/ImuManager), [FPP flat state machines](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc)

---

## 5. Notes

- `ImuManager` is instantiated inside the ADCS subtopology. Its `busWrite` and `busWriteRead` ports connect to a `LinuxI2cDriver` or `LinuxSpiDriver` instance at the top-level topology, depending on the IMU's physical bus.
- `imuDataOut` connects to `AttitudeFilter` within the ADCS subtopology, not to `AdcsApplication` directly. `AdcsApplication` reads the fused result back out via its `filter*Get` ports.
- `ImuManager` is **excluded from health monitoring** (`Svc::Health`). Only `AdcsApplication` is health-checked.
- The `Adcs.ImuDataPort` type (carrying timestamped angular rate and linear acceleration vectors) is defined in the ADCS module and shared with `AdcsApplication`.
- Specific IMU register addresses and power management register layout depend on the selected IMU device (e.g., MPU-6000 or equivalent); these are implementation details for the component's C++ source, not specified here.
- `RESET_WAIT_TICKS` must be set to cover the IMU's datasheet-specified reset stabilization time at 10 Hz tick rate (e.g., 10 ms reset time → 1 tick minimum; add margin).
- Deferred: exact `consecutiveFailures` threshold that triggers `AdcsApplication` to cut IMU power is a system-level parameter to be defined during detailed design.
- `controlIn` is driven by `AdcsApplication`'s `imuControl` output port; `AdcsApplication` sends `OFF` on entry to its own `Off` mode and `ON` on exit, per `docs/superpowers/specs/2026-05-05-adcs-hardware-manager-off-state-design.md`.
