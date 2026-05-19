# F' (F Prime) Repository Summary
> Reference guide for writing a satellite Software Design Document using the F' framework.
> Sources: `nasa/fprime@devel` and `nasa/fprime-examples@devel`

---

## 1. Framework Overview

F' (F Prime) is a **component-driven embedded software framework** developed at NASA JPL, designed for small-scale space missions (CubeSats, SmallSats). Key characteristics:

- Systems decompose into discrete **Components** communicating through **Ports**
- A **Topology** is the runtime wiring graph of component instances
- Built-in support for commands (uplink), events, telemetry channels, and parameters (downlink)
- Code generation from FPP (F Prime Prime) modeling language
- C++ runtime infrastructure providing message queues and threading
- Pre-built library of reusable service and driver components

---

## 2. Repository Structure (`nasa/fprime@devel`)

```
fprime/
├── docs/                    # All documentation
│   ├── user-manual/
│   │   ├── overview/        # Core concepts and architecture
│   │   ├── design-patterns/ # Architectural design patterns
│   │   ├── framework/       # Framework features (topology, data products, etc.)
│   │   ├── gds/             # Ground Data System docs
│   │   └── security/
│   └── reference/           # Nomenclature, types, GDS plugins
├── Fw/                      # Framework core (Buffer, Cmd, Com, Log, Tlm, Time, Sm, etc.)
├── Svc/                     # Pre-built service components (~65 components)
│   └── Subtopologies/       # Pre-assembled reusable subtopologies
├── Drv/                     # Driver components (TCP, UDP, UART, SPI, I2C, GPIO)
├── Os/                      # OS abstraction layer
├── Utils/                   # Utility components
├── CFDP/                    # CCSDS File Delivery Protocol
├── Ref/                     # Reference application (example satellite-like deployment)
│   └── Top/                 # Ref topology definition — best example of full assembly
└── Fpp/                     # FPP language toolchain
```

**Rule:** Always check docs first (markdown), then FPP source code, then C++ implementation.

---

## 3. Core Concepts

### 3.1 Ports
Typed interfaces between components. Four kinds:
| Kind | Synchronous | Thread | Returns data |
|------|-------------|--------|--------------|
| `output` | Yes | Caller's | Yes |
| `sync_input` | Yes | Caller's | Yes |
| `async_input` | No (queued) | Component's own | No |
| `guarded_input` | Yes, mutex-locked | Caller's | Yes |

**Key doc:** `docs/user-manual/overview/03-port-comp-top.md`

### 3.2 Components
Three types:
| Type | Queue | Thread | Use case |
|------|-------|--------|----------|
| Passive | No | No | Simple logic, called inline |
| Active | Yes | Yes | Background tasks, high autonomy |
| Queued | Yes | No | Deferred execution without own thread |

**Key doc:** `docs/user-manual/overview/03-port-comp-top.md`

### 3.3 Commands, Events, Channels, Parameters
- **Commands** — uplinked operator instructions; routed by `Svc::CmdDispatcher`; can be sequenced via `Svc::CmdSequencer`
- **Events** — activity logs with severity (DIAGNOSTIC → FATAL); timestamped; routed through `Svc::EventManager`
- **Channels (Telemetry)** — current state values downlinked periodically; stored in `Svc::TlmChan`
- **Parameters** — non-volatile settings; persisted by `Svc::PrmDb`; survive reboots

**Key doc:** `docs/user-manual/overview/04-cmd-evt-chn-prm.md`

### 3.4 Topology Assembly (C++)
Six steps in `<Project>Topology.cpp`:
1. Instantiate components (static, heap, or stack)
2. Call `init()` on each (pass queue size for active/queued)
3. Wire ports (`set_<port>_OutputPort` / `get_<port>_InputPort`)
4. Register commands (`regCommands()`)
5. Load parameters (`loadParameters()`)
6. Start active components (`start(id, priority, stackSize)`)

**Key doc:** `docs/user-manual/framework/building-topology.md`
**Code example:** `Ref/Top/RefTopology.cpp`, `Ref/Top/topology.fpp`

---

## 4. Pre-Built Service Components (`Svc/`)

