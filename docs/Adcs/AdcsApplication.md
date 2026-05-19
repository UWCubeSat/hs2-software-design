# AdcsApplication SDD

## 1. Overview

`AdcsApplication` is the Layer 3 Active component for the ADCS subsystem. It owns the attitude control loop and switches operating mode on command from `SatStateMachine`. It dispatches sensor requests and actuator commands to Layer 2 hardware managers (`ImuManager`, `SunSensorManager`, `MagnetorquerManager`) and consumes attitude from `StarTrackerManager` and timing from `GnssManager` (both top-level).

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-ADC-001 | AdcsApplication shall detumble the spacecraft | Inspection |
| HS2-ADC-002 | AdcsApplication shall point the spacecraft at a commanded target quaternion | Inspection |
| HS2-ADC-003 | AdcsApplication shall maintain a slew rate under 0.04 degrees per second upon command | Inspection |
| HS2-ADC-004 | AdcsApplication shall respond to health pings within the required deadline | Inspection |
| HS2-ADC-005 | AdcsApplication shall assert WARNING_HI events followed by a FATAL if hardware is unable to recover from errors | Inspection |

---

## 3. Design

### 3.1 Component Type

Active component with internal hierarchical F' state machine (`Fw::Sm`).

### 3.2 Mode Interface

`AdcsApplication` receives its operating mode from `SatStateMachine` via a typed port:

```fpp
sync input port modeIn: Sat.AdcsModePort   # carries Adcs.Mode
```

