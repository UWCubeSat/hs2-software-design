# LinuxPwmDriver SDD

## 1. Overview

`LinuxPwmDriver` is a Layer 1 passive driver for one Linux PWM channel. It gives a hardware manager (Layer 2 component) a synchronous F' interface for setting the channel period and duty cycle and for enabling or disabling the output. It contains no device or actuator logic.

It uses the Linux PWM sysfs interface (`/sys/class/pwm/pwmchipN/pwmM/`). One driver instance owns one channel. 

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-PWM-001 | LinuxPwmDriver shall open and own one configured PWM channel during topology setup. | Inspection |
| HS2-PWM-002 | LinuxPwmDriver shall provide synchronous operations to set the channel period, set the duty cycle, and enable or disable the channel. | Inspection |
| HS2-PWM-003 | LinuxPwmDriver shall reject a duty cycle greater than the most recently applied period without writing the duty-cycle value. | Inspection |
| HS2-PWM-004 | LinuxPwmDriver shall report an open failure, an invalid request, or a runtime write failure to its caller. | Inspection |
| HS2-PWM-005 | LinuxPwmDriver shall report open and runtime failures through F' events. | Inspection |

---

## 3. Design

### 3.1 Component Type

Passive component. Its port handlers run synchronously on the calling thread. It has no state machine, commands, parameters, telemetry channels, or health-monitoring interface. The driver retains only its open state and the last successfully applied period.

### 3.2 Ports

The following project-defined ports form the `Drv.Pwm` interface.

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `pwmSetPeriod` | Input (sync) | `Drv.PwmSetPeriod` | Set the period in nanoseconds. |
| `pwmSetDutyCycle` | Input (sync) | `Drv.PwmSetDutyCycle` | Set the duty cycle in nanoseconds. |
| `pwmEnable` | Input (sync) | `Drv.PwmEnable` | Enable or disable the channel. |
| `logOut` | Output | `Fw.Log` | Report open and write failures. |

Each operation returns `Drv.PwmStatus`.

| Status | Meaning |
|--------|---------|
| `PWM_OK` | Operation succeeded. |
| `PWM_NOT_OPENED` | The channel was not successfully opened. |
| `PWM_OPEN_ERR` | Exporting or opening the channel failed. |
| `PWM_WRITE_ERR` | A write to the PWM interface failed. |
| `PWM_INVALID_PARAM` | The requested duty cycle exceeds the configured period. |
| `PWM_OTHER_ERR` | An unexpected driver error occurred. |

### 3.3 Configuration and Operation

Topology setup calls `open(chipNum, channelNum)` before the connected manager makes any port calls. The driver exports the selected channel when necessary and opens its `period`, `duty_cycle`, and `enable` entries. If setup fails, port calls return `PWM_NOT_OPENED`.

A manager configures a channel in this order:

1. Set the period.
2. Set a duty cycle no greater than that period.
3. Enable the channel.

The manager owns actuator-safe behavior. For example, duty cycle should be set to 0 before enabling a channel or when dealing with an error.