### Command & Data Handling (CDH)
| Component | Type | Purpose |
|-----------|------|---------|
| `Svc::CmdDispatcher` | Active | Routes raw command buffers to components by opcode |
| `Svc::CmdSequencer` | Active | Loads and executes command sequences from files |
| `Svc::EventManager` | Active | Receives and downlinks events; routes FATAL events |
| `Svc::TlmChan` | Active | Double-buffered telemetry database |
| `Svc::TlmPacketizer` | Active | Packages telemetry channels into downlink packets |
| `Svc::PrmDb` | Active | Parameter database; persists to `PrmDb.dat` |
| `Svc::Health` | Queued | Monitors component liveness via ping/pong protocol |
| `Svc::Version` | Passive | Reports software version telemetry |
| `Svc::AssertFatalAdapter` | Passive | Converts C++ asserts to FATAL events |

### Timing & Scheduling
| Component | Type | Purpose |
|-----------|------|---------|
| `Svc::RateGroupDriver` | Passive | Clock source; performs frequency division |
| `Svc::ActiveRateGroup` | Active | Executes a list of components at a fixed rate |
| `Svc::PassiveRateGroup` | Passive | Same but synchronous in caller's thread |
| `Svc::LinuxTimer` | Active | Linux-based timer driving rate group driver |
| `Svc::PosixTime` | Passive | POSIX clock time source |
| `Svc::ChronoTime` | Passive | C++ chrono-based time source |
| `Svc::Cycle` | Passive | Cycle port utilities |

### Communications & Framing
| Component | Type | Purpose |
|-----------|------|---------|
| `Svc::ComQueue` | Active | Queues events and telemetry packets for downlink |
| `Svc::ComAggregator` | Passive | Aggregates multiple outgoing data streams |
| `Svc::ComStub` | Passive | Com interface bridge to byte-stream driver |
| `Svc::FrameAccumulator` | Passive | Accumulates bytes into complete frames |
| `Svc::FprimeRouter` | Passive | Routes deframed packets to correct destinations |
| `Svc::FprimeFramer` | Passive | Frames packets in F' native protocol |
| `Svc::FprimeDeframer` | Passive | Deframes F' native protocol packets |
| `Svc::Ccsds::TmFramer` | Passive | CCSDS TM frame framer |
| `Svc::Ccsds::TcDeframer` | Passive | CCSDS TC frame deframer |
| `Svc::Ccsds::SpacePacketFramer` | Passive | CCSDS Space Packet framer |
| `Svc::Ccsds::SpacePacketDeframer` | Passive | CCSDS Space Packet deframer |
| `Svc::Ccsds::ApidManager` | Passive | Manages CCSDS APID sequence counters |
| `Svc::ComRetry` | Passive | Retry logic for COM send failures |
| `Svc::ComSplitter` | Passive | Splits outgoing COM stream |

### File Operations
| Component | Type | Purpose |
|-----------|------|---------|
| `Svc::FileUplink` | Active | Receives file uplink packets; reconstructs files |
| `Svc::FileDownlink` | Active | Queues and downlinks files to ground |
| `Svc::FileManager` | Active | Provides commands to manipulate filesystem |
| `Svc::FileWorker` | Active | Worker for file operations (manager-worker pattern) |
| `Svc::FileDispatcher` | Active | Dispatches file handling to workers |
| `Svc::BufferLogger` | Active | Logs incoming buffers to files |
| `Svc::ComLogger` | Active | Logs COM data to files |

### Data Products (Science Data / Recording)
| Component | Type | Purpose |
|-----------|------|---------|
| `Svc::DpManager` | Active | Allocates memory for data product containers |
| `Svc::DpWriter` | Active | Writes filled data product containers to storage |
| `Svc::DpCatalog` | Active | Catalogs stored DPs; manages downlink priority queue |

### Memory & Buffer Management
| Component | Type | Purpose |
|-----------|------|---------|
| `Svc::BufferManager` | Active | Pool allocator for `Fw::Buffer` objects |
| `Svc::BufferAccumulator` | Active | Accumulates/holds buffers |
| `Svc::BufferRepeater` | Passive | Fans out a buffer to multiple outputs |
| `Svc::StaticMemory` | Passive | Static memory allocation helper |

