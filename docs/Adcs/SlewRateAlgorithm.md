# SlewRateAlgorithm SDD

## 1. Overview

`SlewRateAlgorithm` is a Layer 2.5 **Passive** component within the ADCS subtopology. Given the estimated attitude quaternion, angular rate, target quaternion, and slew rate limit, it computes a slew-rate-limited magnetic moment vector — either capping the commanded rotation to the slew limit or passing the full `QuaternionPdAlgorithm` output through when the commanded rate is already within bounds.

`SlewRateAlgorithm` holds no internal state. `AdcsApplication` calls it synchronously each 10 Hz tick before calling `QuaternionPdAlgorithm` in any pointing mode, ensuring the spacecraft never exceeds the 0.04 °/s slew rate limit commanded per HS2-ADC-003.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-SRA-001 | SlewRateAlgorithm shall compute a slew-rate-limited magnetic moment vector from estimated attitude, angular rate, target quaternion, and slew rate limit | Inspection |
| HS2-SRA-002 | SlewRateAlgorithm shall maintain no internal state between calls | Inspection |

---

## 3. Design

### 3.1 Component Type

Passive component. No thread, no queue, no internal state. Called synchronously by `AdcsApplication`.

### 3.2 Parameters

None.

### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `computeIn` | Input (sync) | `Adcs.SlewRateInputPort` | Takes estimated attitude quaternion, angular rate, target quaternion, and slew rate limit; returns slew-rate-limited magnetic moment vector |

### 3.4 Commands

None.

---

## 4. Notes

- No telemetry, no logging, no health monitoring. Pure stateless computation.
- Called by `AdcsApplication` before `QuaternionPdAlgorithm` in all pointing modes.
- `Adcs.SlewRateInputPort` carries: estimated attitude quaternion, angular rate vector (rad/s), target quaternion, slew rate limit (°/s); return value is magnetic moment vector (A·m²).
- The 0.04 °/s limit from HS2-ADC-003 is passed in by `AdcsApplication` at call time, not stored in this component.
