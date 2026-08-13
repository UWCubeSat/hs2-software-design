# ScienceInferenceApplication SDD

## 1. Overview

`ScienceInferenceApplication` is the Layer 3 Active component for the ScienceInference subtopology. It processes raw images stored on flash by running them through the HS2 science algorithms (LOST, FOUND). It operates on a scheduled polling cycle — finding unprocessed images, invoking the appropriate algorithm based on experiment metadata, storing results for downlink, compressing flagged images, and marking each image as processed.

`ScienceInferenceApplication` is fully schedule-driven and receives no ground commands directly. It activates and deactivates based on mode commands from `SatStateMachine`.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|-------------|
| HS2-SIA-001 | ScienceInferenceApplication shall poll flash storage for unprocessed images when in ProcessImages mode | Inspection |
| HS2-SIA-002 | ScienceInferenceApplication shall process each unprocessed image with its designated algorithm (LOST, FOUND) based on experiment type metadata | Inspection |
| HS2-SIA-003 | ScienceInferenceApplication shall store algorithm outputs for downlink with matching experiment ID and timestamp | Inspection |
| HS2-SIA-004 | ScienceInferenceApplication shall compress flagged images for downlink | Inspection |
| HS2-SIA-005 | ScienceInferenceApplication shall mark each image as processed upon successful algorithm completion | Inspection |
| HS2-SIA-006 | ScienceInferenceApplication shall assert WARNING_HI if algorithm processing fails for an image and continue to the next | Inspection |
| HS2-SIA-007 | ScienceInferenceApplication shall switch operating mode on command from SatStateMachine | Inspection |
| HS2-SIA-008 | ScienceInferenceApplication shall respond to health ping within the required deadline | Inspection |

---

## 3. Design

### 3.1 Component Type

Active component with internal hierarchical F' state machine (`Fw::Sm`).

### 3.2 Mode Interface

`ScienceInferenceApplication` receives its operating mode from `SatStateMachine` via a typed port:

```fpp
sync input port modeIn: Sat.ScienceInferenceModePort   # carries ScienceInference.Mode
```

Mode enum (owned by this component's module):

```fpp
module ScienceInference {
    enum Mode { Off, ProcessImages }
}
```

If the incoming mode matches the current mode, the handler returns immediately (idempotent).

### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `modeIn` | Input | `Sat.ScienceInferenceModePort` | Mode command from SatStateMachine |
| `schedIn` | Input | `Svc.Sched` | Rate group tick (0.1 Hz) |
| `storageQuery` | Output | `Fw.Cmd` | Query flash for unprocessed images |
| `imageRead` | Output | `Fw.Dp` | Read image data from flash |
| `resultWrite` | Output | `Fw.Dp` | Write algorithm outputs to flash for downlink |
| `imageWrite` | Output | `Fw.Dp` | Write compressed flagged images to flash for downlink |
| `pingIn` / `pingOut` | In/Out | `Svc.Ping` | Health monitoring |
| `logOut` | Output | `Fw.Log` | Event logging |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (images processed, failures) |

### 3.4 Commands

None. `ScienceInferenceApplication` is fully schedule-driven.

---

## 4. State Machine

`ScienceInferenceApplication` uses a hierarchical F' state machine. Mode is the top-level state; operational substates are nested inside each mode. A single `switchMode: ScienceInference.Mode` signal defined at the top level is inherited by all leaf states.

```
OFF

PROCESS_IMAGES
  ├─ QUERYING        (on tick: poll flash for unprocessed images)
  ├─ PROCESSING      (invoke LOST, FOUND per image metadata)
  └─ STORING         (write results + compressed flagged images to flash)

# Inherited by all leaf states:
on switchMode(ScienceInference.Mode.Off)           enter OFF
on switchMode(ScienceInference.Mode.ProcessImages) enter PROCESS_IMAGES
```

**Per-image loop within `PROCESS_IMAGES`:** QUERYING → PROCESSING → STORING cycles for each unprocessed image found. On algorithm failure: log `WARNING_HI`, skip to next image.

Reference: [FPP inherited transitions](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#inherited-transitions), [FPP substates](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#substates)

---

## 5. External Libraries

| Library | Algorithm | Camera | Experiment Type |
|---------|-----------|--------|----------------|
| LOST | Lost-in-space star identification | Camera 1 | L&F |
| FOUND | Follow-up optical navigation | Camera 2 | L&F |


---

## 6. Notes

- `ScienceInferenceApplication` does not delete images. Image purge handled separately.
- Flash polling rate is 0.1 Hz to avoid compute contention with other rate group work.
- Images are written to flash by `DataCollectionApplication` and read here — no direct communication between the two application components.
