# CommsApplication SDD

## 1. Overview

`CommsApplication` is the Layer 3 Active component for the Comms subtopology. It manages the EnduroSat S-band radio operating mode — switching between `BEACON` to transmit real-time SOH telemetry at 1Hz, `STANDARD_DOWNLINK` mode to transmit all real-time telemetry including payload experiment data, and `STORED_PLAYBACK` mode to downlink stored and real-tim SOH telemetry concurrently.
---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-COM-001 | CommsApplication shall activate `BEACON` downlink mode when commanded by `SatStateMachine` | Inspection |
| HS2-COM-002 | CommsApplication shall activate `STANDARD_DOWNLINK` downlink mode when commanded by `SatSatMachine` | Inspection |
| HS2-COM-003 | CommsApplication shall activate `STORED_PLAYBACK` downlink mode only when commanded | Inspection |
| HS2-COM-004 | CommsApplication shall respond to health ping within the required deadline | Inspection |
| HS2-COM-005 | CommsApplication shall enable the transmission of stored telemetry along with real-time telemetry when `STORED_PLAYBACK` downlink mode is activated | Inspection |
| HS2-COM-006 | CommsApplication shall enable the transmission of SOH telemetry only at a 1Hz rate when `BEACON` downlink mode is activated | Inspection |
| HS2-COM-006 | CommsApplication shall enable the transmission of all real-time telemetry when `STANDARD_DOWNLINK` downlink mode is activated | Inspection |

---

## 3. Design

### 3.1 Component Type

Active component with internal hierarchical F' state machine (`Fw::Sm`).

### 3.2 Mode Interface

`CommsApplication` receives its operating mode from `SatStateMachine` via a typed port:

```fpp
sync input port modeIn: Sat.CommsModePort   # carries Comms.Mode
```

Mode enum (owned by this component's module):

```fpp
module Comms {
    enum Mode { BEACON, STANDARD_DOWNLINK, STORED_PLAYBACK }
}
```

If the incoming mode matches the current mode, the handler returns immediately (idempotent).

#### 3.2.1 `Beacon` Mode

During this mode, only SOH telemetry shall be transmitted. This mode is intended for the satellite to establish a connection with the ground station, particularly when detumbling or when outside of a communication window. In `Beacon` mode, the satellite shall only transmit a single SOH telemetry packet at 1Hz to ensure minimum power draw.

This mode will set all packets, except SOH, in the `TLMPacketizer` to have a `RateLogic` of `SILENCED`. 

#### 3.2.2 `STANDARD_DOWNLINK` mode

During this mode, all real-time telemetry shall be downlinked to the ground station at 1Hz.

This mode will set all packets, except SOH, in the `TLMPacketizer` to have a `RateLogic` of `ON_CHANGE_MIN`. 

#### 3.2.3 `STORED_PLAYBACK` mode

During this mode, all stored telemetry, which includes payload experiment data, will be downlinked to the ground station. The priority in this mode is to downlink images and payload data captured during experiments. Stored telemetry will be downlinked at a rate of 5 Hz while real-time SOH telemetry will be downlinked at a rate of 1 Hz.

This mode will set SOH telemetry in the `TLMPacketizer` to have a `RateLogic` of `EVERY_MAX` to ensure stored telemetry is given priority. The `StorageManager` will then forward all stored telemetry in its internal queue to the `ComQueue` for downlink.

### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `modeIn` | Input | `Sat.CommsModePort` | Mode command from SatStateMachine |
| `schedIn` | Input | `Svc.Sched` | Rate group tick |
| `downlinkMode` | Output | `Fw.Cmd` | Configure EnduroSatManager operating mode |
| `pingIn` | `pingOut` | In/Out | `Svc.Ping` | Health monitoring |
| `logOut` | Output | `Fw.Log` | Event logging |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (current mode, link state) |
---

## 4. State Machine

`CommsApplication` uses a hierarchical F' state machine. Mode is the top-level state. A single `switchMode: Comms.Mode` signal defined at the top level is inherited by all leaf states.

```
OMNI_ONLY
  └─ ACTIVE          (omni link maintained; small telemetry packets only)

HIGH_GAIN_DOWNLINK
  ├─ CONFIGURING     (configure EnduroSatManager for high-rate downlink)
  └─ DOWNLINKING     (high-rate downlink active; requires AntennaPointing from AdcsApplication)

# Inherited by all leaf states:
on switchMode(Comms.Mode.OmniOnly)          enter OMNI_ONLY
on switchMode(Comms.Mode.HighGainDownlink)  enter HIGH_GAIN_DOWNLINK
```

Reference: [FPP inherited transitions](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#inherited-transitions), [FPP substates](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#substates)

---

## 5. Notes

- `HIGH_GAIN_DOWNLINK` requires `AdcsApplication` to be in `AntennaPointing` mode. `SatStateMachine` is responsible for commanding both simultaneously via the translation table — `CommsApplication` does not check ADCS state directly.
- `EnduroSatManager` is instantiated at the top-level topology (shared with `ComCcsds` subtopology); `CommsApplication` connects to it via the top-level topology wiring.
- Detailed high-gain link configuration and `EnduroSatManager` interface to be defined during detailed design.
- The `TLMPacketizer` component gives the `CommsApplication` capabilities to configure the rate at which certain packets are sent for the various operating modes.