### Diagnostics & System
| Component | Type | Purpose |
|-----------|------|---------|
| `Svc::SystemResources` | Passive | Reports CPU, memory, disk usage as telemetry |
| `Svc::Fatal` | Passive | FATAL event handler hook |
| `Svc::FatalHandler` | Passive | Resets system on FATAL events |
| `Svc::WatchDog` | — | Watchdog stroking |
| `Svc::ActiveTextLogger` | Active | Logs events as human-readable text |
| `Svc::PassiveConsoleTextLogger` | Passive | Console text event logger |
| `Svc::PolyDb` | Passive | Generic polymorphic data store |

---

## 5. Pre-Built Driver Components (`Drv/`)

| Component | Purpose |
|-----------|---------|
| `Drv::TcpClient` | TCP client byte-stream driver |
| `Drv::TcpServer` | TCP server byte-stream driver |
| `Drv::Udp` | UDP byte-stream driver |
| `Drv::LinuxUartDriver` | Linux UART byte-stream driver |
| `Drv::LinuxSpiDriver` | Linux SPI bus driver |
| `Drv::LinuxI2cDriver` | Linux I2C bus driver |
| `Drv::LinuxGpioDriver` | Linux GPIO driver |
| `Drv::ByteStreamDriverModel` | Abstract model; custom drivers implement this |
| `Drv::AsyncByteStreamBufferAdapter` | Adapts async byte-stream to buffer-based interface |

**ByteStreamDriverModel interface** (`Drv/ByteStreamDriverModel/docs/sdd.md`):
- `send` port → transmit data; returns `OP_OK`, `SEND_RETRY`, or `OTHER_ERROR`
- `recv` port → deliver received data; returns `OP_OK`, `RECV_NO_DATA`, or `OTHER_ERROR`
- `ready`, `poll` ports for driver signaling

---

## 6. Pre-Built Subtopologies (`Svc/Subtopologies/`)

Subtopologies are importable, pre-wired component clusters. Use `import <Name>.Subtopology` in FPP.

### CdhCore (`Svc/Subtopologies/CdhCore/CdhCore.fpp`)
The core CDH stack. Includes:
- `cmdDisp` (Svc.CommandDispatcher)
- `events` (Svc.EventManager)
- `$health` (Svc.Health)
- `version` (Svc.Version)
- `textLogger` (Svc.PassiveTextLogger)
- `fatalAdapter` (Svc.AssertFatalAdapter)

Exposes ports: command buffer in/out, event packet send, telemetry packet send, telemetry trigger, command dispatch schedule, health schedule.

### ComCcsds (`Svc/Subtopologies/ComCcsds/ComCcsds.fpp`)
Full CCSDS communications stack. Includes:
- `comQueue`, `frameAccumulator`, `commsBufferManager`, `fprimeRouter`
- `tcDeframer`, `spacePacketDeframer` (uplink)
- `framer` (TmFramer), `spacePacketFramer`, `apidManager`, `aggregator` (downlink)
- `comStub` (bridges to byte-stream driver)

Downlink path: `ComQueue → SpacePacketFramer → Aggregator → TmFramer → ComStub → Driver`
Uplink path: `Driver → ComStub → FrameAccumulator → TcDeframer → SpacePacketDeframer → FprimeRouter`

### FileHandling (`Svc/Subtopologies/FileHandling/FileHandling.fpp`)
File uplink, downlink, and management. Includes:
- `fileUplink` (Svc.FileUplink)
- `fileDownlink` (Svc.FileDownlink)
- `fileManager` (Svc.FileManager)
- `prmDb` (Svc.PrmDb, uses `PrmDb.dat`)

### DataProducts (`Svc/Subtopologies/DataProducts/DataProducts.fpp`)
Science/recorded data pipeline. Includes DpManager, DpWriter, DpCatalog.

---

## 7. Design Patterns (Documentation)

All patterns documented in `docs/user-manual/design-patterns/`.

### Application-Manager-Driver (App-Man-Drv)
**File:** `docs/user-manual/design-patterns/app-man-drv.md`

Three-tier layered architecture:
- **Application** — mission logic; orchestrates subsystems
- **Manager** — translates high-level commands to driver messages; generic peripheral interface
- **Driver** — hardware-only; knows the bus, not the device

Enforces no circular dependencies. Enables component reuse and testability.

### Rate Group Pattern
**File:** `docs/user-manual/design-patterns/rate-group.md`

