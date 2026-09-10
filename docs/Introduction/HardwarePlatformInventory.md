# Hardware and Platform Inventory

The BeagleBone Black is the on-board Linux flight computer. It runs the F' deployment, hosts the flight-software topology, and accesses spacecraft hardware through Linux-backed F' drivers. Linux provides the operating-system abstractions for UART, I2C, SPI, GPIO, PWM, timing, and file storage; device-specific behavior belongs in the higher-level manager or application that owns the hardware.

| Hardware | Interface | Primary software owner |
|----------|-----------|------------------------|
| Camera 1 and Camera 2 | SPI/I2C | Data collection subsystem |
| Star tracker | UART | Data collection and ADCS subsystems |
| aGNSS receiver | UART | Payload / data collection subsystem |
| IMU | SPI/I2C | ADCS subsystem |
| Sun sensors | I2C/GPIO | ADCS subsystem |
| Magnetorquers | PWM | ADCS subsystem |
| EPS board | I2C/UART | EPS subsystem |
| EnduroSat S-band radio | UART | Communications subsystem |
| External flash | SPI | File and data-product services |
| Temperature sensors | I2C | Thermal subsystem |
| Heaters | PWM | Thermal subsystem |

The BeagleBone Black and Linux are platform services rather than mission components. The driver layer exposes their operating-system interfaces; hardware managers interpret device registers and protocols; applications coordinate subsystem behavior.
