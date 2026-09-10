# MpptManager SDD

## 1. Overview

`MpptManager` is the Layer 2 Queued hardware manager responsible for all communication with the BQ25756 MPPT and battery charging IC. No other component reads or writes the BQ25756 directly.

On each rate group tick in `RUN` state, `MpptManager` reads all relevant measurement, status, and flag registers over I2C (voltages, currents, charging status, charger/fault flags), assembles the data into a state struct, calls `EPSApplication`'s `batteryStateIn` port, and emits telemetry for the values it just read. If any abnormal charger/fault flag bit is set, it emits a `WARNING_HI` event.

`MpptManager` receives register-access commands directly from ground via the command dispatcher — six commands split by register width and operation. Each command performs the corresponding I2C transaction against the named register and emits a confirmation event.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-MIM-001 | MpptManager shall be the sole flight-software owner of the BQ25756 IC. | Inspection |
| HS2-MIM-002 | MpptManager shall publish the BQ25756 measurement, charging status, and flag state on batteryStateOut each rate group tick. | Inspection |
| HS2-MIM-003 | MpptManager shall perform the commanded register write, set, or clear transaction over I2C on receipt of a register-access command. | Inspection |
| HS2-MIM-004 | MpptManager shall read the charger and fault flag registers each rate group tick and emit a WARNING_HI event when any abnormal status bit is set. | Inspection |
| HS2-MIM-005 | MpptManager shall emit telemetry channels for vbatt, ibatt, vac, iac, and charging status. | Inspection |

---

## 3. Design

### 3.1 Component Type

Queued component with internal flat F' state machine (`Fw::Sm`).

### 3.2 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | Rate group tick; drives the SM step (measurement read cycle in RUN) |
| `busWrite` | Output | `Drv.I2c` | Write BQ25756 registers (reset, configure, register-access commands) |
| `busWriteRead` | Output | `Drv.I2cWriteRead` | Write register pointer then read in one repeated-start transaction; all measurement, status, and flag reads |
| `batteryStateGet` | Input | Custom `sync_input` port returning batteryState struct | Battery and IC state (vbatt, ibatt, vac, iac, vfb, temperature, charging status, charger/fault flags, MPPT state) |
| `cmdIn` | Input | `Fw.Cmd` | Register-access commands via CmdDispatcher |
| `cmdResponseOut` | Output | `Fw.CmdResponse` | Command completion status |
| `logOut` | Output | `Fw.Log` | Event logging |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (vbatt, ibatt, vac, iac, charging status) |

### 3.3 Commands

Each command names its target register through a width-specific enum (`BQ25756Reg8` or `BQ25756Reg16`), so ground can only address a register of the matching width.

| Mnemonic | Args | Description |
|----------|------|-------------|
| `MPPT_WRITE_REG8` | `regAddr: BQ25756Reg8`, `value: U8` | Write to an 8-bit register |
| `MPPT_SET_REG8` | `regAddr: BQ25756Reg8`, `mask: U8` | Read-modify-write: `reg \|= mask` |
| `MPPT_CLEAR_REG8` | `regAddr: BQ25756Reg8`, `mask: U8` | Read-modify-write: `reg &= ~mask` |
| `MPPT_WRITE_REG16` | `regAddr: BQ25756Reg16`, `value: U16` | Write to a 16-bit register |
| `MPPT_SET_REG16` | `regAddr: BQ25756Reg16`, `mask: U16` | Read-modify-write: `reg \|= mask` |
| `MPPT_CLEAR_REG16` | `regAddr: BQ25756Reg16`, `mask: U16` | Read-modify-write: `reg &= ~mask` |

On success each command emits an activity event carrying the operation, register, and the byte/halfword actually written (the post-modify value for set/clear). On I2C failure it emits a `WARNING_HI` write-error event and returns an `EXECUTION_ERROR` command response.

### 3.4 Telemetry