- `Svc::RateGroupDriver` divides a clock interrupt into multiple rates
- `Svc::ActiveRateGroup` (or Passive) executes a list of components each tick
- Passive rate groups execute synchronously; blocking work delays other groups
- Active rate groups run on their own thread; jitter possible but isolated
- **Rate group slip** → `WARNING_HI` event if execution exceeds cycle time

Typical satellite rates: control at 10 Hz, telemetry at 1 Hz, background at 0.1 Hz.

**Code example:** `Ref/Top/instances.fpp` — uses `blockDrv`, `rateGroup1/2/3Comp`

### Health Checking Pattern
**File:** `docs/user-manual/design-patterns/health-checking.md`

- `Svc::Health` sends periodic pings to critical active components
- Components must have async `pingIn` and output `pingOut` ports
- Handler echoes the ping key back immediately
- `WARNING_HI` → first timeout; `FATAL` → second timeout (triggers system reset)
- Configure warning/fatal tick thresholds in topology `PingEntries`
- Non-critical components (e.g., background workers) typically excluded

**Relevant Svc:** `Svc/Health/Health.fpp`

### Manager-Worker Pattern
**File:** `docs/user-manual/design-patterns/manager-worker.md`
**Example code:** `nasa/fprime-examples` → `FlightExamples/ManagerWorker/`

Use when a component needs high-priority responsiveness AND long-running background work:
- **Manager** (high priority, active): receives commands, tracks busy state, delegates via `startWork` port, handles `doneRecv` callback
- **Worker** (low priority, active): only talks to manager; executes background task; signals done via `workDone` port
- Synchronous `cancelWork` port allows immediate cancellation (atomic flag pattern)
- No other component should connect directly to the worker

Ports:
- `manager.startWorker → worker.startWork` (Fw.Signal, async)
- `manager.cancelWorker → worker.cancelWork` (Fw.Signal, sync)
- `worker.workDone → manager.doneRecv` (Fw.CompletionStatus)

**Example files:**
- `FlightExamples/ManagerWorker/Manager/Manager.fpp` — manager component definition
- `FlightExamples/ManagerWorker/Worker/Worker.fpp` — worker component definition
- `FlightExamples/ManagerWorker/Subtopology/ManagerWorker.fpp` — wiring subtopology
- `FlightExamples/ManagerWorker/Manager/docs/sdd.md` — Manager SDD
- `FlightExamples/ManagerWorker/Worker/docs/sdd.md` — Worker SDD
- `FlightExamples/ManagerWorker/Subtopology/docs/sdd.md` — Subtopology SDD

### Subtopologies Pattern
**File:** `docs/user-manual/design-patterns/subtopologies.md`

- Group component instances + connections into a reusable block
- Import with `import <Name>.Subtopology` in FPP topology
- Override configuration by copying config files to your deployment directory
- Available core subtopologies: CdhCore, ComCcsds, DataProducts, FileHandling

### Common Port Patterns
**File:** `docs/user-manual/design-patterns/common-port-patterns.md`

1. **Synchronous Get** — return a pre-computed value with minimal work; name as `get<Value>`
2. **Callback** — separate request and response ports; requester continues while request is processed
3. **Parallel Ports** — port arrays connecting to multiple components; use `match A with B` in FPP for auto-validation
4. **Synchronous Cancel** — sync port sets atomic flag; async handler checks flag periodically

---

## 8. Framework Features

### Data Products (Science Recording)
**File:** `docs/user-manual/framework/data-products.md`

- **Record** = basic unit (struct, fixed array, or variable byte array)
- **Container** = stores records + ID + priority
- Producer components request containers from `DpManager`, fill them, send to `DpWriter`
- `DpCatalog` manages stored containers and priority-ordered downlink
- Four port types: product get (sync), product request (async), product receive, product send
- Ground decoding: `fprime-dp` CLI tool

### Ground Interface
**File:** `docs/user-manual/framework/ground-interface.md`

Two-sided (uplink/downlink), two-layered (framing/driver) architecture:
- **Uplink**: Driver → FrameAccumulator → Deframer → FprimeRouter → CmdDispatcher
- **Downlink**: EventManager/TlmChan → ComQueue → Framer → Driver
- Custom drivers implement `Drv::ByteStreamDriverModel`
- Supports F' native protocol and CCSDS

