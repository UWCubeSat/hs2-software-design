# CurrentSensorManager SDD

## 1. Overview

`CurrentSensorManager` is the Layer 2 hardware manager for the INA3221 triple-channel current/voltage monitor on the PDS (Power Distribution System) board. It owns all bus communication with the INA3221 — resetting, verifying, configuring, and reading the device — and both publishes per-rail measurements to `EPSApplication` and emits them as telemetry on each rate group tick.

The INA3221 monitors three power rails (12 V, 5 V, 3.3 V), reporting bus voltage and current for each. `CurrentSensorManager` has no satellite mode awareness; it runs its startup and read loop unconditionally once initialized, driven entirely by the rate group tick. All bus access goes through the Layer 1 `LinuxI2cDriver` — `CurrentSensorManager` has no direct hardware knowledge beyond register addresses.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-CSM-001 | CurrentSensorManager shall be the sole flight-software owner of the INA3221 over I2C. | Inspection |
| HS2-CSM-002 | CurrentSensorManager shall read the bus voltage and current of all three rails each rate group tick. | Inspection |
| HS2-CSM-003 | CurrentSensorManager shall publish the assembled rail measurements on railStateOut to EPSApplication each rate group tick. | Inspection |
| HS2-CSM-004 | CurrentSensorManager shall emit telemetry channels for each rail's bus voltage and current. | Inspection |
| HS2-CSM-005 | CurrentSensorManager shall log a WARNING_HI event and transition to RESET on any I2C bus error. | Inspection |

---

## 3. Design

### 3.1 Component Type

Queued component with internal flat F' state machine (`Fw::Sm`).

### 3.2 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | Rate group tick; drives the SM step |
| `busWrite` | Output | `Drv.I2c` | Write INA3221 registers (reset, configure) |
| `busWriteRead` | Output | `Drv.I2cWriteRead` | Write register pointer then read in one repeated-start transaction; channel reads |
| `railStateGet` | Input | Custom `sync_input` port returning railState struct | Per-rail bus voltage and current |
| `cmdIn` | Input | `Fw.Cmd` | Register-access commands via CmdDispatcher |
| `cmdResponseOut` | Output | `Fw.CmdResponse` | Command completion status |
| `logOut` | Output | `Fw.Log` | Event logging (state transitions, I2C errors) |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (per-rail voltage and current) |

### 3.3 Commands

| Mnemonic | Args | Description |
|----------|------|-------------|
| `CURRENT_SENSOR_WRITE_REG16` | `regAddr: INA3221Reg`, `value: U16` | Write to a register |
| `CURRENT_SENSOR_SET_REG16` | `regAddr: INA3221Reg`, `mask: U16` | Read-modify-write: `reg \|= mask` |
| `CURRENT_SENSOR_CLEAR_REG16` | `regAddr: INA3221Reg`, `mask: U16` | Read-modify-write: `reg &= ~mask` |

`CurrentSensorManager` is not health-monitored. Bus-error recovery is handled autonomously by the self-healing SM.

### 3.4 Telemetry

| Channel | Type | Description |
|---------|------|-------------|
| `CH1_VOLTAGE` | `U32` | 12 V rail bus voltage (mV) |
| `CH1_CURRENT` | `F32` | 12 V rail current (mA) |
| `CH2_VOLTAGE` | `U32` | 5 V rail bus voltage (mV) |
| `CH2_CURRENT` | `F32` | 5 V rail current (mA) |
| `CH3_VOLTAGE` | `U32` | 3.3 V rail bus voltage (mV) |
| `CH3_CURRENT` | `F32` | 3.3 V rail current (mA) |

---

## 4. State Machine

`CurrentSensorManager` uses a flat four-state F' state machine following the hardware-manager pattern: `RESET → WAIT_RESET → CONFIGURE → RUN`. The INA3221 has no enable step, so the `ENABLE` state is omitted.

```
RESET
  entry: write the RST bit in the INA3221 CONFIGURATION register to restore register defaults
  on tick → WAIT_RESET

WAIT_RESET
  on tick: one-tick settling delay for the reset to complete → CONFIGURE
           if error → RESET

CONFIGURE
  on tick: write the CONFIGURATION register to set averaging mode and conversion times
    if write OK → RUN
    if write error → log WARNING_HI → RESET

RUN
  on tick: read the bus-voltage and shunt/current registers for all three channels
           assemble railState struct
           call railStateOut port
           emit telemetry channels
           if any busWriteRead error → log WARNING_HI (throttled) → RESET
```

**Error self-healing:** any bus error from `CONFIGURE` or `RUN` emits a `WARNING_HI` event and re-enters `RESET`, retrying the full startup sequence on subsequent ticks.

Reference: [`INA3221Manager` (FeatherCdh)](https://github.com/UWCubeSat), [FPP flat state machines](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc)

---

## 5. Notes

- Current is derived from the INA3221 shunt-voltage registers and the per-channel shunt resistance; exact scaling and shunt values are hardware-team inputs resolved during detailed design.
- `CurrentSensorManager` is instantiated inside the EPS subtopology. Its `busWrite`/`busWriteRead` ports connect to a `LinuxI2cDriver` instance at the top-level topology; `railStateOut` connects to `EPSApplication`.
- `CurrentSensorManager` is **excluded from health monitoring** (`Svc::Health`). Only `EPSApplication` is health-checked.
- The INA3221 is PDS-dependent hardware; it is present only in the flight/PDS configuration.
- In the current hardware, Pin A0 is tied to SCL, meaning that the device I2C address is 0b1000011
