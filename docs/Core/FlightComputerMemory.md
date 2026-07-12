# FlightComputerMemory SDD

## 1. Overview

This document details the memory storage architecture, flow, and tiering for the CubeSat flight computer.

## 2. Requirements

This document is informational and does not contain formal requirements. It describes the memory architecture design for the HS2 CubeSat flight computer.

## 3. Design

### 3.1 Memory Tiers and Hardware Mapping

```mermaid
graph LR
    T0["T0: Volatile, fast R/W"] --> RAM[RAM]
    T1["T1: Static Configs (Persistent, infrequent R/W)"] --> EEPROM[EEPROM]
    T2["T2: Mission Critical Persistent State"] --> NOR0[Nor Flash 0]
    T2 --> NOR1[Nor Flash 1]
    T2 --> NOR2[Nor Flash 2]
    T2 --> NOR3[Nor Flash 3]
    T3["T3: Mass Storage (Persistent)"] --> eMMC_microSD[eMMC + microSD]
    T4["T4: Reboot Images"] --> BB_eMMC[BB eMMC]
```

### Tier Definitions and Contents

- **T0 (RAM):** Image buffers, DMA buffers, sensor readings, temp, telemetry packets, command buffers, thread space.
- **T1 (EEPROM):** Board serial no, Sensor/I2C/SPI/T2/T3 device configs, manufacturing constants.
- **T2 (NOR Flash):** Four NOR flash chips (only one active at a time via SPI chip select); stores satellite state (safe mode, current operating mode, mission phase), Sun sensor calibration, magnetometer bias, Battery SOC/Health estimates, Fault management (watchdog, fault log), LOST/FOUND processing results (w/ reference to image), info about chip state (bitmap, wear stats per chip), payload logs, status of FC software copy; health monitoring per chip.
- **T3 (eMMC + microSD):** Images, Science data, payload logs.
- **T4 (BB eMMC):** Linux kernel, bootloader, root/base FS, factory configs, FC SW.

### 3.2 Data Storage Decision Flowchart

```mermaid
flowchart TD
    Start([Piece of data]) --> Q1{Needs to survive reboot?}
    Q1 -- No --> T0[T0]
    Q1 -- Yes --> Q2{Hardware config <br/> static?}
    
    Q2 -- Yes --> T1[T1]
    Q2 -- No --> Q3{Executable/ <br/> recovery info?}
    
    Q3 -- Yes --> T4[T4]
    Q3 -- No --> Q4{Bulk/Payload <br/> data?}
    
    Q4 -- Yes --> T3_choice[T3]
    Q4 -- No --> Q5{Expensive and/or <br/> hard to recreate?}
    
    Q5 -- No --> Drop[Don't store <br/> or regenerate <br/> later]
    Q5 -- Yes --> T2[T2]
```

### 3.3 System Architecture and Interfaces

```mermaid
graph TD
    subgraph CDH_Subsystems ["CDH / Subsystems (F')"]
        Comms[Comms]
        EPS[EPS]
        Payload[Payload]
        ADCS[ADCS]
        FlightSW[Flight SW]
    end

    subgraph F_Component ["F' Component"]
        StorageManager["Storage Manager<br/>(CRC verification, categorize data, hamming codes, wear management, create data block)"]
    end

    subgraph F_OSAL ["F' OSAL (Memory Tiers)"]
        T1_Interface[T1]
        T2_Interface[T2]
        T3_Interface[T3]
        T4_Interface[T4]
    end

    subgraph Linux_Drivers ["Linux Dev. Drivers"]
        I2C[I2C]
        SPI1[SPI]
        SPI2[SPI]
        MMC1[MMC]
        MMC2[MMC]
    end

    subgraph HW ["HW"]
        EEPROM_HW[EEPROM]
        NOR0_HW[Nor Flash 0]
        NOR1_HW[Nor Flash 1]
        NOR2_HW[Nor Flash 2]
        NOR3_HW[Nor Flash 3]
        eMMC_HW[eMMC]
        microSD_HW[microSD]
        BB_eMMC_HW[BB eMMC]
    end

    %% Data Flow
    CDH_Subsystems -- "data to store<br/>type of data" --> StorageManager
    
    %% Storage Manager to Tiers
    StorageManager -- "device configs" --> T1_Interface
    StorageManager -- "sensitive data" --> T2_Interface
    StorageManager -- "images / payload" --> T3_Interface
    StorageManager -- "handle reboots" --> T4_Interface

    %% Interfaces to Drivers
    T1_Interface <--> I2C
    T2_Interface <--> SPI1
    T3_Interface <--> SPI2
    T3_Interface <--> MMC1
    T4_Interface <--> MMC2

    %% Drivers to HW
    I2C <--> EEPROM_HW
    SPI1 <--> NOR0_HW
    SPI1 <--> NOR1_HW
    SPI1 <--> NOR2_HW
    SPI1 <--> NOR3_HW
    SPI2 <--> eMMC_HW
    MMC1 <--> microSD_HW
    MMC2 <--> BB_eMMC_HW
```

## 4. State Machine

This document does not describe a state machine as it is not an F' component. The memory architecture is managed by the Storage Manager component and OSAL layers.

## 5. Notes
- This document is intended to aid understanding of the memory storage design and is not an F' component specification.
- **Integrations:** This memory architecture is implemented by the Storage Manager F' component (Layer 2), which interfaces with the CDH subsystems (Comms, EPS, DataCollection, ScienceInference, ADCS) via the F' framework, and with the Linux drivers and hardware for the memory tiers. Related documentation includes the subsystem SDDs and driver specifications.
