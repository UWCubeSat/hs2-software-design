# EPSApplication SDD

## 1. Overview

`EPSApplication` is the Layer 3 Active component for the EPS subtopology. It accepts commands from both ground and `SatStateMachine` to configure the power system (charging, MPPT, JEITA thresholds) and forwards them to `MpptIcManager` via a single register-write port, and accepts the panel deployment command and forwards it to `DeployPanelsManager`. It monitors battery and power system health by consuming state data published by `MpptIcManager` on each rate group tick and exposing a health/status struct to `SatStateMachine` via a synchronous get port so submode decisions can be made.

Unlike other Layer 3 components, `EPSApplication` has no internal state machine — it operates identically regardless of satellite mode.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-EPS-001 | EPSApplication shall expose a synchronous get port that returns the latest powerState struct when invoked by SatStateMachine. | Inspection |
| HS2-EPS-002 | EPSApplication shall emit a WARNING_HI event when vbatt falls below POWER_THRESHOLD or CRITICAL_THRESHOLD. | Inspection |
| HS2-EPS-003 | EPSApplication shall forward SET_IC_REGISTER commands to MpptIcManager without refusal logic. | Inspection |
| HS2-EPS-004 | EPSApplication shall forward DEPLOY_PANELS commands to DeployPanelsManager without refusal logic. | Inspection |
| HS2-EPS-005 | EPSApplication shall operate continuously regardless of satellite mode. | Inspection |
| HS2-EPS-006 | EPSApplication shall respond to health ping within the required deadline. | Inspection |

---

## 3. Design

### 3.1 Component Type

Active component. No hierarchical state machine — `EPSApplication` operates continuously and is fully driven by rate group ticks and incoming commands with no satellite-mode-driven state transitions.

### 3.2 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | Rate group tick (1 Hz) |
| `cmdIn` | Input | `Fw.Cmd` | Ground commands via CmdDispatcher |
| `cmdResponseOut` | Output | `Fw.CmdResponse` | Command completion status |
| `batteryStateIn` | Input | Custom struct port | Battery and IC state from MpptIcManager (vbatt, ibatt, vac, iac, charging status, fault flags, MPPT state, temperature) — type TBD |
| `powerStateGet` | Input | Custom `sync_input` port returning powerState struct | Synchronous get invoked by SatStateMachine; returns the latest assembled powerState (vbatt, ibatt, MPPT status, fault flags, charging status, temperature) — struct type TBD |
| `setRegister` | Output | Custom port (`BQ25756Reg`, `U32`) | Register write forwarded to MpptIcManager; register identified by `BQ25756Reg` enum |
| `deploy` | Output | `Fw.Signal` | Trigger deployment sequence on DeployPanelsManager |
| `pingIn` / `pingOut` | In/Out | `Svc.Ping` | Health monitoring |
| `logOut` | Output | `Fw.Log` | Event logging |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (vbatt, ibatt, charging status, fault flags) |
| `prmGet` | Output | `Fw.PrmGet` | Load power threshold parameters from PrmDb |

### 3.3 Commands

| Mnemonic | Args | Description |
|----------|------|-------------|
| `SET_IC_REGISTER` | `reg: BQ25756Reg`, `value: U32` | Write a BQ25756 register via MpptIcManager. `reg` is a named enum (e.g., `BQ25756Reg::CHG_ENABLE`, `BQ25756Reg::JEITA_T1`, `BQ25756Reg::ICHG_LIMIT`). |
| `DEPLOY_PANELS` | — | Trigger panel deployment sequence via DeployPanelsManager |

---

## 4. Operational Behavior

`EPSApplication` does not use a hierarchical state machine. Each 1 Hz rate group tick it reads the latest battery state received from `MpptIcManager`, runs protection threshold checks, emits any required fault events, and caches the assembled health struct as the current `powerState`. `SatStateMachine` retrieves that struct on demand by invoking the `powerStateGet` synchronous get port. All command handlers are thin forwarders — they validate the request, call the appropriate output port, and return a command response.

**Rate group tick flow:**

```
schedIn fires (1 Hz)
  → read latest batteryState from MpptIcManager
  → check vbatt against POWER_THRESHOLD parameter
      if below WARNING threshold → log WARNING_HI (LOW_BATTERY)
      if below CRITICAL threshold → log WARNING_HI (CRITICAL_BATTERY)
  → assemble and cache powerState struct
  → emit telemetry channels
```

**Synchronous get flow (`powerStateGet`):**

```
SatStateMachine invokes powerStateGet (caller's thread, 1 Hz)
  → return cached powerState struct
```

**Command flow (`SET_IC_REGISTER`):**

```
cmdIn SET_IC_REGISTER(reg, value)
  → validate arguments
  → call setRegister(reg, value) to MpptIcManager
  → emit activity event
  → send cmdResponse (OK or EXECUTION_ERROR)
```

**Command flow (`DEPLOY_PANELS`):**

```
cmdIn DEPLOY_PANELS
  → call deploy port to DeployPanelsManager
  → emit activity event
  → send cmdResponse OK
```

---

## 5. Notes

- `EPSApplication` does not autonomously enable or disable MPPT or charging. All such changes require an explicit `SET_IC_REGISTER` command from ground or `SatStateMachine`. However, it may in the future autonomously adjust charging thresholds (e.g. JEITA limits) based on received battery state data — this could require an internal state machine and is TBD pending further design.
- There is no command refusal logic for `SET_IC_REGISTER` — `EPSApplication` forwards all register writes to `MpptIcManager` unconditionally without checking current IC state.
- `EPSApplication` forwards `DEPLOY_PANELS` unconditionally to `DeployPanelsManager`. Deployment state tracking and re-attempt behavior are owned by `DeployPanelsManager`'s state machine.
- `batteryStateIn` port type is a custom struct carrying all BQ25756 measurement and status data; final type to be resolved during detailed design. Consider splitting into a measurements port and a flags port if the struct becomes unwieldy.
- `powerStateGet` is a synchronous get port — `sync_input` on `EPSApplication`, called from `SatStateMachine`'s caller thread on its 1 Hz tick. The returned struct is the cached value last assembled by the EPS rate group tick; no recomputation occurs inside the get handler. Struct type is custom and must carry at minimum vbatt, ibatt, MPPT status, fault flags, and charging status for `SatStateMachine` submode evaluation.
- `BQ25756Reg` enum definition (register addresses and semantics) to be produced during detailed design alongside hardware bring-up. Readable names (e.g., `CHG_ENABLE`, `JEITA_T1`, `ICHG_LIMIT`, `VRECHG`) replace raw hex values.
- `MpptIcManager`, `HardwareResetManager`, `WatchdogPinger`, and `DeployPanelsManager` are all instantiated within the EPS subtopology. `EPSApplication` is health-monitored; the hardware managers are not.
- Power threshold parameters (`POWER_THRESHOLD`, `CRITICAL_THRESHOLD`) persisted via `PrmDb`. Specific threshold values and actions TBD pending battery characterization testing.
