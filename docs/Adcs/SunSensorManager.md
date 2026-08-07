# SunSensorManager SDD

## 1. Overview

`SunSensorManager` is the Layer 2 hardware manager for the sun sensor array within the ADCS subtopology. It owns all bus communication with the sun sensors, confirming the ADC is alive and reading each sensor channel, computes a body-frame sun vector from the raw light intensity readings, and publishes that vector to `AdcsApplication` each rate-group tick.

`SunSensorManager` is a **Queued** component driven by `RateGroup1` (10 Hz). It has no satellite mode awareness; it runs its startup and read loop unconditionally once initialized. `AdcsApplication` (Layer 3) is responsible for interpreting sun vector data, acting on failures, and, if necessary, commanding a power cut to the sensor array via EPS.

The HS2 sun sensor array consists of six photodiode sensors (one per CubeSat face) read through an **MCP3204/3208 SPI ADC** which is a 12-bit converter with no writable configuration registers of any kind (no reset, no enable/power, no resolution control; see datasheet DS21298E §5.0). Its entire interface is a single full-duplex SPI transaction: clock in a start bit + channel-select bits, clock back that channel's 12-bit conversion result in the same exchange. All bus access goes through the Layer 1 `LinuxSpiDriver`.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-SSM-001 | SunSensorManager shall confirm the sun sensor ADC is present and responding before proceeding to enable | Inspection |
| HS2-SSM-002 | SunSensorManager shall wait for the reset stabilization period before proceeding to enable | Inspection |
| HS2-SSM-003 | SunSensorManager shall enable all sun sensor ADC channels via the SPI chip select sequence | Inspection |
| HS2-SSM-004 | SunSensorManager shall read all active sun sensor channels each rate-group tick while in RUN | Inspection |
| HS2-SSM-005 | SunSensorManager shall compute a body-frame sun unit vector from the raw per-face intensity readings using the calibration model | Inspection |
| HS2-SSM-006 | SunSensorManager shall publish a valid sun vector sample to AdcsApplication after each successful read cycle | Inspection |
| HS2-SSM-007 | SunSensorManager shall log a WARNING_HI event and transition to RESET on any SPI read or write failure | Inspection |
| HS2-SSM-008 | SunSensorManager shall track consecutive failure count and report it as a telemetry channel | Inspection |
| HS2-SSM-009 | SunSensorManager shall apply updated F' parameter values by transitioning from RUN to CONFIGURE without a full reset | Inspection |
| HS2-SSM-010 | SunSensorManager shall not perform any bus operations while in WAIT_RESET | Inspection |
| HS2-SSM-011 | SunSensorManager shall expose the most recently cached sun vector via a synchronous getter port, so a consumer can obtain the current value on demand rather than waiting for the next RUN publish cycle | Inspection |

---

## 3. Design

### 3.1 Component Type

Queued component with internal flat F' state machine (`Fw::Sm`). Has a message queue but no dedicated thread — executes on the `RateGroup1` caller thread each 10 Hz tick.

### 3.2 Parameters

`SunSensorManager` is not satellite-mode-aware and receives no mode commands. Configuration is entirely parameter-driven. Parameters are loaded from `Svc::PrmDb` in the `CONFIGURE` state. A parameter update at runtime sends a `reconfigure` signal, causing `SunSensorManager` to re-enter `CONFIGURE` from `RUN` without a full reset.

| Parameter | Type | Description |
|-----------|------|-------------|
| `CHANNEL_MASK` | `U8` | Bitmask selecting which sensor faces are active |
| `CALIBRATION_SCALE_0`…`CALIBRATION_SCALE_5` | `F32` (×6) | Per-channel sensitivity scale factors for sun vector computation |
| `CALIBRATION_OFFSET_0`…`CALIBRATION_OFFSET_5` | `F32` (×6) | Per-channel dark-current offset corrections |
| `VALIDITY_THRESHOLD` | `F32` | Sun vector magnitude threshold below which the computed vector is marked invalid (eclipse or low signal) |
| `RESET_WAIT_TICKS` | `U32` | Number of rate-group ticks to wait after reset before proceeding |


### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | 10 Hz rate group tick — drives the SM step |
| `getSunVector` | Input (sync) | Custom port (`vectorGetter_p`) | Returns the most recently cached sun vector on demand, independent of the tick cycle |
| `spiWriteRead` | Output | `Drv.SpiWriteRead` | SPI transaction with the ADC, used for the RESET ping and channel reads. One transaction per channel. |
| `sunVectorOut` | Output | `Adcs.SunVectorPort` | Publish body-frame sun unit vector + validity flag to AdcsApplication |
| `prmGet` | Output | `Fw.PrmGet` | Load parameters from PrmDb during CONFIGURE |
| `logOut` | Output | `Fw.Log` | Event logging (state transitions, errors, eclipse detection) |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (SM state, consecutive failure count, raw channel intensities, sun vector) |

