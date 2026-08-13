# GnssManager SDD

## 1. Overview

`GnssManager` is the Layer 2 hardware manager for the GNSS receiver. It is a SkyFox Labs piNAV-NG with DROP (Dead Reckoning Orbital Propagator). Like `StarTrackerManager` and `EnduroSatManager`, it isn't scoped to a single subtopology and instead it is at the **top-level topology** and shared across `DataCollectionApplication`, `AdcsApplication`, and `SatStateMachine`. It also feeds a PPS (Pulse Per Second) timing signal to Time services, sourced from the receiver's VPP (Valid Position Pulse) output.

`GnssManager` is an **Active** component. The bus is **UART**, 9600 baud, 8N1, LVCMOS levels. All bus access goes through `LinuxUartDriver`, wired to the `ByteStreamDriverClient` port pattern like in `EnduroSatManager`. The receiver's RXD line is unused so there's no command channel for that.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-GNS-001 | GnssManager shall recognize `piNAV-NG` in the receiver's postreset identification banner as confirmation the receiver is alive before entering CONFIGURE | Inspection |
| HS2-GNS-002 | GnssManager shall continuously ingest and parse the receiver's NMEA/piNAV sentence stream as it arrives over UART | Inspection |
| HS2-GNS-003 | GnssManager shall cache the most recent position, velocity, and GPS time | Inspection |
| HS2-GNS-004 | GnssManager shall classify each cached fix as Autonomous, DROP Estimated, or Invalid | Inspection |
| HS2-GNS-005 | GnssManager shall publish the cached fix and its classification to DataCollectionApplication, AdcsApplication, and SatStateMachine on each RateGroup1 tick | Inspection |
| HS2-GNS-006 | GnssManager shall pass the receiver's VPP rising edge through as a PPS timing reference to Time services, tagged with the GPS time reported in the following LSP/LSV sentences. | Inspection |
| HS2-GNS-007 | GnssManager shall report receiver health telemetry each tick | Inspection |
| HS2-GNS-008 | GnssManager shall track and report the elapsed time since the last Autonomous fix, used to detect prolonged reliance on DROP Estimated or Invalid fixes | Inspection |
| HS2-GNS-009 | GnssManager shall log a WARNING_HI event and increment a consecutive failure counter on error | Inspection |
| HS2-GNS-010 | GnssManager shall force the receiver back into Cold Start when the elapsed time since the last Autonomous fix exceeds `DROP_MAX_AGE_TICKS` | Inspection |

---

## 3. Design

### 3.1 Component Type

Active component and an internal flat F' state machine (`Fw::Sm`). This is a change from the other Queued pattern as the piNAV-NG streams NMEA sentences continuously at a fixed 1 Hz over UART, and the `GnssManager` needs to keep draining that stream on its own schedule rather than only when polled. `schedIn` is driven by `RateGroup1` (1 Hz) so the manager parses every incoming 1 Hz sentence group as it arrives, caches the most recent value, and reports the cached value out on each 1 s tick, matching the receiver's own update rate.


### 3.2 Parameters

Almost nothing about the receiver itself is changing at run. The baud rate, update rate (fixed 1 Hz), and the output sentence set are all fixed. 

| Parameter | Type | Description |
|-----------|------|-------------|
| `RESET_WAIT_TICKS` | `U32` | Ticks to wait after powering on before expecting the receiver's boot |
| `BANNER_WAIT_TICKS` | `U32` | Ticks to wait in ENABLE for the boot before giving up and returning to RESET |
| `DROP_MAX_AGE_TICKS` | `U32` | Ticks since the last fix before GnssManager forces a RESET-driven Cold Start |
| `MAX_CONSECUTIVE_UART_ERRORS` | `U32` | Consecutive errors in RUN before forcing a return to RESET |

### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | RateGroup1 (1 Hz) tick. Drives the state machine's ticks and, in RUN, the cached fix publish |
| `drvConnected` | Input | `Drv.ByteStreamReady` | Ready signal for the piNAV-NG's UART connection |
| `drvReceiveIn` | Input | `Drv.ByteStreamData` | Receives raw NMEA/piNAV bytes from `LinuxUartDriver` |
| `drvReceiveReturnOut` | Output | `Fw.BufferSend` | Returns ownership of the buffer arriving on `drvReceiveIn` |
| `resetOut` | Output | `Drv.GpioWrite` | Drives the receiver's /RESET pin (active low), used at startup and optionally to force an early Cold Start if necessary |
| `vppIn` | Input (async) | `Svc.Cycle` | Fires on the VPP pin's rising edge, captured via GPIO interrupt |
| `getGnssFix` | Input (sync) | `Payload.gnssFixGetter_p` | Returns the most recently cached GNSS fix |
| `ppsOut` | Output | `Payload.gnssPps_p` | PPS record derived from the VPP pin, GPS time tagged, to Time services |
| `positionOut` | Output | `Payload.gnssFix_p` | Cached ECEF position, velocity, GPS time, fix classification. To DataCollectionApplication, AdcsApplication, SatStateMachine |
| `prmGet` | Output | `Fw.PrmGet` | Load parameters from PrmDb during CONFIGURE |
| `logOut` | Output | `Fw.Log` | Event logging (state transitions, fix classification changes, UART errors) |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (SM (State Machine) state, failure count, time since last fix, receiver voltages and current and temperature) |

### 3.4 Commands

`GnssManager` accepts no ground commands as the receiver has no command channel. Not health monitored, same as the rest of Layer 2. All recovery is autonomous, self-healing, or escalated via telemetry.

---

## 4. State Machine

`GnssManager` uses the standard Layer 2 flat SM (`RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN`).

```
RESET
  entry: flip /RESET low, hold briefly, flip hi
         clear cached fix, reset failure counter
         reset wait tick counter
  on tick → WAIT_RESET

WAIT_RESET
  on tick: increment wait tick counter
           if counter >= RESET_WAIT_TICKS → ENABLE

ENABLE
  entry: begin listening on UART
  on receiving of "piNAV-NG" anywhere in a line → CONFIGURE
  on not receiving boot banner within BANNER_WAIT_TICKS → log WARNING_HI → RESET

CONFIGURE
  entry: load RESET_WAIT_TICKS, BANNER_WAIT_TICKS, DROP_MAX_AGE_TICKS,
         MAX_CONSECUTIVE_UART_ERRORS from PrmDb
  on tick → RUN

RUN
  (continuously, based off of the UART stream): parse each incoming sentence group
    update cached position/velocity/GPS time
    classify fix as Autonomous / DROP Estimated / Invalid
    feed PPS from vppIn (VPP edge), GPS time tagged, to Time services (event driven,
    not gated by schedIn)
    if Autonomous: reset "time since last Autonomous fix" counter
    if DROP Estimated or Invalid: increment that counter
  on schedIn tick (RateGroup1, 1 Hz): publish cached fix and classification and telemetry
  on error: log WARNING_HI (throttled), increment consecutive failure count
    if consecutive failures >= MAX_CONSECUTIVE_UART_ERRORS → RESET
  on "time since last Autonomous fix" >= DROP_MAX_AGE_TICKS, but only once at least
  one Autonomous fix has ever been achieved:
    log WARNING_HI, assert /RESET to force an early Cold Start → RESET

# From any state:
on reconfigure signal → CONFIGURE   # parameter update path from RUN
```

**Errors:** UART error in `ENABLE` or `RUN` emits a throttled `WARNING_HI`, bumps `consecutiveFailures`, and enters `RESET`.

**Parameter reconfiguration:** `reconfigure` signal drops `RUN` back to `CONFIGURE` without a full reset, just like the other managers.

Reference: [`fprime-community/fprime-sensors` `ImuManager`](https://github.com/fprime-community/fprime-sensors/tree/devel/fprime-sensors/MpuImu/Components/ImuManager) (flat SM reference pattern), [FPP flat state machines](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc)

---

## 5. Notes

- `GnssManager` is created at the top level topology and wires out to three applications: `DataCollectionApplication` (position + time), `AdcsApplication` (position/timing, used in `AntennaPointing`), `SatStateMachine` (orbital state, sun/eclipse). 
- Reset timing TBD: neither datasheet does not specify a minimum /RESET pulse width or reset settle time. It just says "no reset pulse needed," which doesn't help with the `RESET_WAIT_TICKS` number.
- Deferred: exact `consecutiveFailures` threshold and escalation path 
