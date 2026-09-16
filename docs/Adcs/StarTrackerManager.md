# StarTrackerManager

## Overview {.unnumbered .unlisted}

`StarTrackerManager` is the Layer 2 hardware manager for the star tracker, an arcsec NV **Sagitta**. 
The Sagitta determines attitude on its own (camera + onboard lost-in-space and tracking algorithms) and reports it as a mounting calibrated quaternion. `StarTrackerManager`'s job is to boot the device into main firmware, load its configuration, subscribe to its solution telemetry, and cache/republish the calibrated attitude.


## Requirements {.unnumbered .unlisted}

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-STR-001 | StarTrackerManager shall verify communication with the star tracker via the Ping action before considering it reachable | Inspection |
| HS2-STR-002 | StarTrackerManager shall confirm the star tracker has booted before entering CONFIGURE, issuing a Boot action if the star tracker is still in bootloader | Inspection |
| HS2-STR-003 | StarTrackerManager shall upload the Mounting, Camera, Distortion, Algo, and Subscription parameter sets to the star tracker's active configuration during CONFIGURE | Inspection |
| HS2-STR-004 | StarTrackerManager shall configure the star tracker's Algo parameters for autonomous mode so the star tracker selects between LISA and tracking without ground intervention | Inspection |
| HS2-STR-005 | StarTrackerManager shall subscribe to the Solution telemetry field so it is pushed asynchronously rather than polled | Inspection |
| HS2-STR-006 | StarTrackerManager shall synchronize the star tracker's onboard clock via the SetTime action at CONFIGURE entry | Inspection |
| HS2-STR-007 | StarTrackerManager shall cache the most recently received calibrated attitude quaternion, its trustworthiness flag, and its timestamp | Inspection |
| HS2-STR-008 | StarTrackerManager shall publish the cached attitude to AdcsApplication on each new Solution arrival, and reply to AdcsApplication's requests via `getAttitude` | Inspection |
| HS2-STR-009 | StarTrackerManager shall log a WARNING_HI event and increment a consecutive failure counter on any communication or action reply error | Inspection |
| HS2-STR-010 | StarTrackerManager shall return to RESET after `MAX_CONSECUTIVE_COMM_ERRORS` is reached | Inspection |


## Design {.unnumbered .unlisted}

### Component Type {.unnumbered .unlisted}

Active component with an internal flat F' state machine (`Fw::Sm`). All bus access goes through `LinuxUartDriver`, wired to the `ByteStreamDriverClient` port pattern.

The Omnetics connector has no dedicated hardware reset line so `StarTrackerManager` has no `resetOut`/`GpioWrite` port. Power sequencing of the unit itself is owned by EPS. RESET here just means waiting for the star tracker's own power on boot into its bootloader to settle.

### Reference Frame & Mounting {.unnumbered .unlisted}

The Sagitta's local frame has its origin at the center of the camera, X out through the aperture (boresight), Y in the mounting plane, Z completing the right hand set (Manual Figure 2.2, §2.2, p.17). The star tracker itself rotates its raw tracking solution into the spacecraft frame onboard, using a mounting quaternion that must be uploaded during CONFIGURE. The `CalibratedQuaternion_*` fields of the Solution telemetry are already expressed in the spacecraft frame. `StarTrackerManager` reads and republishes those fields directly, with no additional rotation.

### Parameters {.unnumbered .unlisted}


| Parameter | Type | Description |
|-----------|------|-------------|
| `BOOT_TIMEOUT_TICKS` | `U32` | Ticks in ENABLE to reach main firmware (Version telemetry `program` = 2) before giving up and returning to RESET |
| `CONFIG_UPLOAD_TIMEOUT_TICKS` | `U32` | Ticks in CONFIGURE to complete the parameter upload and SetTime sequence before giving up and returning to RESET |
| `SETTIME_RESYNC_TICKS` | `U32` | Ticks between periodic SetTime resyncs in RUN, bounding RTC drift |
| `MAX_CONSECUTIVE_COMM_ERRORS` | `U32` | Consecutive comm/action reply errors in RUN before forcing a return to RESET |

### Ports {.unnumbered .unlisted}

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | `RateGroup3` tick (0.1 Hz). Drives the state machine's ticks and the periodic SetTime resync in RUN |
| `drvConnected` | Input | `Drv.ByteStreamReady` | Ready signal for the star tracker's RS485 connection |
| `drvReceiveIn` | Input | `Drv.ByteStreamData` | Receives raw SLIP framed bytes from `LinuxUartDriver` |
| `drvReceiveReturnOut` | Output | `Fw.BufferSend` | Returns ownership of the buffer arriving on `drvReceiveIn` |
| `drvSendOut` | Output | `Drv.ByteStreamSend` | Sends SLIP framed action and parameter requests to the star tracker |
| `getAttitude` | Input (sync) | `Adcs.AttitudePort` | Returns the most recently cached calibrated attitude, trust flag, and timestamp. Queried by `AdcsApplication`, which uses it to serve `DataCollectionApplication`'s attitude requests |
| `attitudeOut` | Output (async) | `Adcs.AttitudePort` | Pushes the cached calibrated attitude to `AdcsApplication` whenever a new Solution telemetry arrives |
| `cmdIn` | Input | `Fw.Cmd` | Ground commands via CmdDispatcher |
| `cmdResponseOut` | Output | `Fw.CmdResponse` | Command completion status |
| `prmGet` | Output | `Fw.PrmGet` | Load parameters from PrmDb |
| `logOut` | Output | `Fw.Log` | Event logging |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry |

