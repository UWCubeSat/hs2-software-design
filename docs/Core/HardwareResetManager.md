# HardwareResetManager SDD

## 1. Overview

`HardwareResetManager` is a future **Layer 5** component intended to live alongside the `CdhCore` subtopology in the system infrastructure layer. It is not currently wired into any subtopology.

Its future role is to receive fault notifications from a general-purpose `FaultManager` (also Layer 5, not yet designed) and perform targeted hardware-level resets of individual components or power rails — providing a graduated recovery path below the full system reset that `fatalHandler` performs. It will also notify `SatStateMachine` of the fault so that mode-level responses can be coordinated.

---

## 2. Requirements

TODO — to be defined when Layer 5 / FaultManager design is undertaken.

---

## 3. Design

Design deferred. Component is reserved for the Layer 5 / CdhCore integration work.

**Anticipated interface (subject to change):**

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `hardwareFaultIn` | Input | Custom port (component/rail ID) | Called by FaultManager after threshold breach |
| `faultNotifyOut` | Output | Custom port (component/rail ID) | Notifies `SatStateMachine` of the hardware fault |
| `resetOut[N]` | Output | TBD pending hardware confirmation — port type is set when the component is implemented and can be updated in the FPP definition and topology wiring at that time | One per monitored component; asserted to perform hardware reset |
| `logOut` | Output | `Fw.Log` | Event logging |

---

## 4. Notes

- `HardwareResetManager` is intentionally excluded from all current subtopologies.
- The Layer 5 design (including `FaultManager` and `HardwareResetManager`) is a future work item. The scope includes general-purpose fault handling and targeted hardware reset capabilities.
- Whether `resetOut[N]` GPIO ports are present depends on hardware team confirmation of which rails have software-controllable reset pins.
