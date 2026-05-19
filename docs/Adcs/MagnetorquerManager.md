# MagnetorquerManager SDD

## 1. Overview

`MagnetorquerManager` is the Layer 2 hardware manager for the magnetorquer coils within the ADCS subtopology. It is an **actuator manager** — it receives a magnetic moment vector from `AdcsApplication`, converts it to per-axis PWM duty cycles, and drives the H-bridge current driver via six PWM signals (two per axis: one signal per current direction).

`MagnetorquerManager` is a **Queued** component driven by `RateGroup1` (10 Hz). It has no satellite mode awareness and performs no control law computation. On receiving a moment vector, it computes the corresponding duty cycles and applies them as a burst over N ticks; after the burst completes it idles (zero current on all axes) until the next moment vector arrives. This burst-then-idle pattern allows the magnetometer to read an uncontaminated B-field between actuation pulses.

Each H-bridge axis requires two PWM inputs for bidirectional current control: driving the positive-direction channel to the desired duty cycle while holding the negative-direction channel at zero produces current in one direction; swapping which channel is non-zero reverses the current. No GPIO pins are required. All bus access goes through six `LinuxPwmDriver` instances (one per PWM channel).

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-MTQ-001 | MagnetorquerManager shall reset the coil driver to a known safe state (zero current on all axes) before any configuration is applied | Inspection |
| HS2-MTQ-002 | MagnetorquerManager shall wait for the driver reset stabilization period before proceeding to enable | Inspection |
| HS2-MTQ-003 | MagnetorquerManager shall set the PWM period and enable all six PWM channels before entering the run loop | Inspection |
| HS2-MTQ-004 | MagnetorquerManager shall configure PWM frequency from F' parameters before entering the run loop | Inspection |
| HS2-MTQ-005 | MagnetorquerManager shall zero all coil currents on entry to RESET | Inspection |
| HS2-MTQ-006 | MagnetorquerManager shall accept a magnetic moment vector from AdcsApplication via a sync input port while in RUN | Inspection |
| HS2-MTQ-007 | MagnetorquerManager shall convert the received magnetic moment vector to per-axis PWM duty cycles | Inspection |
| HS2-MTQ-008 | MagnetorquerManager shall apply the computed duty cycles for BURST_TICKS ticks, then idle (zero current) until the next moment vector | Inspection |
| HS2-MTQ-009 | MagnetorquerManager shall log a WARNING_HI event and transition to RESET on any bus write failure | Inspection |
| HS2-MTQ-010 | MagnetorquerManager shall track consecutive failure count and report it as a telemetry channel | Inspection |
| HS2-MTQ-011 | MagnetorquerManager shall apply updated F' parameter values by transitioning from RUN to CONFIGURE without a full reset | Inspection |
| HS2-MTQ-012 | MagnetorquerManager shall not perform any bus operations while in WAIT_RESET | Inspection |
| HS2-MTQ-013 | MagnetorquerManager shall transition to OFF from any state on receipt of an OFF command via controlIn | Inspection |
| HS2-MTQ-014 | MagnetorquerManager shall write zero current to all three coil axes on OFF entry | Inspection |
| HS2-MTQ-015 | MagnetorquerManager shall perform no bus operations while in OFF | Inspection |
| HS2-MTQ-016 | MagnetorquerManager shall re-enter RESET on receipt of an ON command via controlIn while in OFF | Inspection |
| HS2-MTQ-017 | MagnetorquerManager OFF entry hardware shutdown sequence beyond coil zeroing is TBD pending H-bridge hardware selection | Deferred |

---

## 3. Design

### 3.1 Component Type

Queued component with internal flat F' state machine (`Fw::Sm`). Has a message queue but no dedicated thread — executes on the `RateGroup1` caller thread each 10 Hz tick.

### 3.2 Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `PWM_FREQUENCY` | `U32` | PWM carrier frequency for the H-bridge driver (Hz) |
| `BURST_TICKS` | `U32` | Number of ticks to apply duty cycles after receiving a moment vector |
| `RESET_WAIT_TICKS` | `U32` | Number of rate-group ticks to wait after reset before proceeding |

### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | 10 Hz rate group tick — drives SM step and burst countdown |
| `momentVectorIn` | Input (sync) | `Adcs.MomentVectorPort` | Magnetic moment vector from AdcsApplication; converted to duty cycles |
| `controlIn` | Input (async) | `Adcs.ManagerControlPort` | ON/OFF command from AdcsApplication |
| `pwmSetPeriodOut[6]` | Output | `Drv.PwmSetPeriod` | Set PWM period on all six channels during CONFIGURE |
| `pwmSetDutyCycleOut[6]` | Output | `Drv.PwmSetDutyCycle` | Duty cycle control — channels 0–1: X-axis (±), 2–3: Y-axis (±), 4–5: Z-axis (±) |
| `pwmEnableOut[6]` | Output | `Drv.PwmEnable` | Enable all six channels on CONFIGURE entry; disable on error before → RESET |
| `prmGet` | Output | `Fw.PrmGet` | Load parameters from PrmDb during CONFIGURE |
| `logOut` | Output | `Fw.Log` | Event logging (state transitions, errors) |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (SM state, applied duty cycles per axis, consecutive failure count) |

