# LinuxUsbDriver SDD

## 1. Overview

`LinuxUsbDriver` is a Layer 1 passive driver for a native USB device attached to the Linux flight computer in host mode. It provides control and bulk transfers through a device-independent interface; device enumeration, packet formats, and recovery policy remain the responsibility of the owning Layer 2 hardware manager.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-USB-001 | LinuxUsbDriver shall open the configured USB device and claim its configured interface during topology setup. | Inspection |
| HS2-USB-002 | LinuxUsbDriver shall provide synchronous USB control-transfer operations. | Inspection |
| HS2-USB-003 | LinuxUsbDriver shall provide synchronous bulk read and bulk write operations. | Inspection |
| HS2-USB-004 | LinuxUsbDriver shall serialize access to the claimed USB interface. | Inspection |
| HS2-USB-005 | LinuxUsbDriver shall return a transfer status to its caller and report open and transfer failures through F' events. | Inspection |
| HS2-USB-006 | LinuxUsbDriver shall release the claimed interface and close the device during topology teardown. | Inspection |

---

## 3. Design

### 3.1 Component Type

Passive component. All transfers execute on the caller's thread and are protected by the component mutex. The driver supports control and bulk endpoints only. Streaming, isochronous, or interrupt-endpoint devices would need a device-specific extension that defines buffering, scheduling, and recovery behavior.

### 3.2 Ports

The following project-defined ports form the `Drv.Usb` interface.

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `usbControl` | Input (sync) | `Drv.UsbControl` | Perform one USB control transfer using the supplied setup fields and buffer. |
| `usbBulkRead` | Input (sync) | `Drv.UsbBulkRead` | Read data from a supplied bulk-IN endpoint into a caller-owned buffer. |
| `usbBulkWrite` | Input (sync) | `Drv.UsbBulkWrite` | Write a caller-owned buffer to a supplied bulk-OUT endpoint. |
| `logOut` | Output | `Fw.Log` | Report device-open and transfer failures. |

Each operation returns `Drv.UsbStatus`.

| Status | Meaning |
|--------|---------|
| `USB_OK` | Operation succeeded. |
| `USB_NOT_OPENED` | The device or interface is not available. |
| `USB_INVALID_PARAM` | The request contains an invalid endpoint, setup field, buffer, or timeout. |
| `USB_TIMEOUT` | The transfer did not complete before its timeout. |
| `USB_TRANSFER_ERR` | The operating system or USB stack rejected or failed the transfer. |
| `USB_OTHER_ERR` | An unexpected driver error occurred. |

### 3.3 Configuration and Operation

Topology setup calls `open()` with the device's vendor ID, product ID, optional serial number, configuration value, interface number, and alternate setting. The driver uses `libusb` to find the device, select the configuration and alternate setting, and claim the interface. It does not select a device by its Linux bus address.

The manager supplies endpoint addresses, transfer buffers, and timeouts with each operation. Endpoint selection and all device-protocol interpretation belong to the manager because they depend on the selected USB device.

The caller owns every buffer passed to the driver. The driver neither retains nor releases those buffers.

---

## 4. Notes

- `LinuxUsbDriver` requires `libusb` in the flight build.

- Add a concrete topology instance only after a USB peripheral and its transfer model have been selected. At that point, record the vendor/product IDs, interface, endpoint map, expected bandwidth, timeout values, and power-cycle owner in that device's manager SDD.
