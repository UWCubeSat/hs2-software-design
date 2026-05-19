# AttitudeFilter SDD

## 1. Overview

`AttitudeFilter` is a Layer 2.5 **Queued** component within the ADCS subtopology. It fuses IMU angular rate/acceleration, sun vector, and star tracker attitude quaternion into a satellite state estimate. IMU data is the primary always-on source; sun vector and star tracker attitude are optional, arriving only when those managers are active and healthy.

Each source's last received timestamp is tracked independently. `AdcsApplication` queries individual getter ports each tick to retrieve state values and assess data freshness before passing them to algorithm components. `AttitudeFilter` does not reject stale data — it stores the most recent valid reading from each source and reports when it was received.

`AttitudeFilter` also stores the previous B-field measurement for use by `BDotAlgorithm`. The B-field source is TBD pending confirmation of whether the IMU is a 9-axis device (with integrated magnetometer) or a separate magnetometer manager is required.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-ATF-001 | AttitudeFilter shall update its estimated angular rate on each received IMU sample | Inspection |
| HS2-ATF-002 | AttitudeFilter shall update its estimated attitude quaternion on each received star tracker attitude sample | Inspection |
| HS2-ATF-003 | AttitudeFilter shall update its sun vector estimate on each received sun vector sample | Inspection |
| HS2-ATF-004 | AttitudeFilter shall record the timestamp of each received measurement per source | Inspection |
| HS2-ATF-005 | AttitudeFilter shall return the most recently stored value for each getter port query | Inspection |
| HS2-ATF-006 | AttitudeFilter shall store the previous B-field measurement for use by BDotAlgorithm | Inspection |

---

## 3. Design

### 3.1 Component Type

Queued component. Manager data arrives asynchronously via async input ports on the rate group tick. `AdcsApplication` queries state synchronously via individual getter ports. The queue serializes writes from the rate group thread against reads from `AdcsApplication`'s Active component thread.

### 3.2 Parameters

None.

### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `imuDataIn` | Input (async) | `Adcs.ImuDataPort` | IMU angular rate + acceleration sample from ImuManager |
| `sunVectorIn` | Input (async) | `Adcs.SunVectorPort` | Body-frame sun vector from SunSensorManager (when active) |
| `attitudeIn` | Input (async) | `Adcs.AttitudePort` | Absolute attitude quaternion from StarTrackerManager (when active) |
| `estimatedAttitudeGet` | Input (sync) | `Adcs.AttitudePort` | Returns current estimated attitude quaternion |
| `estimatedAngularRateGet` | Input (sync) | `Adcs.AngularRatePort` | Returns current estimated angular rate vector |
| `previousBFieldGet` | Input (sync) | `Adcs.BFieldPort` | Returns previous B-field measurement (for BDot derivative) |
| `imuTimestampGet` | Input (sync) | `Adcs.TimestampPort` | Returns timestamp of last received IMU sample |
| `sunVectorTimestampGet` | Input (sync) | `Adcs.TimestampPort` | Returns timestamp of last received sun vector |
| `starTrackerTimestampGet` | Input (sync) | `Adcs.TimestampPort` | Returns timestamp of last received star tracker attitude |
| `logOut` | Output | `Fw.Log` | Event logging |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (estimated attitude, angular rate, per-source last timestamps) |

### 3.4 Commands

None.

---

## 4. Notes

- `AttitudeFilter` does not implement a formal sensor fusion algorithm (e.g. EKF). It stores the most recent valid reading from each source and makes it available via getter ports. Any fusion logic lives in the algorithm components or `AdcsApplication`.
- Staleness is determined by `AdcsApplication`: it reads the per-source timestamp getters and decides whether a source is fresh enough for the active algorithm. `AttitudeFilter` makes no staleness judgement itself.
- `AttitudeFilter` is instantiated inside the ADCS subtopology.
- B-field source is deferred: if the IMU is 9-axis, `imuDataIn` carries magnetic field data alongside angular rate and acceleration. If not, a separate magnetometer manager port (`magFieldIn`) will be added.
- `StarTrackerManager` is a top-level shared component; its `attitudeOut` port connects to `AttitudeFilter` via the ADCS subtopology boundary.
