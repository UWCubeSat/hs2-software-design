# Hardware and Platform Inventory

The BeagleBone Black is the on-board Linux flight computer. It runs the F' deployment, hosts the spacecraft software topology, and accesses spacecraft hardware through Linux-backed F' drivers. Linux provides the operating-system abstractions for UART, I2C, SPI, GPIO, PWM, timing, USB, and file storage; device-specific behavior belongs in the higher-level manager or application that owns the hardware.

The inventory below is derived from the current hardware-manager interfaces.

| Hardware or device | Interface to BeagleBone Black | Owning software |
|--------------------|-------------------------------|-----------------|
| LOST camera | USB | `DataCollectionApplication` |
| FOUND camera | USB | `DataCollectionApplication` |
| Arcsec Sagitta Star Tracker | UART; `StarTrackerManager` | ADCS and data collection |
| SkyFox piNAV-NG GNSS receiver | UART; reset GPIO; VPP/PPS GPIO input | `GnssManager` |
| VectorNav VN-100 IMU/AHRS | RS-232/UART | `ImmuManager` |
| Six sun photodiodes and MCP3208 ADC | SPI; chip-select GPIO | `SunSensorManager` |
| Two torque rods and one air coil | PWM duty cycle and direction outputs | `MagnetorquerManager` |
| INA3221 three-channel current monitor | I2C | `CurrentSensorManager` |
| BQ25756 MPPT/battery charger | I2C | `MpptManager` |
| Solar-panel deployment burn wire | GPIO | `DeployPanelsManager` |
| EPS hardware watchdog | GPIO pulse | `WatchdogPinger` |
| EnduroSat S-band transceiver | UART | `TmtcRadioManager` |
| Temperature sensors | SPI; sensor-select GPIO; DRDY input | `TemperatureSensorManager` |
| Heaters | PWM | `HeaterManager` |
| microSD card and external SSD | Linux filesystem services | F' file-management and data-product services; mission applications |

The BeagleBone Black and Linux are platform services rather than mission components. The driver layer exposes their operating-system interfaces; hardware managers interpret device registers and protocols; applications coordinate subsystem behavior.