### Commands {.unnumbered .unlisted}

| Command Name | Parameters | Description | Response format | Postcondition |
|--------------|------------|-------------|--------------------------------------|----------------|
| `REBOOT` | — | Issues the Reboot action (ID 7) to power cycle the star tracker's firmware. Request has no fields | Empty response | `StarTrackerManager` reenters `RESET` |
| `ENABLE_PROTECTION` | `wdt: bool`, `current: u8` (0 = no change, 1 = enable, 2 = disable) | Issues the EnableProtection action (ID 8) to toggle the external watchdog timer and power cycling | Empty response | Star tracker's watchdog timer and current protection set as requested |
| `UPLOAD_DISTORTION_CONFIG` | config reference | Reruns OpenFile (ID 33) / WriteFile (ID 35) / CloseFile (ID 34) against `ARC_FILE_CURRENT_CONFIG`. | OpenFile returns the opened file's `size: u32`. WriteFile and CloseFile responses are empty | Star tracker's active `Distortion` (ID 8) and `Camera.focallength` parameters updated to the uploaded values, produced by arcsec's in-orbit calibration procedure |
| `CAPTURE_CALIBRATION_FRAME` | — | Issues a Camera action (ID 15, `actionid=4`, "take a frame"). The frame must already be held in memory (`ImageProcessor.store = 1`) for Download to retrieve it. Then issues repeated Download actions (ID 9), each returning at most 1024 bytes. | Camera response is empty. Each Download response returns `position: u32` (echoed) + `data: u8[1024]` | A calibration frame is captured and downloaded, for ground troubleshooting or in-orbit diagnostics |
| `FORCE_RESET` | — | Forces `StarTrackerManager`'s own state machine back to RESET | N/A | `StarTrackerManager` is in `RESET` |


## State Machine {.unnumbered .unlisted}

```
RESET
  entry: clear cached attitude/trust/timestamp, reset consecutive failure counter,
         reset wait tick counter
         load BOOT_TIMEOUT_TICKS, CONFIG_UPLOAD_TIMEOUT_TICKS, SETTIME_RESYNC_TICKS,
         MAX_CONSECUTIVE_COMM_ERRORS from PrmDb
  on tick → WAIT_RESET

WAIT_RESET
  on tick: increment wait tick counter
  if counter >= BOOT_TIMEOUT_TICKS → ENABLE

ENABLE
  entry: send Ping action (ID 0)
  on valid Ping reply (echoed ID matches) → request Version telemetry (ID 2)
    if Version.program == 1 (bootloader) → issue Boot action (ID 1, region=1)
      → rerequest Version telemetry
    if Version.program == 2 (main firmware) → CONFIGURE
  on no Ping reply, or Version.program still 1 after recheck, within BOOT_TIMEOUT_TICKS → retry
  on BOOT_TIMEOUT_TICKS exceeded without reaching main firmware → log WARNING_HI → RESET

CONFIGURE
  entry: open ARC_FILE_CURRENT_CONFIG in WRITE mode (OpenFile, ID 33)
         write Mounting (ID 6), Camera (ID 9), Distortion (ID 8),
         Algo.mode=12/AUTO with l2t/t2l thresholds (ID 16),
         Subscription.telemetry1=24/Solution (ID 18)
         via WriteFile (ID 35), then CloseFile (ID 34) to activate
  entry: issue SetTime action (ID 14) to sync the RTC
  on config upload confirmed (CloseFile reply status 0) and SetTime reply status 0 → RUN
  on CONFIG_UPLOAD_TIMEOUT_TICKS exceeded, or any action reply status != 0 → log WARNING_HI → RESET

RUN
  (continuously, off the RS485 stream): parse each incoming asynchronous_telemetry_reply
    carrying Solution telemetry (ID 24)
    cache CalibratedQuaternion_*, IsTrustworthy, StableCount, timestamp
    publish cached attitude on attitudeOut whenever a new Solution arrives
  on schedIn tick (RateGroup3, 0.1 Hz): serve getAttitude requests with the cached value;
    publish telemetry
  every SETTIME_RESYNC_TICKS: issue SetTime action (ID 14) to resync the RTC and
    bound drift
  on error (frame checksum failure, UART error, action reply status != 0):
    log WARNING_HI (throttled), increment consecutive failure count
    if consecutive failures >= MAX_CONSECUTIVE_COMM_ERRORS → RESET
```

**Error recovery:** `ENABLE` returns to `RESET` on boot timeout. `CONFIGURE` returns to `RESET` on upload/SetTime failure or timeout. `RUN` returns to `RESET` on consecutive comm errors.

**Errors:** Any action reply status other than 0, or a frame checksum failure, emits a throttled `WARNING_HI`, bumps `consecutiveFailures`, and (in RUN) triggers `RESET` once the threshold is hit.

Reference: [`fprime-community/fprime-sensors` `ImuManager`](https://github.com/fprime-community/fprime-sensors/tree/devel/fprime-sensors/MpuImu/Components/ImuManager) (flat SM reference pattern), [FPP flat state machines](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc)


## Notes {.unnumbered .unlisted}
