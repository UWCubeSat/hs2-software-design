# Software Development Process

HuskySat-2 flight-software source, deployment configuration, component design documents, and test artifacts are maintained as version-controlled engineering data. The process below applies the same configuration-management discipline to flight software that is used for hardware and system design.

## Configuration Management

The team maintains flight-software and SDD changes in Git repositories. Each change records its author, review history, and associated issue or task. Released deployment configurations identify the source revision, generated artifacts, build environment, target hardware configuration, and test evidence so a result can be reproduced.

Changes that affect commands, telemetry, events, interfaces, or operational behavior will update the applicable SDD page, Command and Telemetry List (CTL), interface documentation, and verification records in the same change set or linked task. The team will retain prior test procedures and as-run results rather than overwriting historical evidence.

## Component Design and Coding Standards

Each mission component will be developed from its SDD entry. The author will first define the component's ports, commands, events, telemetry, parameters, state-machine states, and failure responses in FPP, then update the SDD when the interface or behavior changes. FPP will remain the source of truth for the machine-checked interface; generated `*Ac.hpp` and `*Ac.cpp` files will not be edited by hand. Mission behavior will be implemented in the hand-written component `.hpp` and `.cpp` files and will be covered by the component's tests.

The C++ implementation will follow these standards:

- Names will identify the subsystem and purpose of a function, variable, port, event, or telemetry channel; unexplained abbreviations and magic numeric constants will not be used.
- Comments will explain intent, assumptions, units, state-transition rationale, timing constraints, ownership, and fault handling. Comments will not merely repeat what an obvious line of code does. Non-obvious algorithms, hardware workarounds, and recovery paths will include a short explanation and a reference to the applicable requirement or design decision.
- Every command handler will validate its arguments and current operating state before changing hardware or persistent data. It will return a defined command status and emit the appropriate event for rejection, timeout, retry exhaustion, or other abnormal behavior.
- Every asynchronous or queued path will document buffer ownership, queue behavior, timeout behavior, and what happens when the queue or storage is full. Shared state will have an explicit owner and synchronization strategy.
- Initialization, normal operation, reset/recovery, and shutdown behavior will be visible in the component state machine or design notes. Hardware managers will leave devices in a known safe state after failed initialization or unrecoverable communication errors.

## Branching, Review, and Merge

Each change will use a scoped Git branch tied to a task, issue, or design decision. A pull request will describe the reason for the change, affected components and interfaces, test commands and results, and any remaining limitations. Direct changes to the protected integration branch will not be used for normal development.

Before requesting review, the author will build the affected deployment, run the relevant unit tests, check formatting and generated-code consistency, and update the SDD, FPP, CTL, and requirements traceability when applicable. A reviewer will verify:

- the implementation matches the FPP interface and SDD behavior;
- command validation, event severity, telemetry, timeouts, retries, and fault recovery are defined;
- active-component threading, queue sizes, buffer ownership, and rate-group timing are safe;
- hardware access remains inside the appropriate manager or driver boundary;
- tests cover normal operation, boundary values, invalid inputs, and failure paths; and
- comments, names, documentation, and repository changes are clear enough for another team member to maintain.

Changes to `SatStateMachine`, `Svc::FatalHandler`, storage recovery, command routing, shared services, or cross-subtopology wiring will receive review from both the responsible subsystem owner and the flight-software integration owner. Merge conflicts will be resolved by the author, followed by a fresh build and test run before approval. A pull request will not be merged when required tests fail, the SDD and interface model disagree, or generated files are modified without the source FPP change.

## Test, Release, and Issue Tracking

The verification sequence is component test, GDS deployment test, subsystem test, and integrated-topology test, as defined in the Verification and Test Plan appendix. Component tests will exercise state transitions, command acceptance and rejection, boundary values, invalid sensor or bus data, timeout and retry behavior, queue/storage exhaustion, and fatal or recovery paths where applicable. GDS tests will verify topology wiring, command responses, event and telemetry visibility, rate-group behavior, and deployment startup. Subsystem and integrated tests will use representative hardware and workload configurations.

Test procedures and reports will identify the source revision, FPP/generated-code state, build configuration, hardware or GDS deployment, inputs, pass/fail criteria, observed telemetry and events, anomalies, and as-run result. A release candidate will be tagged or otherwise identified only after the required evidence for its intended test level has been reviewed. Test failures and anomalies will be recorded as issues with an owner, reproduction information, severity, disposition, and regression-test status.

The team will track work in the program schedule and task tracker. Each task will identify an owner, dependencies, planned milestone, and completion criteria. The Schedule document is the authoritative source for milestone dates, deadlines, and review dates; this SDD will refer to that document rather than duplicate dates that can become stale. Milestones will cover component completion, hardware-manager integration, application/subsystem deployment, integrated flight-software deployment, and the test-readiness points needed for the UNP review schedule.

## Roles and Accountability

The following role definitions establish the responsibilities that will be assigned to named team members in the controlled team roster before final CDR submission:

| Role | Responsibility |
|---|---|
| Flight Software Lead | Owns software architecture, integration decisions, review readiness, and overall delivery of flight-software evidence. |
| Subsystem Software Owner | Owns the subsystem application's and managers' requirements, design, implementation, unit tests, and documentation. |
| Flight Software Integration and Test Owner | Owns deployment configuration, GDS/subsystem/integrated test planning, test evidence, and anomaly tracking. |
| Configuration and Release Owner | Owns release identification, build reproducibility, repository hygiene, and configuration records. |
| Ground Interface Owner | Coordinates the CTL, GDS interface, command-response behavior, telemetry/event visibility, and operational test interfaces. |