### State Machines
**File:** `docs/user-manual/framework/state-machines.md`
State machine definitions integrated into FPP; `Fw/Sm/` provides the base framework.

### Multi-Core Support
**File:** `docs/user-manual/framework/run-multi-core.md`

---

## 9. Reference Application (`Ref/`)

The best existing example of a full F' deployment assembly. Study this when writing the satellite topology.

**Key files:**
- `Ref/Top/instances.fpp` — all 18 component instances with types, priorities, queue/stack sizes
- `Ref/Top/topology.fpp` — imports subtopologies, wires rate groups, COM pipeline, data products
- `Ref/Top/RefTopology.cpp` — C++ init/start sequence
- `Ref/Top/RefTopologyDefs.hpp` — topology constants and configuration

**Subtopologies used by Ref:**
- `CdhCore.Subtopology`
- `ComCcsds.Subtopology`
- `FileHandling.Subtopology`
- `DataProducts.Subtopology`

**Additional Ref components:**
- `Ref::BlockDriver` — simulated timer/clock source
- `Ref::SignalGen` — synthetic telemetry signal generator (queued)
- `Ref::PingReceiver` — health check target example (active)
- `Ref::SendBuff` / `Ref::RecvBuff` — buffer passing demo (queued)
- `Ref::DpDemo` — data product demonstration (active)

---

## 10. Framework Core (`Fw/`)

Key framework modules:
| Module | Purpose |
|--------|---------|
| `Fw::Buffer` | Fundamental data buffer type |
| `Fw::Cmd` | Command port types |
| `Fw::Com` | Communication buffer types |
| `Fw::Log` | Event log port types |
| `Fw::Tlm` | Telemetry port types |
| `Fw::Time` | Time representation |
| `Fw::Prm` | Parameter port types |
| `Fw::Dp` | Data product framework types |
| `Fw::Sm` | State machine base framework |
| `Fw::Port` | Port base classes |
| `Fw::Obj` | Component base object |
| `Fw::Comp` | Component base classes |

---

## 11. FPP Language Quick Reference

FPP (F Prime Prime) is the modeling language. `.fpp` files are compiled to C++.

```fpp
# Component definition
active component MyComp {
  # Standard framework ports (always include these)
  command recv port CmdDisp
  command reg port CmdReg
  command resp port CmdStatus
  event port Log
  text event port LogText
  time get port Time
  telemetry port Tlm

  # Custom ports
  async input port startWork: Fw.Signal
  sync input port cancelWork: Fw.Signal
  output port workDone: Fw.CompletionStatus
  output port pingOut: [1] Svc.Ping
  async input port pingIn: [1] Svc.Ping

  # Commands
  async command START opcode 0
  async command STOP opcode 1

  # Events
  event WorkStarted severity activity high format "Work started"

  # Telemetry
  telemetry WorkCount: U32
}

# Topology instance
instance myComp: ManagerWorker.Manager base id 0x1000 \
  queue size Default.QUEUE_SIZE \
  stack size Default.STACK_SIZE \
  priority 80
```

**Matched ports** (for parallel port validation):
```fpp
match pingIn with pingOut
```

---

## 12. Key Documentation URLs

