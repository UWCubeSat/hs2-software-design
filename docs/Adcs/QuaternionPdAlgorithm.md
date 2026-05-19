# QuaternionPdAlgorithm SDD

## 1. Overview

`QuaternionPdAlgorithm` is a Layer 2.5 **Passive** component within the ADCS subtopology. It implements a quaternion proportional-derivative (PD) attitude control law: given the estimated attitude quaternion, angular rate, and a target quaternion (all retrieved from `AttitudeFilter` by `AdcsApplication`), it computes a magnetic moment vector to drive the spacecraft toward the commanded attitude.

`QuaternionPdAlgorithm` holds no internal state. `AdcsApplication` calls this component synchronously each 10 Hz tick while in any pointing mode (`SUN_POINTING`, `ANTENNA_POINTING`, `EARTH_LIMB_POINTING`, `ATTITUDE_HOLD`), after `SlewRateAlgorithm` has confirmed the commanded slew is within limits.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-QPD-001 | QuaternionPdAlgorithm shall compute a magnetic moment vector from estimated attitude, angular rate, and target quaternion | Inspection |
| HS2-QPD-002 | QuaternionPdAlgorithm shall maintain no internal state between calls | Inspection |

---

## 3. Design

### 3.1 Component Type

Passive component. No thread, no queue, no internal state. Called synchronously by `AdcsApplication`.

### 3.2 Parameters

None.

### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `computeIn` | Input (sync) | `Adcs.QuaternionPdInputPort` | Takes estimated attitude quaternion, angular rate, and target quaternion; returns magnetic moment vector |

### 3.4 Commands

None.

---

## 4. Notes

- No telemetry, no logging, no health monitoring. Pure stateless computation.
- Called by `AdcsApplication` in pointing modes after `SlewRateAlgorithm` confirms the slew rate is within bounds.
- `Adcs.QuaternionPdInputPort` carries: estimated attitude quaternion, angular rate vector (rad/s), target quaternion; return value is magnetic moment vector (A·m²).
- PD gains are implementation details for the component's C++ source; they may be exposed as F' parameters if in-flight tuning is required.
