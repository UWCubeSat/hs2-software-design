# EnduroSatManager SDD

## 1. Overview
`EnduroSatManager` is the layer 2 passive component for the Comms subtopology. It directly interacts with the Endurosat S-band transceiver through a UART interface. `The `EnduroSatManager` is one of many layer 2 hardware managers as described in `sdd.md`.

---

## 2. Requirements
| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-ESM-001 | EnduroSatManager shall forward CCSDS Space Protocol telemetry packets over UART to the S-Band transceiver for downlink.  | Inspection |
| HS2-ESM-002 | EnduroSatManager shall forward CCSDS Space Protocol command packets received over UART to the `CommandDispatcher` for processing. | Inspection |
| HS2-ESM-003| EnduroSatManager shall maintain a status indicating the state of connection with the S-Band transceiver at all times | Inspection/Unit tests |

---

## 3. Design

## 3.1 Component Type
Passive component that implements the F' `Communication Adapter Interface` that specifies both the ports and protocols used to operate with the standard F´ uplink and downlink components.

## 3.2 Communication Adapter Interface
Any communication component (e.g. a radio component) that is intended for use with the standard F´ uplink and downlink stack should implement the Communication Adapter Interface. This interface specifies both the ports and protocols used to operate with the standard F´ uplink and downlink components.

The communication adapter interface protocol is designed to work alongside the framer status protocol and the com queue protocol to ensure that data messages do not overload a communication interface. These protocols are discussed below.

### 3.2.1 Communication Queue Protocol
`Svc::ComQueue` queues messages until the communication adapter is ready to receive these messages. For each Fw::Success::SUCCESS message received, Svc::ComQueue will emit one message. Svc::ComQueue will not emit messages at any other time. This implies several things:

1. An initial Fw::Success::SUCCESS message must be sent to Svc::ComQueue to initiate data flow.
2. A Fw::Success::SUCCESS must be eventually received in response to each message for data to continue to flow.

### 3.2.2 Communication Framer Protocol
Framing typically happens between `Svc::ComQueue` and a communications adapter. The action taken by this protocol is dependent on the number of framed messages (frames) sent to the communication adapter in response to each message received from `Svc::ComQueue`.

The `Svc::FprimeFramer` implements the Framer status protocol by emitting a frame for each message received from `Svc::ComQueue`. Therefore, the Svc::FprimeFramer receives a status from the communication adapter on each sent frame and forwards it back to `Svc::ComQueue`. This allows the data flow to continue, or pause if the communication adapter is unable to receive more data.

Framer implementations may choose to implement a different behavior. For example, a framer may choose to accept multiple messages from Svc::ComQueue and concatenate them in a single frame. In this case, the framer implementation must manage the flow of statuses accordingly. This is summarized in the table below.

| Produced Frames | Action | Rationale |
|-----------------|--------|-----------|
| 0 | Send one `Fw::Success::SUCCESS` status | Data must continue to flow to produce a frame |
| 1 | Pass through received status from ComQueue | Frame produced and communication adapter produces status |

For HS-2, we will be utilizing the `Svc::Ccsds::SpacePacketFramer` and `Svc::Ccsds::SpacePacketDeframe`.

## 3.3 Ports
The communication adapter interface is composed of five ports. These ports are used to transmit outgoing data through some communication hardware and receive incoming data from that same hardware. These ports share types with the ByteStreamDriver model for backwards compatibility:

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
|`dataIn`| Input | `Svc.ComDataWithContext`| Data to be sent on wire (coming in to component) |
|`dataOut`| Output | `Svc.ComDataWithContext` | Data received from the wire (going out of component)|
|`comStatusOut`| Output | `Fw.SuccessCondition` | Status of last transmission (required for data flow)|
|`dataReturnOut`| Output | `Svc.ComDataWithContext`| Port returning ownership of data that came in on dataIn|
|`dataReturnIn` | Input | `Svc.ComDataWithContext` | Port receiving back ownership of buffer sent out on dataOut|

Additionally, the `EnduroSatManager` uses the `ByteStreamDriverClient` port interface to communicate over UART with the S-Band transciever as described in the table below:

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
|`drvConnected`| Input | `Drv.ByteStreamReady`| Ready signal for S-Band Transceiver's UART connection |
|`drvReceiveIn`| Input | `Drv.ByteStreamData`| Receives data from UART driver to S-Band Transceiver|
|`drvReceiveReturnOut`| Output | `Fw.BufferSend`| Returns ownership of buffer arriving on `drvReceiveIn`|
|`drvSendOut`| Output | `Drv.ByteStreamSend`| Sends data to S-Band transceiver|

## 4. Notes
- `EnduroSatManager` is instantiated at the top-level topology (shared with `ComCcsds` subtopology); `CommsApplication` connects to it via the top-level topology wiring.
- Detailed high-gain link configuration and `EnduroSatManager` interface to be defined during detailed design.
