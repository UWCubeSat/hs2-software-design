# ThermalApplication SDD

## 1. Overview

`ThermalApplication` is the Layer 3 Active component for the Thermal subtopology. It receives a mode command from `SatStateMachine` and executes the satellite's thermal control policy — reading temperature sensor data from `TemperatureSensorManager` each rate group tick and commanding a heater duty cycle to `HeaterManager`.

In `NoHeating` mode the heater is commanded off and the component monitors temperature for out-of-range conditions. In `ActiveHeating` mode a PID control loop runs each tick and modulates the heater duty cycle toward a configurable setpoint. If any sensor reading is invalid the heater is clamped to zero and a WARNING_HI is emitted.

PID setpoint, gains, and temperature alarm thresholds are F' parameters persisted via `PrmDb` and configurable by ground.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|-------------|
| HS2-THA-001 | ThermalApplication shall switch operating mode on command from SatStateMachine | Inspection |
| HS2-THA-002 | ThermalApplication shall command zero heater duty in NoHeating mode | Inspection |
| HS2-THA-003 | ThermalApplication shall run a PID control loop each rate group tick in ActiveHeating | Inspection |
| HS2-THA-004 | ThermalApplication shall clamp heater duty to zero and emit WARNING_HI when sensor data is invalid in ActiveHeating | Inspection |
| HS2-THA-005 | ThermalApplication shall emit WARNING_HI when any temperature reading exceeds its configured alarm threshold | Inspection |
| HS2-THA-006 | ThermalApplication shall not command a heater duty below 0% or above 100% | Inspection |
| HS2-THA-007 | ThermalApplication shall respond to health ping within the required deadline | Inspection |
| HS2-THA-008 | ThermalApplication shall load setpoint, PID coefficients, and alarm thresholds from PrmDb | Inspection |

---

## 3. Design

### 3.1 Component Type

Active component with internal hierarchical F' state machine (`Fw::Sm`).

### 3.2 Mode Interface

`ThermalApplication` receives its operating mode from `SatStateMachine` via a typed port:

```fpp
sync input port modeIn: Sat.ThermalModePort   # carries Thermal.Mode
```

Mode enum (owned by this component's module):

```fpp
module Thermal {
    enum Mode { NoHeating, ActiveHeating }
}
```

If the incoming mode matches the current mode, the handler returns immediately (idempotent).

### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `modeIn` | Input (sync) | `Sat.ThermalModePort` | Mode command from `SatStateMachine` |
| `schedIn` | Input (sync) | `Svc.Sched` | 1 Hz rate group tick; drives control loop |
| `tempReadOut` | Output | `Thermal.TempRead` | Synchronous read of all sensor readings from `TemperatureSensorManager`; returns array of temperatures and validity flags |
| `heaterCmdOut` | Output | `Thermal.HeaterCmd` | Duty cycle command to `HeaterManager` |
| `pingIn` / `pingOut` | In/Out | `Svc.Ping` | Health monitoring |
| `prmGetOut` | Output | `Fw.PrmGet` | Load setpoint, PID gains, and alarm thresholds from PrmDb |
| `cmdIn` / `cmdRegOut` / `cmdResponseOut` | In/Out | `Fw.Cmd` | Ground command reception and registration |
| `logOut` | Output | `Fw.Log` | Event logging |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (temperatures, duty cycle, current mode) |
| `timeGetOut` | Output | `Fw.Time` | Timestamps for events and telemetry |

### 3.4 Commands

| Mnemonic | Args | Description |
|----------|------|-------------|
| `SET_HEATER_OVERRIDE` | `dutyPercent: F32` | Force heater to a fixed duty cycle regardless of PID output |

---

## 4. State Machine

`ThermalApplication` uses a hierarchical F' state machine with two flat leaf states. A single `switchMode: Thermal.Mode` signal defined at the top level is inherited by both states.

```
NO_HEATING
  entry: command 0% duty to HeaterManager; reset PID integrator
  on schedIn: read all temperatures; emit WARNING_HI on any out-of-range reading

ACTIVE_HEATING
  on schedIn: read all temperatures
              if any reading invalid: clamp duty to 0%, reset PID integrator, emit WARNING_HI SENSOR_INVALID
              otherwise: run PID against setpoint; clamp output to [0%, 100%]; command duty to HeaterManager
              emit WARNING_HI on any out-of-range reading

# Inherited by both states:
on switchMode(Thermal.Mode.NoHeating)     → enter NO_HEATING
on switchMode(Thermal.Mode.ActiveHeating) → enter ACTIVE_HEATING
```

Reference: [FPP inherited transitions](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#inherited-transitions)

---

## 5. Notes

- `SatStateMachine` owns the translation from satellite state to `Thermal.Mode`. `ThermalApplication` has no knowledge of `Sat::StandbySubmode` or `Sat::Mode`.
- The PID integrator is reset on entry to `NO_HEATING` and inline whenever a sensor reading is invalid, to prevent integrator windup.
- A reading is marked invalid by `TemperatureSensorManager` on an I2C bus error or a sensor-reported fault status bit. Out-of-range but communicating sensors return `valid = true` and trigger the alarm threshold check separately.
- `ThermalApplication` is health-monitored; `TemperatureSensorManager` and `HeaterManager` are excluded.
- Wired to `RateGroup2` (1 Hz).
- The `Thermal.TempRead` and `Thermal.HeaterCmd` custom port types are defined alongside the component FPP files in the Thermal subtopology source.
