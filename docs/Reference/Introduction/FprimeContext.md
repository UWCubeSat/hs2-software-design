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

## From Component SDD to FPP and C++

The component SDD is the design contract for implementation: its port, command, event, telemetry, parameter, and state-machine descriptions are translated into an FPP component definition. The FPP file is the interface source of truth, similar to an API or header, but it is also machine-readable so F' can check connections and generate code.

For example, a port and command from an SDD become declarations in a component model:

```fpp
active component Example {
    sync input port dataIn: ExampleData
    output port resultOut: ExampleResult
    async command PROCESS(value: U32)
}
```

Running the F' tooling (often through `fprime-util new --component`, `fprime-util impl`, and the deployment build) generates typed C++ base files such as `ExampleComponentAc.hpp` and `ExampleComponentAc.cpp`. These generated files provide the base class, port and command plumbing, and framework integration. Developers implement the behavior in the hand-written `Example.hpp` and `Example.cpp` files by filling in the generated handler methods; they do not edit the generated `Ac` files. The topology's FPP definitions then instantiate the component and connect its ports. In this workflow, the SDD tells an implementer what the component must do, the FPP defines the precise machine-checked interface, and the generated C++ files provide the implementation starting point.

See the F' [component development process](https://fprime.jpl.nasa.gov/latest/docs/user-manual/overview/development-practice/), [Hello World component tutorial](https://fprime.jpl.nasa.gov/latest/tutorials-hello-world/docs/hello-world), and [autocoded functions reference](https://fprime.jpl.nasa.gov/latest/docs/user-manual/framework/autocoded-functions/) for the complete workflow.

## Topology Assembly

The deployment topology is responsible for more than simply listing components. It must:

1. instantiate components with the required queues, priorities, and stack sizes;
2. call `init()` and apply deployment configuration;
3. connect typed ports and validate the intended data flow;
4. register commands and load parameters;
5. configure rate groups and health-monitoring entries; and
6. start active components in a safe order.

HuskySat-2 will preserve the same component interfaces when moving from a GDS deployment to the BeagleBone Black flight deployment. That keeps application and manager tests independent of the final physical bus wiring.

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

Refer to the separate Ground Data System (GDS) document for the ground-station hardware, ground software, networking, and operator-facing workflows. This SDD defines the flight-software boundary: command decoding, validation, dispatch, status response, telemetry, and event interfaces that the ground system consumes. It does not duplicate the ground-software design; the GDS document is the authoritative source for that material.
