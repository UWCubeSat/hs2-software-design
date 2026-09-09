# DataCollection::CameraManager

`CameraManager` is a Layer 2 active component driving one physical camera. It is meant to be
instantiated once per camera (e.g. `lostCameraManager`/`foundCameraManager` in
`DataCollectionApplication`'s subtopology, two instances of this same component distinguished only
by instance name/config - see that component's SDD, Design/Open Items) - so its ports never carry a
camera identifier; the instance itself *is* the identity.

It exposes two ways in: an async `captureImageIn` port that requests a single image capture after
roughly a requested `delay`, and a sync `healthCheckIn` port that answers immediately with the
camera's current status. Triggering itself is hardware, not software: `captureImageIn_handler`
pulses a GPIO output port (`cameraTriggerOut`) wired to the camera's digital trigger input, rather
than issuing a software-trigger command over USB - see Timing. The camera hardware itself is a
XIMEA xiC USB3 camera on a Sony Pregius CMOS sensor (see
`docs/xiC-USB3-Sony-CMOS-Pregius-cameras_Technical-manual-DWL_manual.pdf`, the vendor's xiAPI
technical manual).

## Component Type

**Active.** `captureImageIn` needs to sleep out part of its requested `delay` (see Timing below)
without blocking anything else talking to this component. An active component owns its own thread
for its async input ports, so that sleep only holds up further `captureImageIn` calls (queued
behind it, same as any active component's async port) - it does not block `healthCheckIn`, which is
a `sync input port` and therefore always runs directly on its caller's thread, unaffected by
whatever the async side is doing. A `queued` component was considered (matching an earlier draft's
doc comment on this file) but rejected: queued components have a message queue but no thread of
their own, so draining `captureImageIn` would happen on whichever other component's thread calls in
to dispatch the queue - sleeping there would block that caller, not just this component.

## Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `captureImageIn` | Input, async | `DataCollection.CameraCapture(imageType: Types.ImageType, delay: F32, outputPath: string)` | Requests one capture of `imageType`, no sooner than `delay` seconds after the call, writing the retrieved frame to `outputPath`. No return value - see Timing and Outcome Reporting below. |
| `healthCheckIn` | Input, sync | `DataCollection.CameraCheckup() -> Types.CameraStatus` | Per-camera health check; answers immediately. |
| `cameraTriggerOut` | Output | `Drv.GpioWrite($state: Fw.Logic) -> Drv.GpioStatus` | Drives the GPIO pin wired to this camera's hardware trigger input; pulsed high then low by `captureImageIn_handler` - see Timing. Safe to leave unconnected (no-op) until a GPIO driver instance exists in topology. |
| `timeCaller`, `prmGetOut`/`prmSetOut` | Output | standard AC ports | Time, parameter get/set - required for events/telemetry/params. |
| (command recv/response) | In/Out | `Fw.Command` | Required by the autocoder because `CAMERA_TRIGGER_DELAY_S` is a parameter (`SET_CMD`/`SAVE_CMD` are dispatched as commands even though this component defines no commands of its own) - see Parameters. |

`DataCollection.CameraCapture` and `DataCollection.CameraCheckup` (`FlightComputer/Ports/
DataCollection/DataCollectionPorts.fpp`) previously carried a `camera: Types.Camera` argument, back
when they were stub ports with no real implementation to size against. That argument is dropped:
now that `CameraManager` is real and one instance is one camera, a `camera` field on every call
would just have to agree with the instance it's already being called on - so instead of a
runtime-checkable-but-always-redundant field, the instance itself carries that identity. This is a
breaking change to those two port types (`CameraPower`, the third sibling port, is unaffected here
and keeps its original shape); anything that connects to a `CameraManager` instance keys off
*which* instance, not a port argument.

`CameraCapture` additionally gained an `outputPath: string size 256` argument once real image
retrieval landed: `xiGetImage` hands back a frame buffer that has to be written *somewhere*, and
`CameraManager` deliberately knows nothing about the image-partition naming convention. The caller
(`DataCollectionApplication`, which already owns that convention) resolves the full path and passes
it down, mirroring how `Science::ScienceApplication` hands explicit paths to its algorithm
executables - see `docs/xiapi-integration-draft.md`, S4.1.

## Timing

**The problem:** ground/`DataCollectionApplication` wants a picture taken `delay` seconds from now,
but the camera itself is not instantaneous - triggering it costs some latency of its own between
"triggered" and "exposure actually starts." If `CameraManager` slept the full `delay` and then
triggered, the exposure would start at `delay + (camera's own latency)`, not `delay`.

**The fix:** `captureImageIn_handler` sleeps `delay - CAMERA_TRIGGER_DELAY_S`, not `delay`, where
`CAMERA_TRIGGER_DELAY_S` is a parameter holding the camera's own trigger-to-exposure latency. That
way the trigger call happens `CAMERA_TRIGGER_DELAY_S` seconds early, and (trigger call) + (camera's
own latency) lands close to the original `delay`.

**Why the trigger is hardware, not software:** the manual, S3.2.2 ("Trigger controlled
acquisition/exposure"), draws exactly this distinction:

> Software trigger: The trigger signal can be sent to the sensor using a software command. In this
> case, common system related latencies and jitter apply.
>
> Hardware trigger: ... usually used to reduce latencies and jitter in applications that require the
> most accurate timing.

A software trigger (a command sent to the camera over its USB3 control channel) has a latency the
manual explicitly declines to pin down - it depends on the host system, not just the sensor. A
hardware trigger (a GPIO edge into the camera's own digital trigger input, S2.10.1) is the
manual's own recommendation for applications - like this one - that need `delay` to actually mean
something. So `captureImageIn_handler` drives `cameraTriggerOut` (`Drv.GpioWrite`) directly rather
than issuing a software-trigger call.

**Where `CAMERA_TRIGGER_DELAY_S` comes from:** even with a hardware trigger, there's still a gap
between the GPIO edge arriving and the exposure actually starting - the manual gives input-edge
propagation delays for the digital input (low microseconds, e.g. Table 134/137/138) but that's only
part of the sensor's own internal trigger-to-exposure path, and depends on which physical input type
and cabling is actually used. So `CAMERA_TRIGGER_DELAY_S` is a runtime parameter (ground-settable
via the standard `SET_CMD`, defaulted to `0.0`), meant to be measured/calibrated against the real
flight hardware and cabling, not read off a datasheet. Defaulting it to `0.0` means an uncalibrated
`CameraManager` degrades to "sleep the full `delay`, then trigger" - correct-but-late by the
camera's own latency - rather than silently guessing at a number nothing has actually measured.

**The trigger pulse itself:** `captureImageIn_handler` drives `cameraTriggerOut` high, sleeps
`CAMERA_TRIGGER_PULSE_WIDTH_S`, then drives it low - a rising-edge pulse, matching the digital
input's supported trigger modes (S2.10.1: "Rising or falling edge are supported for trigger"). Like
`CAMERA_TRIGGER_DELAY_S`, the pulse width isn't a datasheet number (the manual gives edge
propagation delays but no minimum pulse width) - `CAMERA_TRIGGER_PULSE_WIDTH_S` defaults to a
conservative `0.001` (1ms), several orders of magnitude above the microsecond-scale edge delays the
manual does give, and should be tuned once real hardware is available. If either GPIO write returns
a non-`OP_OK` `Drv.GpioStatus` (e.g. the pin is unconnected or faulted), `GpioTriggerError` is
logged and the capture is reported as failed - see Outcome Reporting.

**Requested delay shorter than the camera's own latency:** if `delay < CAMERA_TRIGGER_DELAY_S`, the
subtraction goes negative. `captureImageIn_handler` clamps the sleep to `0` (triggers immediately)
and logs `CaptureDelayTooShort` (`WARNING_LO`) rather than sleeping a negative amount of time or
silently missing the requested delay - the caller finds out the request couldn't be honored exactly.

## Outcome Reporting

`captureImageIn` is async and its port type has no return value (an async input port cannot have
one - only `sync`/`guarded` ports can), so a caller gets no synchronous result the way a call to
`healthCheckIn` would. Instead, each `captureImageIn` call's progress and outcome are observable via:

- `CaptureStarted` (logged on receipt, before the sleep) / `CaptureCompleted` / `CaptureFailed`
  events, plus `GpioTriggerError` specifically when the trigger pulse's GPIO writes fail
- `LastCaptureStatus` telemetry (the most recently completed call's `Types.CameraStatus`)
- `CapturesCompleted` telemetry (a running count, success or failure, since boot - so a caller
  polling telemetry can tell a new call finished even if the status repeats)

## Parameters

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `CAMERA_TRIGGER_DELAY_S` | `F32` | `0.0` | Camera's own trigger-to-exposure latency, in seconds. Subtracted from every `captureImageIn` request's `delay` - see Timing. Must be calibrated against real hardware; not a datasheet constant. |
| `CAMERA_TRIGGER_PULSE_WIDTH_S` | `F32` | `0.001` | How long `cameraTriggerOut` is held high before being pulled low, in seconds - see Timing. Conservative placeholder; not a datasheet constant. |
| `CAMERA_DEVICE_SN` | `string` | *(empty)* | Serial number of the physical XIMEA camera this instance opens, via `xiOpenDeviceBy(XI_OPEN_BY_SN, ...)`. Empty means "no camera configured" - every xiAPI operation then fails cleanly via `XiApiError`. Serial rather than enumeration index because USB order is not stable across boots - see `docs/xiapi-integration-draft.md`, S4.2. |
| `CAMERA_TIMEOUT_MS` | `U32` | `2000` | `xiGetImage` `TimeOut` in `captureImageIn_handler` - how long to wait for the triggered frame before reporting `CaptureFailed`. |
| `CAMERA_IMAGE_DATA_FORMAT` | `U32` | `0` | xiAPI `XI_IMG_FORMAT` value for `XI_PRM_IMAGE_DATA_FORMAT` (0 = `XI_MONO8`, 5 = `XI_RAW8`, 1 = `XI_MONO16`, 6 = `XI_RAW16`). Not an FPP enum so it can be corrected once the flown sensor variant is known without a type change. |

## Events

| Name | Severity | Description |
|------|----------|-------------|
| `CaptureDelayTooShort` | WARNING_LO | `captureImageIn`'s `delay` was less than `CAMERA_TRIGGER_DELAY_S`; triggered immediately instead. |
| `CaptureStarted` | ACTIVITY_HI | `captureImageIn` accepted; now sleeping out the adjusted wait before triggering. |
| `CaptureCompleted` | ACTIVITY_HI | The camera was triggered and reported success. |
| `CaptureFailed` | WARNING_HI | The camera was triggered but reported failure, including a failed trigger pulse, a `xiGetImage` failure, or a failed disk write. |
| `GpioTriggerError` | WARNING_HI | `cameraTriggerOut` returned a non-OK `Drv.GpioStatus` while pulsing the hardware trigger. |
| `XiApiError` | WARNING_HI | A xiAPI call (`xiOpenDeviceBy`, `xiSetParam`, `xiStartAcquisition`, `xiGetImage`, ...) returned a non-`XI_OK` status; carries the failing call name and the raw `XI_RETURN` code. |
| `ImageWriteError` | WARNING_HI | The retrieved frame could not be written to the requested `outputPath`. |

## Telemetry

| Name | Type | Description |
|------|------|-------------|
| `LastCaptureStatus` | `Types.CameraStatus` | Result of the most recently completed `captureImageIn` request. |
| `CapturesCompleted` | `U32` | Count of `captureImageIn` requests completed (success or failure) since boot. |
| `LastHealthCheckStatus` | `Types.CameraStatus` | Result of the most recent `healthCheckIn` call. |
| `LastImageBytes` | `U32` | Size of the frame retrieved by the most recent successful `xiGetImage` (`XI_IMG.bp_size`). |
| `CameraTemperatureC` | `F32` | Camera-reported temperature (`XI_PRM_TEMP`), refreshed on every `healthCheckIn`. |

## Requirements

| ID | Requirement | Verification |
|----|-------------|-------------|
| HS2-CAM-001 | CameraManager shall trigger a capture no sooner than `delay` seconds after `captureImageIn` is called, adjusted for the camera's own configured trigger-to-exposure latency (`CAMERA_TRIGGER_DELAY_S`) | Test |
| HS2-CAM-002 | CameraManager shall trigger immediately, and log `CaptureDelayTooShort`, if the requested `delay` is less than `CAMERA_TRIGGER_DELAY_S` | Test |
| HS2-CAM-003 | CameraManager shall report the outcome of every `captureImageIn` request via `CaptureCompleted`/`CaptureFailed` and `LastCaptureStatus`/`CapturesCompleted` | Test |
| HS2-CAM-004 | CameraManager shall respond to `healthCheckIn` synchronously, without waiting on any in-flight `captureImageIn` delay | Test |
| HS2-CAM-005 | CameraManager shall trigger the camera via a hardware GPIO pulse (`cameraTriggerOut`), not a software command, and report `GpioTriggerError`/`CaptureFailed` if that pulse's GPIO writes do not both return `Drv.GpioStatus.OP_OK` | Test |

## Open Items

- **Real xiAPI image retrieval and health query are now wired in** (was: "retrieving the resulting
  image is not [implemented]" / `healthCheckIn` "fully stubbed"). `captureImageIn_handler` calls
  `xiGetImage` after the GPIO pulse and writes the frame to `outputPath`; `healthCheckIn_handler`
  queries `XI_PRM_IS_DEVICE_EXIST` and samples `XI_PRM_TEMP`. Both go through `ensureDeviceReady()`,
  which lazily `xiOpenDeviceBy`s the camera by `CAMERA_DEVICE_SN`, configures the digital input as a
  rising-edge frame trigger (`XI_PRM_GPI_MODE` / `XI_PRM_TRG_SOURCE` - without this the GPIO pulse
  does nothing on real hardware), starts acquisition, and flushes the buffers queue. See
  `docs/xiapi-integration-draft.md` for the full rationale. Requires the XIMEA Linux SDK
  (`libm3api` / `<m3api/xiApi.h>`) at build time - wired up as the `xiapi` CMake target in the
  top-level `CMakeLists.txt`.
- **xiAPI per-handle thread-safety is assumed unsafe.** `healthCheckIn` (`sync`, runs on its
  caller's thread) and `captureImageIn` (this component's own thread) can touch the same device
  handle concurrently, and the vendor manual does not document whether that is safe. Every xiAPI
  call is currently serialized behind one `std::mutex`, which means a `healthCheckIn` can block for
  up to `CAMERA_TIMEOUT_MS` behind an in-flight `xiGetImage` - contradicting the SDD's "answers
  immediately". Resolving this properly needs confirmation from XIMEA (`docs/xiapi-integration-draft.md`, S3/S6.1).
- **Camera power is still not switched anywhere.** `xiGetNumberDevices` / `xiOpenDeviceBy` will
  report "no device" until whatever powers the camera on exists; that surfaces as a normal
  `XiApiError` + `CaptureFailed` / health-check failure, not a crash (`docs/xiapi-integration-draft.md`, S6.6).
- **`cameraTriggerOut` has no GPIO driver connected.** Unlike a `void` output port (a safe no-op
  when unconnected), a *return-valued* output port like `Drv.GpioWrite` has no sensible default to
  return - the autocoded `cameraTriggerOut_out` `FW_ASSERT`s if called while unconnected. So
  `captureImageIn_handler` guards the whole trigger-pulse block behind
  `isConnected_cameraTriggerOut_OutputPort(0)`: with nothing wired up (true today), it skips the
  GPIO calls entirely and reports success, purely to keep the timing/telemetry path exercisable in
  isolation - it does *not* mean "trigger succeeded." `GpioTriggerError`/a trigger-caused
  `CaptureFailed` can only fire once a real `Drv.LinuxGpioDriver` (or equivalent) instance is
  connected in topology.
- **`CAMERA_TRIGGER_DELAY_S`/`CAMERA_TRIGGER_PULSE_WIDTH_S` have never been measured.** Their
  defaults are safe-but-suboptimal fallbacks (see Timing) - getting real numbers requires timing an
  actual triggered exposure on flight-like hardware/cabling, not a manual lookup.
- **Not yet wired into any topology.** No `instance` of `CameraManager` exists in `Top/
  instances.fpp` on this branch yet, and no GPIO driver instance is available to connect
  `cameraTriggerOut` to even if it were; `DataCollectionApplication` (which would own
  `lostCameraManager`/`foundCameraManager` instances of it) is developed on a separate branch and
  isn't present here either. Connecting all of this is topology-only work once the pieces merge.

## Change Log

| Date | Description |
|---|---|
| 2026-08-25 | Initial design: `captureImageIn` (async, delay-adjusted for camera latency) and `healthCheckIn` (sync) |
| 2026-08-25 | Added `cameraTriggerOut` (`Drv.GpioWrite`): capture now hardware-triggers via a GPIO pulse instead of an implicit software trigger, with `CAMERA_TRIGGER_PULSE_WIDTH_S` and `GpioTriggerError` |
| 2026-09-02 | Wired in real xiAPI: `ensureDeviceReady()` (open-by-serial + one-time trigger/format config + acquisition start + queue flush), `xiGetImage` + disk write in `captureImageIn` (new `outputPath` port arg), real `healthCheckIn` via `XI_PRM_IS_DEVICE_EXIST`. Added `CAMERA_DEVICE_SN` / `CAMERA_TIMEOUT_MS` / `CAMERA_IMAGE_DATA_FORMAT` params, `XiApiError` / `ImageWriteError` events, `LastImageBytes` / `CameraTemperatureC` telemetry. Build now depends on the XIMEA `libm3api` SDK. |
