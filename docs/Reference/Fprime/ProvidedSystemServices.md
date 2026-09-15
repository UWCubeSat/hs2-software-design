# F' System Services

These F' components provide deployment-wide command, telemetry, health, scheduling, and version services. They are grouped here because their individual descriptions are short; `Svc::FatalHandler` remains on its own page because it contains the mission-specific fatal-recovery design.

## Svc::CmdDispatcher

**Type:** Active component.

`Svc::CmdDispatcher` decodes incoming command buffers and dispatches registered commands to the components that implement them. Ground and sequenced commands enter the topology through this shared command-and-data-handling path.

### Command Dispatch, Validation, and Responses {.unnumbered .unlisted}

At deployment startup, every command-capable component registers its opcodes with `Svc::CmdDispatcher`. The dispatcher maps each registered opcode to the output port connected to its implementing component. When an uplinked or sequenced command arrives, the dispatcher decodes the command buffer, checks that the opcode is registered, assigns a command sequence number, records the command context in its pending-command tracker, and sends the command to the registered component.

`Svc::CmdDispatcher` rejects a malformed command buffer, an unregistered opcode, or a command that cannot be tracked because the pending-command table is full. It logs the resulting condition and returns the applicable status to the command source through the corresponding response path. A full dispatcher input queue is handled by dropping the new buffer and recording an overflow condition rather than blocking the command path. In the HuskySat-2 flight topology, the ground-uplink path is configured as a command source, so the command result is returned through the ground command-response path for downlink and operator visibility.

The component that implements a command owns semantic validation, such as argument ranges, hardware readiness, and whether the requested action is legal in its current operational mode. It reports completion or rejection to `Svc::CmdDispatcher`, which correlates that response with the original command context and returns it to the source. This separates common transport and routing validation from component-specific safety decisions.

### Relationship to SatStateMachine {.unnumbered .unlisted}

Commands do not pass through `SatStateMachine` by default. Routing every command through the state machine would duplicate F' opcode registration, response tracking, and component-level validation. `SatStateMachine` is the owner of satellite-wide operating-mode commands, so those commands are dispatched directly to its registered handlers. Other commands are dispatched directly to the application, manager, or service that owns the action; that handler checks its own state and interlocks before acting. This preserves F's command architecture while keeping satellite-wide mode authority in `SatStateMachine`.

[CmdDispatcher software design document](https://fprime.jpl.nasa.gov/latest/Svc/CmdDispatcher/docs/sdd/)

## Svc::EventManager

**Type:** Active component.

`Svc::EventManager` collects component events, applies event filtering, and packages events for downlink. Applications and hardware managers publish diagnostic, warning, and fatal events to this shared path.

[EventManager software design document](https://fprime.jpl.nasa.gov/latest/Svc/EventManager/docs/sdd/)

## Svc::TlmChan and Svc::TlmPacketizer

**Type:** Active components.

`Svc::TlmChan` stores the most recent telemetry-channel values for collection. `Svc::TlmPacketizer` groups telemetry channels into packets according to configured definitions and rate logic. `ComApplication` changes packet rate logic as the spacecraft communication mode changes.

[TlmChan software design document](https://fprime.jpl.nasa.gov/latest/Svc/TlmChan/docs/sdd/)  
[TlmPacketizer source and interface](https://github.com/nasa/fprime/tree/devel/Svc/TlmPacketizer)

## Svc::Health

**Type:** Queued component.

`Svc::Health` pings active components, detects missed responses, and provides the framework health-monitoring path. Critical applications and hardware managers register their ping ports with the shared service.

[Health software design document](https://fprime.jpl.nasa.gov/latest/Svc/Health/docs/sdd/)

## Svc::RateGroupDriver and Svc::ActiveRateGroup

**Type:** Passive and active scheduling components.

`Svc::RateGroupDriver` divides the primary system tick into scheduled rate-group signals. `Svc::ActiveRateGroup` invokes configured components at those rates so control loops, telemetry collection, and background work can use separate schedules.

[RateGroupDriver software design document](https://fprime.jpl.nasa.gov/latest/Svc/RateGroupDriver/docs/sdd/)  
[ActiveRateGroup source and interface](https://github.com/nasa/fprime/tree/devel/Svc/ActiveRateGroup)

## Svc::Version

**Type:** Passive component.

`Svc::Version` reports flight-software version and build information through the CDH telemetry path for checkout, GDS testing, and flight operations.

[Version software design document](https://fprime.jpl.nasa.gov/latest/Svc/Version/docs/sdd/)

## Svc::AssertFatalAdapter

**Type:** Passive component.

`Svc::AssertFatalAdapter` converts framework assertions into FATAL events for the CDH fault-handling path. Reset and recovery policy remains a system-level deployment decision.

[AssertFatalAdapter software design document](https://fprime.jpl.nasa.gov/latest/Svc/AssertFatalAdapter/docs/sdd/)
