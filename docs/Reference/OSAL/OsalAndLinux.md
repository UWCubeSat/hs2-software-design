# OSAL

## F' OSAL and Linux on the BeagleBone Black

F' uses an operating-system abstraction layer (OSAL) so framework code can request common services without embedding a particular operating system into every component. The OSAL provides wrappers for tasks, queues, mutexes, time, files, directories, and related operating-system resources. The Level 1 driver boundary similarly provides typed F' interfaces for Linux serial and bus access.

On the BeagleBone Black, these abstractions are implemented with Linux and POSIX facilities. Flight components therefore use F' and OSAL interfaces for scheduling, synchronization, timing, and file access, while Linux-specific details remain in the driver or deployment layer. This keeps the application and hardware-manager code testable in GDS and on a host system, while still allowing the deployed topology to use the BeagleBone's UART, I2C, SPI, GPIO, PWM, and filesystem resources.

The OSAL is not a substitute for a device manager. It can open a file or serial endpoint and provide a task or queue, but a Level 2 manager still owns the device protocol, register meanings, initialization sequence, and hardware-specific recovery policy.

## Os::File

`Os::File` provides the F' OSAL interface for opening, reading, writing, seeking, flushing, and closing files. On the BeagleBone Black, the implementation maps these operations to Linux filesystem calls. HuskySat-2 uses this boundary for configuration, stored data, data products, and file-based communications services; file policy and data ownership remain with the application or service that uses the file.

Reference: [F' OSAL source](https://github.com/nasa/fprime/tree/devel/Os)

## Os::Task

`Os::Task` creates and controls execution contexts through a portable F' OSAL interface. The Linux implementation supplies the underlying POSIX thread behavior. HuskySat-2 active components and service components use this abstraction so deployment-specific priorities, stack sizes, and startup behavior remain in the topology configuration.

Reference: [F' OSAL source](https://github.com/nasa/fprime/tree/devel/Os)

## Os::Queue

`Os::Queue` provides the message-passing primitive used by active and queued components. On Linux, the implementation supplies the synchronization needed to enqueue, block, and dequeue work safely. The queue separates command arrival from component execution and helps prevent long-running work from blocking synchronous callers.

Reference: [F' OSAL source](https://github.com/nasa/fprime/tree/devel/Os)

## Os::Mutex

`Os::Mutex` provides portable mutual exclusion for shared state. The Linux implementation maps to the platform's synchronization primitives. Components will keep protected regions short and use F' guarded ports or component-owned synchronization where that makes the ownership of shared state clearer.

Reference: [F' OSAL source](https://github.com/nasa/fprime/tree/devel/Os)

## Os::Time

`Os::Time` provides portable access to clocks, timestamps, and time-related operations. On the BeagleBone Black, the implementation uses Linux clocks while preserving F' time types at component interfaces. This supports telemetry timestamps, scheduling, timeout handling, and tests that substitute controlled time sources.

Reference: [F' OSAL source](https://github.com/nasa/fprime/tree/devel/Os)
