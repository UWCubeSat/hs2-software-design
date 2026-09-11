# Os::Task

`Os::Task` creates and controls execution contexts through a portable F' OSAL interface. The Linux implementation supplies the underlying POSIX thread behavior. HuskySat-2 active components and service components use this abstraction so deployment-specific priorities, stack sizes, and startup behavior remain in the topology configuration.

Reference: [F' OSAL source](https://github.com/nasa/fprime/tree/devel/Os)
