# F' OSAL Queue

F' OSAL queue services provide the message-passing primitive used by active and queued components. On Linux, the implementation supplies the synchronization needed to enqueue, block, and dequeue work safely. The queue separates command arrival from component execution and helps prevent long-running work from blocking synchronous callers.

Reference: [F' OSAL source](https://github.com/nasa/fprime/tree/devel/Os)
