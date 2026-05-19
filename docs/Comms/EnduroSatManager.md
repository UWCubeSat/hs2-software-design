# EnduroSatManager SDD

## 1. Overview
`EnduroSatManager` is the layer 2 active component for the Comms subtopology. It directly interacts with the Endurosat S-band radio using a UART interface. The `CommsApplication` component will forward commands and data to the `EnduroSatManager` to be fed into the Endurosat S-band radio. The `EnduroSatManager` is one of many layer 2 hardware managers as described in `sdd.md`.

---

## 2. Requirements
| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-ESM-001 | EnduroSatManager shall transmit and receive data over UART | Inspection |
| HS2-ESM-002| EnduroSatManager shall maintain a status indicating the state of connection with the S-Band transceiver at all times | Inspection/Unit tests |
| HS2-ESM-003 | EnduroSatManager shall frame all telemetry packets inside of CCSDS Space Packet headers | Inspection/Unit tests |
| HS2-ESM-004 | EnduroSatManager shall validate all uplinked commands against the CCSDS Space Packet Protocol | Inspection/Unit tests |
| HS2-ESM-005 | EnduroSatManager shall deframe all uplinked commands that were validated against the CCSDS Space Packet Protocol | Inspection/Unit tests |

---

## 3. Design

## 3.1 Component Type
Passive component that implements the F' `Communication Adapter Interface` that specifies both the ports and protocols used to operate with the standard F´ uplink and downlink components.

## 3.2 Communication Adapter Interface
Any communication component (e.g. a radio component) that is intended for use with the standard F´ uplink and downlink stack should implement the Communication Adapter Interface. This interface specifies both the ports and protocols used to operate with the standard F´ uplink and downlink components.

The communication adapter interface protocol is designed to work alongside the framer status protocol and the com queue protocol to ensure that data messages do not overload a communication interface. These protocols are discussed below.

## 3.3 Ports
The communication adapter interface is composed of five ports. These ports are used to transmit outgoing data through some communication hardware and receive incoming data from that same hardware. These ports share types with the ByteStreamDriver model for backwards compatibility:

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
|`dataIn`| Input | `Svc.ComDataWithContext`| Data to be sent on wire (coming in to component) |
|`dataOut`| Output | `Svc.ComDataWithContext` | Data received from the wire (going out of component)|
|`comStatusOut`| Output | `Fw.SuccessCondition` | Status of last transmission (required for data flow)|
|`dataReturnOut`| Output | `Svc.ComDataWithContext`| Port returning ownership of data that came in on dataIn|
|`dataReturnIn` | Input | `Svc.ComDataWithContext` | Port receiving back ownership of buffer sent out on dataOut|

Additionally, the `EnduroSatManager` uses the ByteStreamDriverClient port interface to communicate over UART with the S-Band transciever as described in the table below:

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
|`drvConnected`| Input | `Drv.ByteStreamReady`| Ready signal for S-Band Transceiver's UART connection |
|`drvReceiveIn`| Input | `Drv.ByteStreamData`| Receives data from UART driver to S-Band Transceiver|
|`drvReceiveReturnOut`| Output | `Fw.BufferSend`| Returns ownership of buffer arriving on `drvReceiveIn`|
|`drvSendOut`| Output | `Drv.ByteStreamSend`| Sends data to S-Band transceiver|

## 4. Notes
- `EnduroSatManager` is instantiated at the top-level topology (shared with `ComCcsds` subtopology); `CommsApplication` connects to it via the top-level topology wiring.
- Detailed high-gain link configuration and `EnduroSatManager` interface to be defined during detailed design.
