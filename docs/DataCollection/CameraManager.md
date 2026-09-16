## CameraManager

`CameraManager` is a Layer 2 queued component driving one physical camera. It is meant to be
instantiated once per camera.

The camera hardware itself is a XIMEA xiC USB3 camera on a Sony Pregius CMOS sensor. The component 
mainly relies on xiAPI (https://www.ximea.com/support/wiki/apis/XiAPI_Manual) to perform all camera
operations, except for image captures which happens through a GPIO pin.

#### Requirements {.unnumbered .unlisted}

| ID | Requirement | Verification |
|----|-------------|-------------|
| HS2-CAM-001 | CameraManager shall trigger a capture after `captureImageIn` is called | Test |
| HS2-CAM-002 | CameraManager shall report the outcome of every `captureImageIn` request via `CaptureCompleted`/`CaptureFailed` and `LastCaptureStatus`/`CapturesCompleted` | Test |
| HS2-CAM-003 | CameraManager shall respond to `healthCheckIn` synchronously, without waiting on any in-flight `captureImageIn` delay | Test |
| HS2-CAM-004 | CameraManager shall trigger the camera via a hardware GPIO pulse (`cameraTriggerOut`) and report `GpioTriggerError`/`CaptureFailed` if that pulse's GPIO writes do not both return `Drv.GpioStatus.OP_OK` | Test |

#### Ports {.unnumbered .unlisted}

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `captureImageIn` | Input, async | DataCollection.CameraCapture with imageType Types.ImageType, delay F32, and outputPath string | Requests one capture of `imageType`, writing the retrieved frame to `outputPath`. |
| `healthCheckIn` | Input, sync | DataCollection.CameraCheckup returning Types.CameraStatus | Per-camera health check |
| `cameraTriggerOut` | Output | Drv.GpioWrite with state Fw.Logic, returning Drv.GpioStatus | Drives the GPIO pin wired to this camera's hardware trigger input |

#### Parameters {.unnumbered .unlisted}

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `CAMERA_TRIGGER_PULSE_WIDTH_S` | `F32` | `0.001` | How long `cameraTriggerOut` is held high before being pulled low |
| `CAMERA_DEVICE_SN` | `string` | *(empty)* | Serial number of the physical XIMEA camera this instance opens, via `xiOpenDeviceBy(XI_OPEN_BY_SN, ...)` |
| `CAMERA_TIMEOUT_MS` | `U32` | `2000` | how long to wait for the triggered frame before reporting `CaptureFailed`. |
| `CAMERA_IMAGE_DATA_FORMAT` | `U32` | `0` | xiAPI `XI_IMG_FORMAT` value for `XI_PRM_IMAGE_DATA_FORMAT` (0 = `XI_MONO8`, 5 = `XI_RAW8`, 1 = `XI_MONO16`, 6 = `XI_RAW16`) |
