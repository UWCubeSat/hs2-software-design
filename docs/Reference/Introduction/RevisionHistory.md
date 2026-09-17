# Revision History and Update Log

This appendix summarizes the SDD-relevant development history of the `hs2-software-design` repository. Merge commits and commit identifiers are omitted; the Git repository remains authoritative for the complete commit record, review history, and file-level changes.

The project history identifies an initial LaTeX version of the SDD authored by **Josh Lando** in **August 2025**.

| Period | Contributors | Grouped SDD-relevant work |
|---|---|---|
| August 2025 | Josh Lando | Initial HuskySat-2 Software Design Document in LaTeX. |
| April 2026 | Josh Lando; Piyush Acharya | Established the repository and initial SDD structure; documented the mission architecture, hardware-manager startup assumptions, operational modes, architecture status, and draft ADCS, Data Collection, and Science Inference managers. |
| May 2026 | Josh Lando; Henry Adams | Added and reorganized EPS, ADCS, thermal, communications, driver, panel-deployment, watchdog, and hardware-manager documentation; added requirements, recovery behavior, and CCSDS topology updates. |
| May 2026 | Ojeet Deol | Began the communications application documentation. |
| July–August 2026 | Senuka Liyanage; Eleanor Marso; Ojeet Deol; Henry Adams | Expanded thermal, GNSS, sun-sensor, EPS, and communications designs; added component SDDs, telemetry and event behavior, state-machine changes, diagrams, and document clean-up. |
| August 2026 | Senuka Liyanage; Eleanor Marso; Ojeet Deol; Henry Adams | Continued component review and refinement, including heater controls, manager naming, communications modes, downlink behavior, requirements, and topology diagrams. |
| September 1–8, 2026 | Eleanor Marso; Ojeet Deol; Mahir Emran | Added the SatStateMachine SDD, updated radio and SDLS documentation, incorporated meeting feedback, and reworked SDD diagrams. |
| September 9, 2026 | Senuka Liyanage; Mahir Emran; Eleanor Marso | Updated MagnetorquerManager, ImmuManager, CameraManager, DataCollectionApplication, GNSS, sun-sensor, and Linux PWM driver documentation; removed the obsolete Linux USB driver page. |
| September 10, 2026 | Senuka Liyanage; Eleanor Marso; Ojeet Deol; Mahir Emran | Reorganized the SDD, resolved naming and diagram conflicts, aligned ADCS and SatStateMachine interfaces, documented communications events, and updated subsystem modes and application behavior. |
| September 11–15, 2026 | Senuka Liyanage; Eleanor Marso; Mahir Emran | Integrated the latest Science Inference, Thermal, Camera, StarTracker, SatStateMachine, and ADCS application updates, including state diagrams, table fixes, Kalman-filter material, and the ADCS topology diagram. |
| September 16, 2026 | Senuka Liyanage; Mahir Emran | Integrated the latest Data Collection, Camera Manager, and Science Application changes from `main`; standardized the component name as `ScienceApplication`, resolved the file-name conflict, and aligned the merged content with the current SDD structure. |

The current PDF build also includes document-structure, formatting, and compilation fixes maintained on the SDD build branch.