| Channel | Type | Description |
|---------|------|-------------|
| `VBATT_MV` | `U32` | Battery voltage in mV (VBAT_ADC × 2) |
| `IBATT_RAW` | `U16` | Battery current raw ADC (scaling TBD) |
| `VAC_MV` | `U32` | Input voltage in mV (VAC_ADC × 2) |
| `IAC_RAW` | `U16` | Input current raw ADC (scaling TBD) |
| `CHARGING_STATE` | `BQ25756EChargingState` | Charging state from CHARGER_STATUS_1 bits [2:0] |

---

## 4. State Machine

`MpptManager` uses a flat four-state F' state machine: `RESET → WAIT_RESET → CONFIGURE → RUN`. The BQ25756 has no enable step (it does Max Power Point Tracking by default upon being turned on), so the standard `ENABLE` state is omitted.

```
RESET
  on tick: write REG_RST bit in POW_PATH_REV_CONT (0x19 bit 7) to restore register defaults
    if busWrite OK → WAIT_RESET
    if busWrite error → log WARNING_HI, remain in RESET (retry next tick)

WAIT_RESET
  on tick: settling delay for the register reset to complete → CONFIGURE

CONFIGURE
  entry/on tick: write ADC_CONT (0x2B) to enable the ADC in continuous mode
    if all writes OK → RUN
    if any write error → log WARNING_HI → RESET

RUN
  on tick: read measurement registers (VBAT_ADC, IBAT_ADC, VAC_ADC, IAC_ADC, VFB_ADC, TS_ADC)
           read charging status register (CHARGER_STATUS_1)
           read charger/fault flag registers (CHARGER_FLAG_1, CHARGER_FLAG_2, FAULT_STATUS, FAULT_FLAG)
           assemble batteryState struct
           call batteryStateOut port
           emit telemetry channels
           if any abnormal charger/fault flag bit is set → emit WARNING_HI flag event (throttled)
           if any busWriteRead error → log WARNING_HI (throttled) → RESET
  on register-access command (MPPT_WRITE_REG8 / MPPT_SET_REG8 / MPPT_CLEAR_REG8 /
                              MPPT_WRITE_REG16 / MPPT_SET_REG16 / MPPT_CLEAR_REG16):
           perform the I2C write (read-modify-write for set/clear)
           emit register-written event, or WARNING_HI write-error event on I2C failure
```

Fault handling is done by the per-tick flag read: `CHARGER_FLAG_1/2` and `FAULT_FLAG` are cleared-on-read, so each tick captures the events that occurred since the previous tick. An abnormal flag emits a warning event but does not change state; the IC continues running. Only I2C bus errors self-heal by returning to `RESET`.

**Command behavior outside RUN:** register-access commands received while in `RESET`, `WAIT_RESET`, or `CONFIGURE` are queued and execute once the component processes them — ordering against the reset/configure writes is TBD during detailed design.

Reference: [FPP flat state machines](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc)

---

## 5. Notes

- `batteryStateOut` port type is a custom struct carrying all BQ25756 measurement and status data plus the charger/fault flags. Consider splitting into a measurements port and a flags port if the struct becomes unwieldy; see `EPSApplication` notes for the matching `batteryStateIn` discussion.
- `MpptManager` owns and publishes the periodic measurement telemetry (`VBATT_MV`, `IBATT_RAW`, `VAC_MV`, `IAC_RAW`, `CHARGING_STATE`). It still forwards the same values to `EPSApplication` on `batteryStateOut`; `EPSApplication` consumes them for `powerState` and may use them for more complex logic in the future.
- The ADC must be enabled before measurement reads return real values; enabling it in `CONFIGURE` guarantees `RUN` reads are valid without an operator step.
- The set of "abnormal" _FLAG bits that warrant a warning event is TBD pending a detailed pass over the BQ25756 datasheet fault/flag register definitions.
- `BQ25756Reg8` / `BQ25756Reg16` enums enumerate all addressable registers by name and width; the `BQ25756RegOp` enum names the write/set/clear operation carried in the confirmation events.
