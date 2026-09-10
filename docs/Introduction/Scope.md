# Scope

The scope of this Software Design Document is the HuskySat-2 flight-software repository and the deployment architecture required to run it on the BeagleBone Black flight computer.

The document covers:

- software interfaces, ports, commands, events, telemetry, parameters, and state machines for components represented in this repository;
- Linux-backed Level 1 drivers and the Level 2 hardware managers that own device protocols and recovery behavior;
- Level 3 subsystem applications, ADCS algorithms, and the F' framework components used by the application topology;
- the intended decomposition of subsystem topologies and their cross-layer connections;
- integration of the LOST, FOUND, and SCOPE mission libraries;
- the staged verification path from unit tests through GDS, subsystem, and integrated deployment testing.

The detailed design for `SatStateMachine`, camera managers, star-tracker management, radiation tolerance, and storage/data-retention policy will be added when those designs are available. `HardwareResetManager` remains a future system-infrastructure component and is not classified as part of the Level 4 mission-orchestration layer.
