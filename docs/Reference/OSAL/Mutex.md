# F' OSAL Mutex

F' OSAL mutex services provide portable mutual exclusion for shared state. The Linux implementation maps to the platform's synchronization primitives. Components should keep protected regions short and use F' guarded ports or component-owned synchronization where that makes the ownership of shared state clearer.

Reference: [F' OSAL source](https://github.com/nasa/fprime/tree/devel/Os)
