# Software Development Process

HuskySat-2 flight-software source, deployment configuration, component design documents, and test artifacts are maintained as version-controlled engineering data. The process below applies the same configuration-management discipline to flight software that is used for hardware and system design.

## Configuration Management

The team maintains flight-software and SDD changes in Git repositories. Each change records its author, review history, and associated issue or task. Released deployment configurations identify the source revision, generated artifacts, build environment, target hardware configuration, and test evidence so a result can be reproduced.

Changes that affect commands, telemetry, events, interfaces, or operational behavior will update the applicable SDD page, Command and Telemetry List (CTL), interface documentation, and verification records in the same change set or linked task. The team will retain prior test procedures and as-run results rather than overwriting historical evidence.

## Code Review and Integration

Development work will be performed in scoped branches or equivalent isolated changes. Before integration, the author will build the affected deployment, run the relevant unit or deployment tests, update documentation and requirements traceability, and request peer review. Reviewers will verify interface compatibility, command and fault behavior, resource implications, test coverage, and readability before the change is merged.

Code will use clear names, concise comments where intent is not evident from the implementation, and interfaces that make ownership and error handling explicit. Changes that affect safety, storage recovery, commanding, or shared topology wiring will receive focused review from the responsible subsystem and flight-software integration owners.

## Test, Release, and Issue Tracking

The verification sequence is component test, GDS deployment test, subsystem test, and integrated-topology test, as defined in the Verification and Test Plan appendix. Test procedures and reports will state their configuration, pass/fail criteria, observed telemetry and events, anomalies, and the source revision tested. A release candidate will be tagged or otherwise identified only after the required evidence for its intended test level has been reviewed.

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

The named assignment matrix will identify one accountable person for every role and subsystem, with backups documented for critical integration and operations responsibilities.
