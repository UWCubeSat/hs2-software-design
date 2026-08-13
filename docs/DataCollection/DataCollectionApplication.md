# DataCollectionApplication SDD

## 1. Overview

SCOPE

`DataCollectionApplication` is the Layer 3 Active component for the DataCollection subtopology. It executes data collection experiments on command from `SatStateMachine` — powering on cameras, configuring them with experiment parameters from `PrmDb`, simultaneously acquiring images and navigation data, storing results to flash, and powering off all hardware upon completion.

It coordinates `Camera1Manager` and `Camera2Manager` (within its subtopology) and consumes attitude from `StarTrackerManager` and position from `GnssManager` (both top-level).

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|-------------|
| HS2-DCA-001 | DataCollectionApplication shall capture data as defined by experiment parameters from PrmDb | Inspection |
| HS2-DCA-002 | DataCollectionApplication shall power on all cameras prior to an experiment | Inspection |
| HS2-DCA-003 | DataCollectionApplication shall perform a health check on all cameras before running an experiment and assert WARNING_HI if any check fails | Inspection |
| HS2-DCA-004 | DataCollectionApplication shall simultaneously acquire images, attitude, and position within 10ms | Inspection |
| HS2-DCA-005 | DataCollectionApplication shall store all captured images to flash with experiment ID and timestamp | Inspection |
| HS2-DCA-006 | DataCollectionApplication shall not perform an experiment if flash data storage is full | Inspection |
| HS2-DCA-007 | DataCollectionApplication shall power off all cameras upon experiment completion or error | Inspection |
| HS2-DCA-008 | DataCollectionApplication shall report experiment success or failure upon completion | Inspection |
| HS2-DCA-009 | DataCollectionApplication shall switch operating mode on command from SatStateMachine | Inspection |
| HS2-DCA-010 | DataCollectionApplication shall respond to health ping within the required deadline | Inspection |

---

## 3. Design

### 3.1 Component Type

Active component with internal hierarchical F' state machine (`Fw::Sm`).

### 3.2 Mode Interface

`DataCollectionApplication` receives its operating mode from `SatStateMachine` via a typed port:

```fpp
sync input port modeIn: Sat.DataColModePort   # carries DataCollection.Mode
```

Mode enum (owned by this component's module):

```fpp
module DataCollection {
    enum Mode { Off, HealthCheck, RunExperiment }
}
```

If the incoming mode matches the current mode, the handler returns immediately (idempotent).

### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `modeIn` | Input | `Sat.DataColModePort` | Mode command from SatStateMachine |
| `schedIn` | Input | `Svc.Sched` | Rate group tick |
| `cameraPowerOn[2]` | Output | `Fw.Cmd` | Power on Camera1/Camera2Manager |
| `cameraPowerOff[2]` | Output | `Fw.Cmd` | Power off Camera1/Camera2Manager |
| `cameraCmd[2]` | Output | `Fw.Cmd` | Configure and capture commands to CameraManagers |
| `attitudeGet` | Output | `Fw.Dp` | Request attitude snapshot from StarTrackerManager |
| `positionGet` | Output | `Fw.Dp` | Request position snapshot from GnssManager |
| `storageQuery` | Output | `Fw.Cmd` | Query flash storage availability |
| `imageWrite` | Output | `Fw.Dp` | Write captured image and metadata to flash |
| `pingIn` / `pingOut` | In/Out | `Svc.Ping` | Health monitoring |
| `prmGet` | Output | `Fw.PrmGet` | Load experiment parameters from PrmDb |
| `logOut` | Output | `Fw.Log` | Event logging |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (experiment state, storage status) |

### 3.4 Commands

| Mnemonic | Args | Description |
|----------|------|-------------|
| `RUN_EXPERIMENT` | `expId: U8` | Manually trigger a data collection run (ground override) |

---

## 4. State Machine

`DataCollectionApplication` uses a hierarchical F' state machine. Mode is the top-level state; operational substates are nested inside each mode. A single `switchMode: DataCollection.Mode` signal defined at the top level is inherited by all leaf states, making mode switches valid from any nested substate.

Entry/exit actions follow the FPP Least Common Ancestor rule. Every mode re-entry starts from its `initial` substate.

```
OFF

HEALTH_CHECK
  └─ CHECKING        (power on cameras → verify → power off → report)

RUN_EXPERIMENT
  ├─ POWERING_ON     (power on Camera1Manager, Camera2Manager)
  ├─ CAPTURING       (simultaneous: Camera1 image, Camera2 image,
  │                   StarTracker attitude, GNSS position — within 10ms window)
  ├─ STORING         (write images + metadata to flash with experiment ID + timestamp)
  └─ POWERING_OFF    (power off all cameras; on complete → enter OFF)

# Inherited by all leaf states:
on switchMode(DataCollection.Mode.Off)           enter OFF
on switchMode(DataCollection.Mode.HealthCheck)   enter HEALTH_CHECK
on switchMode(DataCollection.Mode.RunExperiment) enter RUN_EXPERIMENT
```

**Storage check:** Performed at entry to `RUN_EXPERIMENT` before `POWERING_ON`. If flash is full, log `WARNING_HI` and transition directly to `OFF`.

Reference: [FPP inherited transitions](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#inherited-transitions), [FPP substates](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#substates)

---

## 5. Notes

- `StarTrackerManager` and `GnssManager` are top-level components; connections wired at top-level topology.
- Experiment type (L&F vs Calibration) is encoded in experiment parameters from `PrmDb`; `DataCollectionApplication` passes parameters to `CameraManagers` without distinguishing experiment type.
- Images stored to flash are subsequently processed by `ScienceInferenceApplication`.
- Mid-operation mode switch behavior (e.g., mode switch arriving during `CAPTURING`) to be defined during detailed design.
