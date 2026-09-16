# F' System Services

These F' components provide deployment-wide telemetry, health, scheduling, and version services. Their individual descriptions are short, so they are grouped here; `Svc::CmdDispatcher` and `Svc::FatalHandler` are documented in dedicated chapters because they contain additional mission-specific design detail.

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
