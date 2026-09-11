# Os::File

`Os::File` provides the F' OSAL interface for opening, reading, writing, seeking, flushing, and closing files. On the BeagleBone Black, the implementation maps these operations to Linux filesystem calls. HuskySat-2 uses this boundary for configuration, stored data, data products, and file-based communications services; file policy and data ownership remain with the application or service that uses the file.

Reference: [F' OSAL source](https://github.com/nasa/fprime/tree/devel/Os)
