# Os::Time

`Os::Time` provides portable access to clocks, timestamps, and time-related operations. On the BeagleBone Black, the implementation uses Linux clocks while preserving F' time types at component interfaces. This supports telemetry timestamps, scheduling, timeout handling, and tests that substitute controlled time sources.

Reference: [F' OSAL source](https://github.com/nasa/fprime/tree/devel/Os)
