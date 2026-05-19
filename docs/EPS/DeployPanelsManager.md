# DeployPanelsManager SDD

## 1. Overview

`DeployPanelsManager` is a Layer 2 Active component that executes the solar panel deployment sequence when commanded by `EPSApplication`. It maintains a two-state machine tracking whether deployment has previously occurred. In both states the burn wire sequence executes; in the `DEPLOYED` state an additional `WARNING_HI` event is emitted to notify the operator that this is a re-attempt. There is no automatic retry — each deployment attempt requires an explicit `DEPLOY_PANELS` command from ground via `EPSApplication`.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-DPM-001 | `DeployPanelsManager` shall activate the burn wire sequence upon receipt of a deploy command originating from ground. | Test |
| HS2-DPM-002 | `DeployPanelsManager` shall emit events upon beginning and ending the burn sequence. | Test |

---

## 3. Design

### 3.1 Component Type

Active component with internal flat F' state machine (`Fw::Sm`).

### 3.2 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `deploy` | Input | `Fw.Signal` | Deploy command forwarded from EPSApplication |
| `burnWire` | Output | `Drv.GpioWrite` | Burn wire GPIO; asserted for deployment duration then deasserted |
| `logOut` | Output | `Fw.Log` | Event logging |

### 3.3 Commands

None. `DeployPanelsManager` is driven entirely by the `deploy` port call from `EPSApplication`, which itself accepts the ground `DEPLOY_PANELS` command.

---

## 4. State Machine

`DeployPanelsManager` uses a flat two-state F' state machine:

```
NOT_DEPLOYED
  └─ (on deploy signal) emit deployment-initiated event
                        assert burnWire GPIO high
                        wait burn wire duration constant
                        deassert burnWire GPIO
                        emit deployment-complete event
                        transition → DEPLOYED

DEPLOYED
  └─ (on deploy signal) emit WARNING_HI "re-attempt: panels previously deployed" event
                        assert burnWire GPIO high
                        wait burn wire duration constant
                        deassert burnWire GPIO
                        emit deployment-complete event
                        (remain in DEPLOYED)
```

---

## 5. Notes

- Burn wire active duration is a hardcoded constant in the component. Value TBD pending hardware team specification. This MAY eventually become a parameter stored in `PrmDb`, but we see no compelling reason for this right now.
- Burn wire duration is timed via `Os::Task::delay()` called from the deploy handler on the Active component's thread. The handler asserts the burn wire GPIO, calls `Os::Task::delay(BURN_DURATION)`, then deasserts the GPIO. No `schedIn` or timer port is required — `Os::Task::delay()` is a direct call into F''s OS abstraction layer and blocks only the component's own thread, leaving all other components unaffected. Tradeoff: the burn cannot be cancelled mid-sequence.
- NOTE: If it turns out that our deployment has performance issues caused by having too many active threads, we can make `DeployPanelsManager` into a Queued/Passive component with a rate group for timing the burn duration.
- The `DEPLOYED` state re-executes the burn sequence to support the case where the panel did not fully deploy on the first attempt. The `WARNING_HI` event alerts the operator that this is a re-attempt.
