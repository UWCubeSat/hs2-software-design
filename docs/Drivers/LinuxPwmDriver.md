# LinuxPwmDriver SDD

## 1. Overview

`LinuxPwmDriver` is a Layer 1 **Passive** bus driver for a single Linux hardware PWM channel. It provides a typed F' port interface over the Linux sysfs PWM subsystem (`/sys/class/pwm/`), allowing higher-level components to set the period, duty cycle, and enable state of a hardware PWM output without any direct kernel interaction. Current consumers include both `MagnetorquerManager` and `HeaterManager`.

`LinuxPwmDriver` is responsible only for period, duty cycle, and channel enable.

`LinuxPwmDriver` has no state machine, no parameters, no commands, and no health monitoring. It is a thin synchronous wrapper over sysfs file writes, executing on the invoking thread.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-PWM-001 | LinuxPwmDriver shall export and open the configured PWM channel via the Linux sysfs interface on initialization | Inspection |
| HS2-PWM-002 | LinuxPwmDriver shall emit a WARNING_HI event and return `PWM_OPEN_ERR` if the sysfs channel cannot be exported or opened during initialization | Inspection |
| HS2-PWM-003 | LinuxPwmDriver shall set the period of the PWM channel when `pwmSetPeriod` is called and return `PWM_OK` on success | Inspection |
| HS2-PWM-004 | LinuxPwmDriver shall set the duty cycle of the PWM channel when `pwmSetDutyCycle` is called and return `PWM_OK` on success | Inspection |
| HS2-PWM-005 | LinuxPwmDriver shall reject a duty cycle greater than the current period and return `PWM_INVALID_PARAM` without writing to sysfs | Inspection |
| HS2-PWM-006 | LinuxPwmDriver shall enable or disable the PWM channel when `pwmEnable` is called and return `PWM_OK` on success | Inspection |
| HS2-PWM-007 | LinuxPwmDriver shall return `PWM_WRITE_ERR` on any sysfs write failure during operation | Inspection |
| HS2-PWM-008 | LinuxPwmDriver shall return `PWM_NOT_OPENED` for any port call made before the channel has been successfully initialized | Inspection |

---

## 3. Design

### 3.1 Component Type

Passive component. No message queue, no dedicated thread. All port handlers execute synchronously on the calling thread. This is consistent with all other Layer 1 bus drivers (`LinuxGpioDriver`, `LinuxI2cDriver`, `LinuxSpiDriver`, `LinuxUartDriver`).

### 3.2 Port Types

`LinuxPwmDriver` introduces three new port types and one status enum, all declared under `module Drv`. These are defined in the project's own source tree — the `module Drv` FPP namespace is open and can be extended from any `.fpp` file without modifying the `nasa/fprime` submodule.

```fpp
module Drv {

    enum PwmStatus {
        PWM_OK            @< Operation succeeded
        PWM_NOT_OPENED    @< Channel not yet exported or initialized
        PWM_WRITE_ERR     @< sysfs write to period, duty_cycle, or enable failed
        PWM_OPEN_ERR      @< Failed to export or open the PWM channel at init
        PWM_INVALID_PARAM @< duty_cycle > period or other illegal argument
        PWM_OTHER_ERR     @< Unknown error
    }

    @ Set the period of this PWM channel in nanoseconds
    port PwmSetPeriod(
        periodNs: U32   @< Written to /sys/class/pwm/pwmchipN/pwmN/period
    ) -> Drv.PwmStatus

    @ Set the duty cycle of this PWM channel in nanoseconds
    port PwmSetDutyCycle(
        dutyCycleNs: U32   @< Written to .../duty_cycle; must be <= current period
    ) -> Drv.PwmStatus

    @ Enable or disable this PWM channel
    port PwmEnable(
        $state: Fw.Logic   @< HIGH = enable, LOW = disable; written to .../enable
    ) -> Drv.PwmStatus

}
```

A companion `Drv::Pwm` interface groups the three ports for import into the component definition:

```fpp
module Drv {
    interface Pwm {
        sync input port pwmSetPeriod:    Drv.PwmSetPeriod
        sync input port pwmSetDutyCycle: Drv.PwmSetDutyCycle
        sync input port pwmEnable:       Drv.PwmEnable
    }
}
```

### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `pwmSetPeriod` | Input (sync) | `Drv.PwmSetPeriod` | Set the channel period in nanoseconds |
| `pwmSetDutyCycle` | Input (sync) | `Drv.PwmSetDutyCycle` | Set the channel duty cycle in nanoseconds |
| `pwmEnable` | Input (sync) | `Drv.PwmEnable` | Enable or disable the channel |
| `logOut` | Output | `Fw.Log` | Event logging (open errors, write errors, diagnostic init confirmation) |

### 3.4 Initialization

The PWM chip number and channel index are fixed at topology assembly time via a C++ `open(chipNum, channelNum)` call in the topology setup function, before any port calls are made. This follows the same convention as `LinuxGpioDriver::open(chip, pin)`. No runtime configuration ports are needed.

On a successful `open()` call, the driver exports the channel via `/sys/class/pwm/pwmchipN/export`, opens the resulting sysfs entries, and emits a `DIAGNOSTIC` event confirming initialization. On failure it emits `WARNING_HI` and marks the channel as unopened — subsequent port calls return `PWM_NOT_OPENED`.

**Events:**

| Event | Severity | Description |
|-------|----------|-------------|
| `ChannelOpened(chip: U32, channel: U32)` | DIAGNOSTIC | Emitted on successful sysfs export and open |
| `OpenChannelError(chip: U32, channel: U32, status: Os.FileStatus)` | WARNING_HI | Emitted if export or open fails during initialization |

---

## 4. Notes

- `LinuxPwmDriver` is not a pre-built F' component — no `LinuxPwmDriver` exists in `nasa/fprime` or `fprime-community`. It must be implemented as a custom project component.
- The Linux sysfs PWM interface requires that `period` be set before `duty_cycle`. `MagnetorquerManager` and other components doing PWM are responsible for this ordering. The driver enforces the constraint defensively via the `PWM_INVALID_PARAM` return on violation (HS2-PWM-005).
- `LinuxPwmDriver` is **excluded from health monitoring** (`Svc::Health`).
- The `Drv::Pwm` interface `.fpp` file and the `Drv::PwmStatus` port type `.fpp` file should live alongside `LinuxPwmDriver` in the project source tree (e.g., `Components/LinuxPwmDriver/`).