### 3.4 Commands

`SunSensorManager` accepts no ground commands. It is not health-monitored. All recovery is handled autonomously by the self-healing SM or escalated via telemetry to `AdcsApplication`.

---

## 4. State Machine

`SunSensorManager` uses a single flat F' state machine following the `fprime-community/fprime-sensors` `ImuManager` reference pattern. All states are peers — no nesting. The MCP3204/3208 has no writable registers, so ENABLE and CONFIGURE do nothing over the bus.

```
RESET
  entry: zero all channel intensity buffers
         reset consecutive-failure counter
         reset wait-tick counter
         ping channel 0 over SPI to confirm the ADC is present and responding
  on ping success → WAIT_RESET
  on ping failure → log WARNING_HI, increment failure count, retry

WAIT_RESET
  on tick: increment wait-tick counter
           if counter >= RESET_WAIT_TICKS → ENABLE
           (no bus operations performed in this state)

ENABLE
  entry: nothing to write.
  on tick → CONFIGURE

CONFIGURE
  entry: load CHANNEL_MASK, CALIBRATION_SCALE_0-5, CALIBRATION_OFFSET_0-5,
         VALIDITY_THRESHOLD from PrmDb
         (nothing written to the ADC; no resolution or channel-enable
          registers exist on this chip)
  on tick → RUN

RUN
  on tick: for each enabled channel (per CHANNEL_MASK):
             one SPI transaction: clock out start+channel-select bits,
             clock in the 12-bit conversion result
           if any spiWriteRead error or lost conversion framing:
             log WARNING_HI (throttled), increment failure count → RESET
           if all reads OK:
             apply CALIBRATION_SCALE/OFFSET to each channel
             compute body-frame sun unit vector from calibrated intensities
             publish sunVectorOut (sun vector, validity flag, timestamp)
             reset consecutive-failure counter

# From any state:
on error signal → RESET        # self-healing fallback
on reconfigure signal → CONFIGURE   # parameter update path from RUN
```

**Sun vector computation:** The body-frame sun unit vector is derived from the six face-mounted sensor readings using a weighted sum model. The validity flag in `sunVectorOut` is set `false` when the vector magnitude falls below `VALIDITY_THRESHOLD` (eclipse or low signal). `AdcsApplication` uses this flag to decide whether to trust the vector for attitude control.

**Error self-healing:** Any SPI error or ADC framing loss from `RESET`, `ENABLE`, `CONFIGURE`, or `RUN` emits a throttled `WARNING_HI` event, increments the `consecutiveFailures` telemetry channel, and re-enters `RESET`. The SM retries the full startup sequence automatically.

**Repeated failure escalation:** `SunSensorManager` does not cut power to itself. If `consecutiveFailures` exceeds a threshold observable by `AdcsApplication` via telemetry, `AdcsApplication` is responsible for commanding an EPS power rail cut. `SunSensorManager` continues attempting self-healing until then.

**Parameter reconfiguration:** When `parameterUpdated()` is called, `SunSensorManager` sends a `reconfigure` signal. If currently in `RUN`, this transitions directly to `CONFIGURE` — no reset required. From any other state the signal is ignored.

Reference: [`fprime-community/fprime-sensors/ImuManager`](https://github.com/fprime-community/fprime-sensors/tree/devel/fprime-sensors/MpuImu/Components/ImuManager), [FPP flat state machines](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc), MCP3204/3208 datasheet DS21298E §3.7 (CS/SHDN behavior) and §5.0 (transaction framing)

---

## 5. Notes

- `SunSensorManager` is instantiated inside the ADCS subtopology. Its `spiWriteRead` port connects to a `LinuxSpiDriver` instance at the top-level topology.
- `sunVectorOut` connects to `AdcsApplication` within the ADCS subtopology.
- `SunSensorManager` is **excluded from health monitoring** (`Svc::Health`). Only `AdcsApplication` is health-checked.
- The `Adcs.SunVectorPort` type (carrying body-frame sun unit vector, validity flag, and timestamp) is defined in the ADCS module and shared with `AdcsApplication`.
- Eclipse detection (vector magnitude below threshold) is surfaced via the validity flag in `sunVectorOut`, not a separate event. `AdcsApplication` determines the satellite is in eclipse; `SunSensorManager` reports only raw and calibrated data.
- Deferred: exact `consecutiveFailures` threshold that triggers `AdcsApplication` to cut sensor power is a system-level parameter to be defined during detailed design.