### 3.4 Commands

`MagnetorquerManager` accepts no ground commands. It is not health-monitored. All recovery is handled autonomously by the self-healing SM or escalated via telemetry to `AdcsApplication`.

---

## 4. State Machine

`MagnetorquerManager` uses a single flat F' state machine. All states are peers — no nesting. The SM is stepped by `schedIn` each tick.

```
OFF
  entry: call pwmSetDutyCycleOut(0) on all six channels
         [further shutdown protocol TBD]
  on tick: no action
  on controlIn(ON) → RESET

RESET
  entry: call pwmSetDutyCycleOut(0) on all six channels
         reset consecutive-failure counter
         reset wait-tick counter
         reset burst tick counter
  on tick → WAIT_RESET

WAIT_RESET
  on tick: increment wait-tick counter
           if counter >= RESET_WAIT_TICKS → CONFIGURE
           (no bus operations)

CONFIGURE
  entry: call pwmSetDutyCycleOut(0) on all six channels
         load PWM_FREQUENCY, BURST_TICKS from PrmDb
         call pwmSetPeriodOut(periodNs) on all six channels
         call pwmEnableOut(HIGH) on all six channels
  on tick:
    if all writes OK → RUN
    if any write error → log WARNING_HI, increment failure count,
                         call pwmEnableOut(LOW) on all six channels → RESET

RUN
  on momentVectorIn: compute per-axis PWM duty cycles from moment vector
                     set burst counter to BURST_TICKS
  on tick: if burst counter > 0:
             for each axis, write computed duty cycle to the positive-direction
             channel and zero to the negative-direction channel (or vice versa
             for negative moment); zero both channels if moment component is zero
             decrement burst counter
             emit applied duty cycles as telemetry
             reset consecutive-failure counter on success
           if burst counter == 0:
             call pwmSetDutyCycleOut(0) on all six channels (coils idle)
           on any write error:
             log WARNING_HI (throttled), increment failure count → RESET

# Global signals — reachable from any state:
on error signal       → RESET
on controlIn(OFF)     → OFF
on controlIn(ON)      → RESET   # no-op from non-OFF states
on reconfigure signal → CONFIGURE
```

**Burst-then-idle:** After each moment vector is received, duty cycles are applied for exactly `BURST_TICKS` ticks. The coils then idle at zero current until the next `momentVectorIn`. This ensures the magnetometer can sample an uncontaminated B-field between actuation pulses.

**Safe zero-current on RESET and OFF:** Every entry to `RESET` or `OFF` immediately calls `pwmSetDutyCycleOut(0)` on all six channels before doing anything else.

**Error self-healing:** Any bus write failure emits a throttled `WARNING_HI` event, increments `consecutiveFailures`, zeroes coil currents, and re-enters `RESET`.

**Repeated failure escalation:** `MagnetorquerManager` does not cut power to itself. If `consecutiveFailures` exceeds a threshold observable by `AdcsApplication` via telemetry, `AdcsApplication` is responsible for commanding an EPS power rail cut.

**Parameter reconfiguration:** When `parameterUpdated()` is called, `MagnetorquerManager` sends a `reconfigure` signal. If currently in `RUN`, this transitions directly to `CONFIGURE`. From any other state the signal is ignored.

Reference: [`fprime-community/fprime-sensors/ImuManager`](https://github.com/fprime-community/fprime-sensors/tree/devel/fprime-sensors/MpuImu/Components/ImuManager), [FPP flat state machines](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc)

---

## 5. Notes

- `MagnetorquerManager` is instantiated inside the ADCS subtopology. Its `pwmSetDutyCycleOut[6]`, `pwmSetPeriodOut[6]`, and `pwmEnableOut[6]` port arrays connect to six `LinuxPwmDriver` instances at the top-level topology.
- `momentVectorIn` is connected from `AdcsApplication` within the ADCS subtopology.
- `MagnetorquerManager` is **excluded from health monitoring** (`Svc::Health`). Only `AdcsApplication` is health-checked.
- Moment-vector to duty-cycle conversion (mapping A·m² to PWM duty cycle percentage) is an implementation detail for the component's C++ source.
- Deferred: exact `consecutiveFailures` threshold that triggers `AdcsApplication` to cut magnetorquer power is a system-level parameter to be defined during detailed design.
- Deferred: OFF entry hardware shutdown sequence beyond coil zeroing depends on H-bridge hardware selection.
