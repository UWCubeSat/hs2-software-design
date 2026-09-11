# F' Context

F' (F Prime) is a component-driven embedded flight-software framework developed at NASA JPL for small-scale spacecraft and other embedded systems. HuskySat-2 uses F' to separate reusable framework infrastructure from mission-specific applications, hardware managers, and drivers.

## Framework Structure

The F' repository is organized around several layers of reusable infrastructure:

| Area | Role in a deployment |
|------|----------------------|
| `Fw/` | Runtime types and core abstractions such as buffers, commands, logs, telemetry, time, and state machines. |
| `Svc/` | Reusable active, queued, and passive service components for CDH, timing, communications, files, data products, and diagnostics. |
| `Drv/` | Operating-system and bus drivers, including Linux UART, SPI, I2C, and GPIO components. |
| `Os/` | Operating-system abstraction layer used by framework and deployment code. |
| `Fpp/` | FPP modeling and code-generation toolchain for component and topology definitions. |
| `Ref/` | Reference deployment showing component instantiation, initialization, wiring, and startup. |

## Core Concepts

F' components communicate through typed ports. A topology creates component instances, initializes them, connects their ports, registers commands, loads parameters, and starts active components. FPP describes the interfaces and topology structure; C++ provides implementation logic and deployment-specific assembly.

F' supports four important port execution styles:

| Port style | Execution behavior | Typical use |
|------------|--------------------|-------------|
| Output | Calls the connected input through the caller's thread. | Synchronous request or data delivery. |
| Synchronous input | Executes immediately in the calling thread and may return data. | Short, deterministic operations. |
| Asynchronous input | Queues work for an active component's execution context. | Commands and longer-running work. |
| Guarded input | Executes synchronously while protecting shared state with a mutex. | Thread-safe access to shared data. |

The main F' data interfaces are commands, events, telemetry channels, and parameters. Commands represent uplinked operator instructions and are routed through `Svc::CmdDispatcher`. Events record component activity and faults and are routed through `Svc::EventManager`. Telemetry channels represent current state and are collected by `Svc::TlmChan` and `Svc::TlmPacketizer`. Parameters hold configurable values and may be persisted by `Svc::PrmDb`.

## Topology Assembly

The deployment topology is responsible for more than simply listing components. It must:

1. instantiate components with the required queues, priorities, and stack sizes;
2. call `init()` and apply deployment configuration;
3. connect typed ports and validate the intended data flow;
4. register commands and load parameters;
5. configure rate groups and health-monitoring entries; and
6. start active components in a safe order.

HuskySat-2 should preserve the same component interfaces when moving from a GDS deployment to the BeagleBone Black flight deployment. That keeps application and manager tests independent of the final physical bus wiring.

## OSAL and Linux on the BeagleBone Black

F' uses an operating-system abstraction layer (OSAL) so framework code can request common services without embedding a particular operating system into every component. The OSAL provides wrappers for tasks, queues, mutexes, time, files, directories, and related operating-system resources. The Level 1 driver boundary similarly provides typed F' interfaces for Linux serial and bus access.

On the BeagleBone Black, these abstractions are implemented with Linux and POSIX facilities. Flight components therefore use F' and OSAL interfaces for scheduling, synchronization, timing, and file access, while Linux-specific details remain in the driver or deployment layer. This keeps the application and hardware-manager code testable in GDS and on a host system, while still allowing the deployed topology to use the BeagleBone's UART, I2C, SPI, GPIO, PWM, and filesystem resources.

The OSAL is not a substitute for a device manager. It can open a file or serial endpoint and provide a task or queue, but a Level 2 manager still owns the device protocol, register meanings, initialization sequence, and hardware-specific recovery policy.

## Pre-Built Services and Subtopologies

F' provides reusable components and importable subtopologies. A subtopology is a pre-wired group of component instances and connections, while its individual components still have their own behavior and interfaces. HuskySat-2 documents the individual F' components used by the Level 3 application layer so their role is visible without reproducing the entire upstream repository summary.

The principal upstream groupings are:

- `CdhCore`: command dispatch, events, health, version, text logging, and fatal adaptation;
- `ComCcsds`: packet routing, CCSDS framing/deframing, aggregation, and the byte-stream communications bridge;
- `FileHandling`: file uplink, file downlink, file management, and parameter storage; and
- `DataProducts`: data-product allocation, writing, cataloging, and downlink preparation.

## Design Patterns Used

### App-Manager-Driver

Applications own mission behavior, managers translate device-specific protocols and recovery behavior, and drivers expose hardware or operating-system access. This prevents mission applications from depending directly on bus details and allows managers to be tested against simulated drivers.

### Rate Groups

`Svc::RateGroupDriver` divides a primary clock into multiple rates, while `Svc::ActiveRateGroup` invokes configured components at those rates. Control loops, telemetry collection, and background work can therefore have separate scheduling requirements. Blocking work in a passive rate group can delay other work; active rate groups isolate execution at the cost of thread scheduling and possible jitter.

### Health Checking

`Svc::Health` periodically pings critical active components through their ping ports. Components echo the ping key promptly; missed responses become warning or fatal health events according to configured thresholds. Background workers and components without mission-critical liveness obligations may be excluded.

### Manager-Worker

The manager-worker pattern separates high-priority command handling from long-running background work. The manager tracks state and cancellation while a worker performs the operation and reports completion. This pattern is relevant to file operations and future workloads that must not block command responsiveness.

### Subtopologies and Common Ports

Subtopologies package reusable component instances and connections for import into a deployment. Common F' port patterns include synchronous gets, asynchronous callbacks, parallel port arrays, and synchronous cancellation. These patterns make interfaces explicit and allow FPP to validate connection compatibility before deployment.

## Ground Interface

The F' ground interface is two-sided and layered. The uplink path follows the general form:

`Driver -> FrameAccumulator -> Deframer -> FprimeRouter -> CmdDispatcher`

The downlink path follows:

`Svc::EventManager/Svc::TlmChan -> Svc::ComQueue -> Framer -> Driver`

HuskySat-2's CCSDS service components fill in the framing and packet-routing stages around `TmtcRadioManager` and `LinuxUartDriver`.

The command and telemetry list, including both F' events and telemetry channels, can be viewed in the CTL document. The CTL document is maintained separately and is not part of this SDD.
