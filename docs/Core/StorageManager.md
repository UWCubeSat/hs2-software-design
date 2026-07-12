# StorageManager SDD

## 1. Overview

`StorageManager` is a Layer 2 Active (worker) component that manages data storage across the flight computer's memory tiers. It receives data storage requests from CDH subsystems (Comms, EPS, Payload, ADCS, FlightSW) with data type information, performs data preparation (CRC, Hamming codes, categorization), manages wear leveling, and routes the data to the appropriate memory tier via the F' OSAL layer. It implements a flat F' state machine following the `RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN` pattern, driven by rate group ticks, with self-healing on error.

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-SM-001 | StorageManager shall accept data storage requests from CDH subsystems with associated data type metadata. | Inspection |
| HS2-SM-002 | StorageManager shall compute and append CRC checksums to all data blocks before storage. | Analysis |
| HS2-SM-003 | StorageManager shall apply Hamming codes for error correction to critical data blocks as specified by data type. | Analysis |
| HS2-SM-004 | StorageManager shall categorize incoming data by type and route it to the appropriate memory tier (T0-T4) based on persistence and criticality requirements. | Inspection |
| HS2-SM-005 | StorageManager shall implement wear leveling algorithms for erasable memory technologies (NOR Flash, eMMC, microSD). | Analysis |
| HS2-SM-006 | StorageManager shall detect and recover from storage errors via self-healing, transitioning to RESET state on error and attempting re-initialization. | Test |
| HS2-SM-007 | StorageManager shall provide storage availability status to requesting components upon query. | Inspection |
| HS2-SM-008 | StorageManager shall operate without satellite mode awareness, responding only to rate group ticks and direct commands. | Inspection |

## 3. Design

### 3.1 Component Type

Active (worker) component with a flat F' state machine (`Fw::Sm`) following the `RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN` pattern. The component is driven by a rate group scheduler port (`schedIn`) and does not have a mode port, as it is unaware of satellite operational modes.

### 3.2 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | Rate group tick for state machine driving |
| `storeDataIn[5]` | Input | `Fw.Dp` | Data storage requests from CDH subsystems (indexed by subsystem: 0=Comms, 1=EPS, 2=Payload, 3=ADCS, 4=FlightSW) |
| `dataTypeIn[5]` | Input | `Fw.PrmGet` | Parameter specifying data type for corresponding storeDataIn port (used to determine handling) |
| `storeDataOut` | Output | `Fw.Dp` | Processed data block ready for OSAL/storage tier transmission |
| `storeStatusOut` | Output | `Fw.Tlm` | Telemetry indicating storage operation status (success, error, wear level) |
| `storeErrorOut` | Output | `Fw.Log` | Error logging for storage failures |
| `pingIn` / `pingOut` | In/Out | `Svc.Ping` | Health monitoring |
| `memGetIn[4]` | Input | `Fw.PrmGet` | Get memory tier status/requests (T1-T4) |
| `memSetOut[4]` | Output | `Fw.PrmSet` | Configuration commands to memory tier interfaces |

### 3.3 Data Handling Flow

1. **Request Reception**: CDH subsystem sends data via `storeDataIn[i]` along with data type via `dataTypeIn[i]`.
2. **Categorization**: StorageManager examines data type to determine:
   - Required persistence (survive reboot?)
   - Criticality level
   - Data bulkiness
   - Regeneration cost
3. **Processing**:
   - Compute CRC-32 and append to data
   - Apply Hamming(7,4) or similar ECC if data type requires error correction
   - Add header with metadata (timestamp, source subsystem, data type, sequence number)
4. **Tier Assignment**: Based on categorization, select target memory tier:
	      - T1 (EEPROM) for static configurations
	      - T2 (NOR Flash) for mission-critical persistent state
	      - T3 (eMMC/microSD) for bulk/payload data
	      - T4 (BB eMMC) for reboot images
	      - Drop if data is transient and cheap to regenerate
5. **Storage Routing**: Forward processed data block to appropriate memory tier interface via `memSetOut[j]` based on categorization.
6. **Wear Management**: For erasable media (T2-T4), track erase counts per block and rotate usage to balance wear.
7. **Completion**: Send status via `storeStatusOut` and log any errors via `storeErrorOut`.

### 3.4 Commands

| Mnemonic | Args | Description |
|----------|------|-------------|
| `STORE_DATA` | `dataId: U32`, `size: U16` | Command to store a data buffer of specified size (alternative to port-driven input) |
| `GET_FREE_SPACE` | `memType: U8` | Request available space in specified memory tier (0=T1,1=T2,2=T3,3=T4) |
| `WEAR_STATUS` | `memType: U8` | Request wear level statistics for specified memory tier |
| `FORCE_GC` | `memType: U8` | Trigger garbage collection on specified memory tier (for log-structured systems) |

## 4. State Machine

`StorageManager` uses a flat F' state machine with the following states:

```
RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN
  ↑_____________ error from any state ___________|
```

- **RESET**: Initialize all internal variables, clear error states, prepare hardware interfaces.
- **WAIT_RESET**: Hold in reset until de-asserted by rate group tick or command.
- **ENABLE**: Attempt to enable and initialize memory tier interfaces (T1-T4). On success, proceed to CONFIGURE; on error, return to RESET.
- **CONFIGURE**: Read configuration parameters from `PrmDb` for each memory tier (e.g., wear leveling thresholds, CRC policies). Apply settings to hardware interfaces.
- **RUN**: 
  - On `schedIn` tick: 
    - Check for incoming data requests on `storeDataIn` ports.
    - Process and route data as per Section 3.3.
    - Update wear leveling counters.
    - Perform background garbage collection if thresholds exceeded.
  - On error from any subsystem (memory tier interface, CRC failure, etc.):
    - Log `WARNING_HI` via `storeErrorOut`.
    - Transition to RESET for self-healing.
  - On `parameterUpdated()`: 
    - If parameter affects memory tier configuration, transition to CONFIGURE state to reload settings.

**Note**: The `RESET` state is entered autonomously on any error detection, enabling self-healing without ground intervention. Configuration via parameters allows ground adjustment of wear thresholds and CRC policies.

## 5. Notes

- The `storeDataIn` array size (5) corresponds to the five CDH subsystems shown in the memory architecture diagram: Comms, EPS, Payload, ADCS, and FlightSW. This allows dedicated data channels per subsystem.
- Data types communicated via `dataTypeIn` ports should follow an enum defined in the StorageManager's module (e.g., `StorageManager.DataType` with values like `DEVICE_CONFIG`, `SENSITIVE_STATE`, `IMAGE_PAYLOAD`, `REBOOT_IMAGE`, `BULK_DATA`, `TRANSIENT`).
- Wear leveling implementation assumes erasable memory technologies expose erase count information via their OSAL interfaces; for NOR Flash, this may require tracking erase counts in a spare area or using built-in wear leveling if available.
- The component does not directly handle T0 (RAM) as volatile temporary storage; this is managed by the requesting components themselves. StorageManager only handles persistent tiers (T1-T4).
- Health monitoring (`Svc::Health`) covers the StorageManager component itself but not the underlying memory hardware, which is presumed to report errors via standard interfaces.
- Related documentation: 
  - [FlightComputerMemory](./FlightComputerMemory.md) - Overall memory architecture and tiers
  - Subsystem SDDs (CommsApplication, EPSApplication, etc.) for data production/consumption patterns
  - F' OSAL documentation for memory tier interfaces
  - Linux driver specifications for actual memory hardware (EEPROM, NOR, eMMC, microSD)
  