Mode enum (owned by this component's module):

```fpp
module Adcs {
    enum Mode { Off, Detumble, SunPointing, AntennaPointing, EarthLimbPointing, AttitudeHold }
}
```

If the incoming mode matches the current mode, the handler returns immediately (idempotent).

### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `modeIn` | Input | `Sat.AdcsModePort` | Mode command from SatStateMachine |
| `schedIn` | Input | `Svc.Sched` | Rate group tick (10 Hz) |
| `imuControl` | Output | `Adcs.ManagerControlPort` | ON/OFF command to ImuManager |
| `sunSensorControl` | Output | `Adcs.ManagerControlPort` | ON/OFF command to SunSensorManager |
| `magnetorquerControl` | Output | `Adcs.ManagerControlPort` | ON/OFF command to MagnetorquerManager |
| `filterAttitudeGet` | Output | `Adcs.AttitudePort` | Query AttitudeFilter for estimated attitude quaternion |
| `filterAngularRateGet` | Output | `Adcs.AngularRatePort` | Query AttitudeFilter for estimated angular rate |
| `filterBFieldGet` | Output | `Adcs.BFieldPort` | Query AttitudeFilter for previous B-field measurement |
| `filterImuTimestampGet` | Output | `Adcs.TimestampPort` | Query AttitudeFilter for IMU data staleness |
| `filterSunTimestampGet` | Output | `Adcs.TimestampPort` | Query AttitudeFilter for sun vector staleness |
| `filterStarTimestampGet` | Output | `Adcs.TimestampPort` | Query AttitudeFilter for star tracker staleness |
| `bDotCompute` | Output | `Adcs.BDotInputPort` | Call BDotAlgorithm |
| `quaternionPdCompute` | Output | `Adcs.QuaternionPdInputPort` | Call QuaternionPdAlgorithm |
| `slewRateCompute` | Output | `Adcs.SlewRateInputPort` | Call SlewRateAlgorithm |
| `momentVectorOut` | Output | `Adcs.MomentVectorPort` | Send computed moment vector to MagnetorquerManager |
| `positionGet` | Output | `Fw.Dp` | Request position/timing from GnssManager |
| `pingIn` / `pingOut` | In/Out | `Svc.Ping` | Health monitoring |
| `logOut` | Output | `Fw.Log` | Event logging |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (attitude estimate, mode, error state) |

### 3.4 Commands

| Mnemonic | Args | Description |
|----------|------|-------------|
| `RESET` | — | Force transition back to current mode's initial substate |

---

## 4. State Machine

`AdcsApplication` uses a hierarchical F' state machine. Mode is the top-level state; operational substates are nested inside each mode. A single `switchMode: Adcs.Mode` signal defined at the top level is inherited by all leaf states, making mode switches valid from any nested substate.

Entry/exit actions follow the FPP Least Common Ancestor rule. Every mode re-entry starts from its `initial` substate — no history.

```
OFF
  entry: send ManagerControl.OFF via imuControl, sunSensorControl, magnetorquerControl
  exit:  send ManagerControl.ON  via imuControl, sunSensorControl, magnetorquerControl

DETUMBLE
  └─ RUNNING
       on tick: query filterAngularRateGet, filterBFieldGet, filterImuTimestampGet
                call bDotCompute(currentBField, previousBField, angularRate) → momentVector
                send momentVectorOut

SUN_POINTING
  ├─ ACQUIRING       (on tick: check filterSunTimestampGet; wait for valid sun vector)
  └─ TRACKING
       on tick: query filterAttitudeGet, filterAngularRateGet, filterSunTimestampGet
                call slewRateCompute(attitude, angularRate, targetQuaternion, 0.04°/s) → momentVector
                call quaternionPdCompute(attitude, angularRate, targetQuaternion) → momentVector
                send momentVectorOut

ANTENNA_POINTING
  ├─ ACQUIRING       (on tick: compute ground station quaternion via GNSS ephemeris)
  └─ TRACKING
       on tick: query filterAttitudeGet, filterAngularRateGet
                call slewRateCompute(attitude, angularRate, targetQuaternion, 0.04°/s) → momentVector
                call quaternionPdCompute(attitude, angularRate, targetQuaternion) → momentVector
                send momentVectorOut

EARTH_LIMB_POINTING
  ├─ ACQUIRING       (on tick: acquire lit Earth limb target + solar optimization)
  └─ TRACKING
       on tick: query filterAttitudeGet, filterAngularRateGet, filterStarTimestampGet
                call slewRateCompute(attitude, angularRate, targetQuaternion, 0.04°/s) → momentVector
                call quaternionPdCompute(attitude, angularRate, targetQuaternion) → momentVector
                send momentVectorOut

ATTITUDE_HOLD
  └─ HOLDING
       on tick: query filterAttitudeGet, filterAngularRateGet
                call slewRateCompute(attitude, angularRate, targetQuaternion, 0.04°/s) → momentVector
                call quaternionPdCompute(attitude, angularRate, targetQuaternion) → momentVector
                send momentVectorOut

# Inherited by all leaf states:
on switchMode(Adcs.Mode.Off)               enter OFF
on switchMode(Adcs.Mode.Detumble)          enter DETUMBLE
on switchMode(Adcs.Mode.SunPointing)       enter SUN_POINTING
on switchMode(Adcs.Mode.AntennaPointing)   enter ANTENNA_POINTING
on switchMode(Adcs.Mode.EarthLimbPointing) enter EARTH_LIMB_POINTING
on switchMode(Adcs.Mode.AttitudeHold)      enter ATTITUDE_HOLD
```

Reference: [FPP inherited transitions](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#inherited-transitions), [FPP substates](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#substates)

---

## 5. Notes

- `StarTrackerManager` and `GnssManager` are top-level components shared with `DataCollectionApplication`; connections are wired at the top-level topology.
- Hardware managers (`ImuManager`, `SunSensorManager`, `MagnetorquerManager`) and Layer 2.5 components (`AttitudeFilter`, `BDotAlgorithm`, `QuaternionPdAlgorithm`, `SlewRateAlgorithm`) are all instantiated inside the ADCS subtopology.
- `AdcsApplication` controls which hardware managers are active via `ManagerControlPort` output ports. Managers that are off do not push data to `AttitudeFilter`.
- `EarthLimbPointing` uses star tracker for precision attitude knowledge — `StarTrackerManager` must be operational and `filterStarTimestampGet` must return a fresh timestamp.
- Mid-operation mode switch behavior (e.g., mode switch arriving mid-maneuver) to be defined during detailed design.
- B-field source for `BDotAlgorithm` is deferred: if the IMU is 9-axis, B-field arrives via `imuDataIn` on `AttitudeFilter`. If not, a separate magnetometer manager is required.