| Topic | URL |
|-------|-----|
| Ports, Components, Topology | `https://raw.githubusercontent.com/nasa/fprime/devel/docs/user-manual/overview/03-port-comp-top.md` |
| Commands, Events, Channels, Params | `https://raw.githubusercontent.com/nasa/fprime/devel/docs/user-manual/overview/04-cmd-evt-chn-prm.md` |
| Rate Group Pattern | `https://raw.githubusercontent.com/nasa/fprime/devel/docs/user-manual/design-patterns/rate-group.md` |
| Health Checking Pattern | `https://raw.githubusercontent.com/nasa/fprime/devel/docs/user-manual/design-patterns/health-checking.md` |
| Manager-Worker Pattern | `https://raw.githubusercontent.com/nasa/fprime/devel/docs/user-manual/design-patterns/manager-worker.md` |
| App-Manager-Driver Pattern | `https://raw.githubusercontent.com/nasa/fprime/devel/docs/user-manual/design-patterns/app-man-drv.md` |
| Subtopologies Pattern | `https://raw.githubusercontent.com/nasa/fprime/devel/docs/user-manual/design-patterns/subtopologies.md` |
| Common Port Patterns | `https://raw.githubusercontent.com/nasa/fprime/devel/docs/user-manual/design-patterns/common-port-patterns.md` |
| Ground Interface | `https://raw.githubusercontent.com/nasa/fprime/devel/docs/user-manual/framework/ground-interface.md` |
| Data Products | `https://raw.githubusercontent.com/nasa/fprime/devel/docs/user-manual/framework/data-products.md` |
| Building Topology | `https://raw.githubusercontent.com/nasa/fprime/devel/docs/user-manual/framework/building-topology.md` |
| CdhCore subtopology FPP | `https://raw.githubusercontent.com/nasa/fprime/devel/Svc/Subtopologies/CdhCore/CdhCore.fpp` |
| ComCcsds subtopology FPP | `https://raw.githubusercontent.com/nasa/fprime/devel/Svc/Subtopologies/ComCcsds/ComCcsds.fpp` |
| FileHandling subtopology FPP | `https://raw.githubusercontent.com/nasa/fprime/devel/Svc/Subtopologies/FileHandling/FileHandling.fpp` |
| DataProducts subtopology FPP | `https://raw.githubusercontent.com/nasa/fprime/devel/Svc/Subtopologies/DataProducts/DataProducts.fpp` |
| Ref instances.fpp | `https://raw.githubusercontent.com/nasa/fprime/devel/Ref/Top/instances.fpp` |
| Ref topology.fpp | `https://raw.githubusercontent.com/nasa/fprime/devel/Ref/Top/topology.fpp` |
| Manager SDD | `https://raw.githubusercontent.com/nasa/fprime-examples/devel/FlightExamples/ManagerWorker/Manager/docs/sdd.md` |
| Worker SDD | `https://raw.githubusercontent.com/nasa/fprime-examples/devel/FlightExamples/ManagerWorker/Worker/docs/sdd.md` |
| Subtopology SDD | `https://raw.githubusercontent.com/nasa/fprime-examples/devel/FlightExamples/ManagerWorker/Subtopology/docs/sdd.md` |
| Manager.fpp | `https://raw.githubusercontent.com/nasa/fprime-examples/devel/FlightExamples/ManagerWorker/Manager/Manager.fpp` |
| Worker.fpp | `https://raw.githubusercontent.com/nasa/fprime-examples/devel/FlightExamples/ManagerWorker/Worker/Worker.fpp` |
| ManagerWorker subtopology.fpp | `https://raw.githubusercontent.com/nasa/fprime-examples/devel/FlightExamples/ManagerWorker/Subtopology/ManagerWorker.fpp` |
| ByteStreamDriverModel SDD | `https://raw.githubusercontent.com/nasa/fprime/devel/Drv/ByteStreamDriverModel/docs/sdd.md` |
| Health.fpp | `https://raw.githubusercontent.com/nasa/fprime/devel/Svc/Health/Health.fpp` |

---

## 13. Guidance for Writing the Satellite SDD

When designing a satellite using F':

1. **Start with subtopologies** — CdhCore covers CDH; ComCcsds covers ground link; FileHandling covers file management; DataProducts covers science recording. Import these rather than assembling from scratch.

2. **Layer components** using App-Manager-Driver: e.g., `ADCS Application → ADCS Manager → IMU Driver`

3. **Use rate groups** for periodic tasks: Attitude control at 10 Hz, telemetry at 1 Hz, background housekeeping at 0.1 Hz

4. **Health check** all critical active components: CmdDispatcher, EventManager, ComQueue, ADCS manager, etc. Exclude background workers.

5. **Use Manager-Worker** for long-running satellite operations: image capture, file processing, calibration routines — keeps high-priority components responsive

6. **Custom components** follow the three-class structure: framework base → autocoded component base → developer implementation class

8. **FPP is the authoritative source** — write `.fpp` models first; C++ is generated from them. Reference `Manager.fpp` and `Worker.fpp` in the ManagerWorker example as templates.

9. **Svc component FPP files** live at `Svc/<ComponentName>/<ComponentName>.fpp` on the devel branch — read these for exact port names and types when wiring connections
