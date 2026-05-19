# ADCS Hardware Manager OFF State — Design Spec

**Date:** 2026-05-05
**Scope:** `ImuManager`, `SunSensorManager`, `MagnetorquerManager`, `AdcsApplication`

---

## 1. Problem

The three ADCS hardware managers (`ImuManager`, `SunSensorManager`, `MagnetorquerManager`) currently run unconditionally — once started they self-heal forever with no way to stop. `AdcsApplication` already has an `OFF` mode, but nothing commands the managers to cease bus operations when that mode is entered. This leaves hardware active and consuming power when the ADCS subsystem is intentionally off.

---

## 2. New Shared ADCS Type

One enum and one port type, defined once in the ADCS module and shared by all three managers and `AdcsApplication`:

```fpp
module Adcs {
    enum ManagerControl { ON, OFF }
    port ManagerControlPort(control: ManagerControl)
}
```

---

## 3. New Port on Each Manager

Each of the three managers gains one new async input port:

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `controlIn` | Input (async) | `Adcs.ManagerControlPort` | ON/OFF command from AdcsApplication |

`AdcsApplication` gains three new output ports connected to the respective manager `controlIn` ports within the ADCS subtopology:

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `imuControl` | Output | `Adcs.ManagerControlPort` | ON/OFF to ImuManager |
| `sunSensorControl` | Output | `Adcs.ManagerControlPort` | ON/OFF to SunSensorManager |
| `magnetorquerControl` | Output | `Adcs.ManagerControlPort` | ON/OFF to MagnetorquerManager |

---

## 4. State Machine Changes (Common Pattern)

The same SM change applies to all three managers. `OFF` is added as a new peer state. Two global signals are added alongside the existing `error` global signal:

```
OFF
  entry: [per-component shutdown actions — see Section 5]
  on tick: no action (no bus operations)
  on controlIn(ON) → RESET

RESET / WAIT_RESET / ENABLE / CONFIGURE / RUN
  (unchanged)

# Global signals — reachable from any state:
on error signal       → RESET   # existing
on controlIn(OFF)     → OFF     # new — immediate from any state, including mid-startup
on controlIn(ON)      → RESET   # new — only meaningful from OFF; no-op from all other states
```

- `controlIn(OFF)` is dispatched when `controlIn` receives `ManagerControl.OFF`.
- `controlIn(ON)` is dispatched when `controlIn` receives `ManagerControl.ON`.
- `controlIn(ON)` from any non-OFF state is a no-op — the manager is already running.

---

## 5. Per-Component OFF Entry Behavior

### ImuManager
The IMU has an onboard computer with a software power-down register:

```
OFF entry:
  attempt software power-down — write sleep bit to IMU power management register
  if busWrite OK: device is in low-power mode
  if busWrite fails: attempt hardware reset via bus (assert reset line or write hard-reset register)
  no further bus operations until ON signal
```

### SunSensorManager
TBD — no onboard computer. Exact power-down protocol depends on ADC hardware selection.
Minimum behavior on OFF entry: zero all channel intensity buffers, stop all bus operations.

### MagnetorquerManager
TBD — no onboard computer. Exact shutdown protocol depends on H-bridge hardware.
Minimum behavior on OFF entry (safety-critical): **write zero current to all three coil axes** (same action already performed on RESET entry). No further bus operations until ON signal.

---

## 6. AdcsApplication Changes

`AdcsApplication`'s `OFF` state gains entry and exit actions to coordinate the managers:

```
OFF
  entry: send ManagerControl.OFF via imuControl, sunSensorControl, magnetorquerControl
  exit:  send ManagerControl.ON  via imuControl, sunSensorControl, magnetorquerControl
```

The exit action fires on any `switchMode` signal leaving `OFF`, so all three managers re-enter `RESET` regardless of which active mode `AdcsApplication` transitions into.

Three new requirements are added to `AdcsApplication`:

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-ADC-011 | AdcsApplication shall command all ADCS hardware managers to OFF on entry to its OFF mode | Inspection |
| HS2-ADC-012 | AdcsApplication shall command all ADCS hardware managers to ON on exit from its OFF mode | Inspection |
| HS2-ADC-013 | AdcsApplication shall provide one `Adcs.ManagerControlPort` output port per ADCS hardware manager | Inspection |

---

## 7. New Manager Requirements

Requirements to add to each manager SDD:

### ImuManager
| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-IMU-011 | ImuManager shall transition to OFF from any state on receipt of an OFF command via controlIn | Inspection |
| HS2-IMU-012 | ImuManager shall attempt a software power-down of the IMU on OFF entry, with hardware reset via bus as fallback if the software command fails | Inspection |
| HS2-IMU-013 | ImuManager shall perform no bus operations while in OFF | Inspection |
| HS2-IMU-014 | ImuManager shall re-enter RESET on receipt of an ON command via controlIn while in OFF | Inspection |

### SunSensorManager
| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-SSM-012 | SunSensorManager shall transition to OFF from any state on receipt of an OFF command via controlIn | Inspection |
| HS2-SSM-013 | SunSensorManager shall perform no bus operations while in OFF | Inspection |
| HS2-SSM-014 | SunSensorManager shall re-enter RESET on receipt of an ON command via controlIn while in OFF | Inspection |
| HS2-SSM-015 | SunSensorManager OFF entry hardware shutdown sequence is TBD pending ADC hardware selection | Deferred |

### MagnetorquerManager
| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-MTQ-013 | MagnetorquerManager shall transition to OFF from any state on receipt of an OFF command via controlIn | Inspection |
| HS2-MTQ-014 | MagnetorquerManager shall write zero current to all three coil axes on OFF entry | Inspection |
| HS2-MTQ-015 | MagnetorquerManager shall perform no bus operations while in OFF | Inspection |
| HS2-MTQ-016 | MagnetorquerManager shall re-enter RESET on receipt of an ON command via controlIn while in OFF | Inspection |
| HS2-MTQ-017 | MagnetorquerManager OFF entry hardware shutdown sequence beyond coil zeroing is TBD pending H-bridge hardware selection | Deferred |

---

## 8. Open Items

- SunSensorManager OFF entry hardware protocol — deferred until ADC hardware is selected.
- MagnetorquerManager OFF entry hardware protocol beyond coil zeroing — deferred until H-bridge hardware is selected.
- Whether `on` received in a non-OFF state should be logged at DIAGNOSTIC for observability — left to implementer judgment.
