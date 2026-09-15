### BDotAlgorithm

#### 1. Overview {.unnumbered .unlisted}

`BDotAlgorithm` is a Layer 2.5 **Passive** component within the ADCS subtopology. It implements the B-dot detumble algorithm: given the current B-field measurement, the previous B-field measurement, and the current angular rate vector (all retrieved from `AttitudeFilter` by `AdcsApplication`), it computes a magnetic moment vector to reduce the spacecraft's angular momentum via magnetorquer actuation.

`BDotAlgorithm` holds no internal state. All state required between calls is stored in `AttitudeFilter`. `AdcsApplication` calls this component synchronously on each 10 Hz tick while in `DETUMBLE` mode.

***

#### 2. Requirements {.unnumbered .unlisted}

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-BDA-001 | BDotAlgorithm shall compute a magnetic moment vector from current B-field, previous B-field, and angular rate | Inspection |
| HS2-BDA-002 | BDotAlgorithm shall maintain no internal state between calls | Inspection |

***

#### 3. Design {.unnumbered .unlisted}

##### 3.1 Component Type {.unnumbered .unlisted}

Passive component. No thread, no queue, no internal state. Called synchronously by `AdcsApplication`.

##### 3.2 Parameters {.unnumbered .unlisted}

None.

##### 3.3 Ports {.unnumbered .unlisted}

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `computeIn` | Input (sync) | `Adcs.BDotInputPort` | Takes current B-field, previous B-field, and angular rate; returns magnetic moment vector |

##### 3.4 Commands {.unnumbered .unlisted}

None.

***

#### 4. Notes {.unnumbered .unlisted}

- No telemetry, no logging, no health monitoring. Pure stateless computation.
- Called by `AdcsApplication` in `DETUMBLE/RUNNING` on each tick after querying `AttitudeFilter` for the required inputs.
- `Adcs.BDotInputPort` carries: current B-field vector (T), previous B-field vector (T), angular rate vector (rad/s); return value is magnetic moment vector (A·m²).
