# EPSApplication SDD

## 1. Overview

`EPSApplication` is the Layer 3 Active component for the EPS subtopology. It monitors battery and power-system health by consuming state data published by `MpptIcManager` (battery/IC state) and `CurrentSensorManager` (per-rail voltage and current) on each rate group tick, and exposes a health/status struct to `SatStateMachine` via a synchronous get port so submode decisions can be made. It also accepts the panel deployment command from ground and forwards it to `DeployPanelsManager`.

Register-access to the BQ25756 is no longer routed through `EPSApplication` — ground commands the IC directly on `MpptIcManager`. `EPSApplication` is a consumer of power-system state, not a command forwarder for it.

Unlike other Layer 3 components, `EPSApplication` has no internal state machine — it operates identically regardless of satellite mode.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-EPS-001 | EPSApplication shall expose a synchronous get port that returns the latest powerState struct when invoked by SatStateMachine. | Inspection |
| HS2-EPS-002 | EPSApplication shall emit a WARNING_HI event when vbatt falls below POWER_THRESHOLD or CRITICAL_THRESHOLD. | Inspection |
| HS2-EPS-003 | EPSApplication shall consume the per-rail measurements published by CurrentSensorManager and incorporate them into the cached powerState. | Inspection |
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
| `batteryStateIn` | Input | Custom struct port | Battery and IC state from MpptIcManager (vbatt, ibatt, vac, iac, charging status, charger/fault flags, MPPT state, temperature) |
| `railStateIn` | Input | Custom struct port | Per-rail bus voltage and current from CurrentSensorManager (12 V, 5 V, 3.3 V) |
| `powerStateGet` | Input | Custom `sync_input` port returning powerState struct | Synchronous get invoked by SatStateMachine; returns the latest assembled powerState (vbatt, ibatt, MPPT status, fault flags, charging status, rail voltages/currents, temperature) |
| `deploy` | Output | `Fw.Signal` | Trigger deployment sequence on DeployPanelsManager |
| `pingIn` / `pingOut` | In/Out | `Svc.Ping` | Health monitoring |
| `logOut` | Output | `Fw.Log` | Event logging |
| `prmGet` | Output | `Fw.PrmGet` | Load power threshold parameters from PrmDb |

`EPSApplication` emits events (e.g. `lowBattery`, `criticalBattery`) but publishes no measurement telemetry — the periodic battery and rail measurement channels are owned and published by `MpptIcManager` and `CurrentSensorManager` respectively.

### 3.3 Commands

| Mnemonic | Args | Description |
|----------|------|-------------|
| `DEPLOY_PANELS` | — | Trigger panel deployment sequence via DeployPanelsManager |

---

## 4. Operational Behavior

`EPSApplication` does not use a hierarchical state machine. Each 1 Hz rate group tick it reads the latest battery state received from `MpptIcManager` and the latest rail state received from `CurrentSensorManager`, runs protection threshold checks, emits any required fault events, and caches the assembled health struct as the current `powerState`. `SatStateMachine` retrieves that struct on demand by invoking the `powerStateGet` synchronous get port. The `DEPLOY_PANELS` handler is a thin forwarder — it validates the request, calls the `deploy` port, and returns a command response.

**Rate group tick flow:**

```
schedIn fires (1 Hz)
  → read latest batteryState from MpptIcManager
  → read latest railState from CurrentSensorManager
  → check vbatt against POWER_THRESHOLD parameter
      if below WARNING threshold → log WARNING_HI (LOW_BATTERY)
      if below CRITICAL threshold → log WARNING_HI (CRITICAL_BATTERY)
  → assemble and cache powerState struct (battery + rail data)
```

**Synchronous get flow (`powerStateGet`):**

```
SatStateMachine invokes powerStateGet (caller's thread, 1 Hz)
  → return cached powerState struct
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

- `EPSApplication` does not autonomously enable or disable MPPT or charging, and it no longer forwards register writes. All BQ25756 register changes are commanded directly on `MpptIcManager` from ground. `EPSApplication` may in the future autonomously adjust charging thresholds based on received battery and rail state — this could require an internal state machine and is TBD pending further design.
- `EPSApplication` forwards `DEPLOY_PANELS` unconditionally to `DeployPanelsManager`. Deployment state tracking and re-attempt behavior are owned by `DeployPanelsManager`'s state machine.
- `batteryStateIn` port type is a custom struct carrying all BQ25756 measurement and status data plus charger/fault flags; final type to be resolved during detailed design. Consider splitting into a measurements port and a flags port if the struct becomes unwieldy.
- `railStateIn` port type is a custom struct carrying the three rails' bus voltage and current, published by `CurrentSensorManager`. `EPSApplication` currently caches it into `powerState`; more complex per-rail fault logic is deferred.
- `powerStateGet` is a synchronous get port — `sync_input` on `EPSApplication`, called from `SatStateMachine`'s caller thread on its 1 Hz tick. The returned struct is the cached value last assembled by the EPS rate group tick; no recomputation occurs inside the get handler. Struct type is custom and must carry at minimum vbatt, ibatt, MPPT status, fault flags, and charging status for `SatStateMachine` submode evaluation; rail voltage/current are included for completeness.
- `BQ25756Reg` enum definitions and register semantics are owned alongside `MpptIcManager` and produced during detailed design; `EPSApplication` no longer references them.
- `MpptIcManager`, `CurrentSensorManager`, `HardwareResetManager`, `WatchdogPinger`, and `DeployPanelsManager` are all instantiated within the EPS subtopology. `EPSApplication` is health-monitored; the hardware managers are not.
- Power threshold parameters (`POWER_THRESHOLD`, `CRITICAL_THRESHOLD`) persisted via `PrmDb`. Specific threshold values and actions TBD pending battery characterization testing.
