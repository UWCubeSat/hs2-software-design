# Verification and Test Plan

Verification proceeds outward from deterministic component behavior toward hardware and system behavior. Every component has a test scope appropriate to its type, interfaces, state machine, and failure modes.

## Component-Level Tests

Unit tests cover algorithms, port handlers, state transitions, parameter handling, command validation, telemetry publication, event severity, and error paths. Passive components will be tested synchronously with controlled inputs. Active and queued components will be tested for queue behavior, scheduling assumptions, and correct responses to health pings and asynchronous completion.

Driver tests will verify configuration, open/close behavior, valid and invalid bus operations, operating-system error translation, and safe behavior after a failed transaction. Hardware managers will be tested with simulated driver responses for initialization, register transactions, timeouts, retries, invalid data, recovery, and escalation to application-visible faults.

## GDS Deployment Tests

Hardware managers and Linux drivers will be exercised in F' Ground Data System (GDS) deployments before flight hardware is available or integrated. GDS tests will send commands, inspect telemetry and events, verify parameter loading, and exercise the same typed ports used in the flight topology.

The GDS environment provides representative device interfaces or deterministic simulators for UART, I2C, SPI, GPIO, and PWM behavior. Tests include nominal responses, delayed responses, malformed data, bus failures, and device reset conditions. This allows recovery behavior to be verified without depending on nondeterministic hardware timing.

## Subsystem Tests

Each subsystem will then be tested in an increasingly complete deployment:

1. connect the application to simulated managers and verify application state machines and algorithm selection;
2. connect managers to simulated or representative drivers and verify device-level data flow;
3. enable rate groups, health monitoring, commands, events, and telemetry together;
4. verify subsystem recovery behavior and safe outputs after manager or driver faults; and
5. verify the subsystem's communications, data-product, and storage interactions where applicable.

Examples include checking that ADCS algorithms produce safe actuator commands from known sensor inputs, that communications mode changes alter telemetry packet behavior, that EPS managers preserve safe power states after failures, and that thermal control does not enable a heater on invalid sensor data.

## Integrated Topology Tests

Integrated tests will exercise cross-subsystem data flow, shared CDH services, CCSDS uplink/downlink paths, file operations, data products, rate-group scheduling, health monitoring, and the `SatStateMachine` mode transitions. Fatal-event routing and the application-owned targeted-reset path will be added to the integrated fault-response tests as those interfaces are finalized.

## Verification Evidence

Each requirement will identify its verification method, such as inspection, unit test, GDS deployment test, subsystem test, or integrated test. Test results will retain the command sequence, configuration, input data, telemetry/events observed, expected result, actual result, and any hardware or simulator version needed to reproduce the run.

## Related Verification Artifacts

This SDD defines the verification strategy and the component-level verification methods shown in the requirements tables. The following controlled documents provide the detailed operational and test evidence rather than duplicating it here:

| Document | Relationship to this SDD |
|---|---|
| Command and Telemetry List (CTL) | Defines the operator-visible commands, telemetry channels, events, and command-response expectations used by the ground interface. |
| Ground Data System (GDS) document | Defines the ground hardware, ground software, and operator workflows used to send commands and observe flight responses during test and operations. |
| Requirements Verification Matrix (RVM) | Traces mission and system requirements to their verification methods, procedures, evidence, and closure status. |
| Unit, GDS, subsystem, and integrated test procedures and reports | Define execution steps, pass/fail criteria, configurations, as-run results, and anomaly records for the verification levels described in this SDD. |
| Data budget | Defines telemetry, event, image, storage, and downlink volume assumptions used to set the test configurations and acceptance criteria. |

The source repository and test records will retain versioned procedures and results. Each verification record will identify the applicable SDD component and requirement IDs so that design, tests, and requirements remain traceable.
