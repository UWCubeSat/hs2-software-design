# FlightComputerMemory SDD

## 1. Overview

This document details the memory storage architecture, flow, and tiering for the CubeSat flight computer.

## 2. Requirements

This document is informational and does not contain formal requirements. It describes the memory architecture design for the HS2 CubeSat flight computer.

## 3. Design

### 3.1 Memory Tiers and Hardware Mapping

```mermaid
graph LR
    T0["T0: Volatile, fast R/W"] --> RAM["RAM (512 MB)"]
    T1["T1: Static Configs"] --> EEPROM["EEPROM (32 KB)"]
    T2["T2: Mission Critical Persistent State"] --> NOR0["Nor Flash 0 (512 MB)"]
    T2 --> NOR1["Nor Flash 1 (512 MB)"]
    T2 --> NOR2["Nor Flash 2 (512 MB)"]
    T2 --> NOR3["Nor Flash 3 (512 MB)"]
    T3["T3: Mass Storage (Persistent)"] --> eMMC["eMMC (64 GB)"]
    T3 --> SSD["eUSB SSD (64 GB)"]
    T4["T4: Reboot Images"] --> BB_eMMC["BB eMMC (4 GB)"]
```

### Tier Definitions and Contents

- **T0 (RAM):** Image buffers, DMA buffers, sensor readings, temp, telemetry packets, command buffers, thread space.
- **T1 (EEPROM):** Board serial no, Sensor/I2C/SPI/T2/T3 device configs, manufacturing constants.
- **T2 (NOR Flash):** Four NOR flash chips (only one active at a time via SPI chip select); stores satellite state (safe mode, current operating mode, mission phase), Sun sensor calibration, magnetometer bias, Battery SOC/Health estimates, Fault management (watchdog, fault log), LOST/FOUND processing results (w/ reference to image), info about chip state (bitmap, wear stats per chip), payload logs, status of FC software copy; health monitoring per chip.
- **T3 (eMMC + eUSB SSD):** Images, Science data, payload logs.
- **T4 (BB eMMC):** Linux kernel, bootloader, root/base FS, factory configs, FC SW.

### 3.2 Data Storage Decision Flowchart

```mermaid
flowchart TD
    Start([Piece of data]) --> Q1{Needs to survive reboot?}
    Q1 -- Yes --> Q2{Hardware config <br/> static?}
    Q1 -- No --> T0[T0]
    
    Q2 -- Yes --> T1[T1]
    Q2 -- No --> Q3{Executable/ <br/> recovery info?}
    
    Q3 -- Yes --> T4[T4]
    Q3 -- No --> Q4{Bulk/Payload <br/> data?}
    
    Q4 -- Yes --> T3_choice[T3]
    Q4 -- No --> Q5{Expensive and/or <br/> hard to recreate?}
    
    Q5 -- Yes --> T2[T2]
    Q5 -- No --> Drop[Don't store <br/> or regenerate <br/> later]
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

    subgraph F_OSAL ["Memory Tiers"]
        T1_Interface[T1]
        T2_Interface[T2]
        T3_Interface[T3]
        T4_Interface[T4]
    end

    subgraph Linux_Drivers ["F' OSAL + Linux Dev. Drivers"]
        I2C[I2C1]
        SPI1[SPI1]
        eUSB[eUSB]
        MMC1[MMC1]
        MMC2[MMC2]
    end

    subgraph HW ["HW"]
        EEPROM_HW[EEPROM]
        NOR0_HW[Nor Flash 0<br/>SPI1_CS0]
        NOR1_HW[Nor Flash 1<br/>SPI1_CS1]
        NOR2_HW[Nor Flash 2<br/>SPI1_CS2]
        NOR3_HW[Nor Flash 3<br/>SPI1_CS3]
        eUSB_SSD_HW[eUSB_SSD]
        eMMC_HW[eMMC]
        BB_eMMC_HW[BB eMMC]
    end

    %% Data Flow - CDH Subsystems to Storage Manager
    Comms -->|data to store<br/>type of data| StorageManager
    EPS -->|data to store<br/>type of data| StorageManager
    Payload -->|data to store<br/>type of data| StorageManager
    ADCS -->|data to store<br/>type of data| StorageManager
    FlightSW -->|data to store<br/>type of data| StorageManager
    
    %% Storage Manager to Tiers
    StorageManager -- "device configs" --> T1_Interface
    StorageManager -- "sensitive data" --> T2_Interface
    StorageManager -- "images / payload" --> T3_Interface
    StorageManager -- "handle reboots" --> T4_Interface

    %% Interfaces to Drivers
    T1_Interface <--> I2C
    T2_Interface <--> SPI1
    T3_Interface <--> eUSB
    T3_Interface <--> MMC1
    T4_Interface <--> MMC2

    %% Drivers to HW
    I2C <--> EEPROM_HW
    SPI1 <--> NOR0_HW
    SPI1 <--> NOR1_HW
    SPI1 <--> NOR2_HW
    SPI1 <--> NOR3_HW
    eUSB <--> eUSB_SSD_HW
    MMC1 <--> eMMC_HW
    MMC2 <--> BB_eMMC_HW
```


## 4. State Machine

This document does not describe a state machine as it is not an F' component. The memory architecture is managed by the Storage Manager component and OSAL layers.

## 5. Notes

- This document is intended to aid understanding of the memory storage design and is not an F' component specification.
- **Integrations:** This memory architecture is implemented by the Storage Manager F' component (Layer 2), which interfaces with the CDH subsystems (Comms, EPS, DataCollection, ScienceInference, ADCS) via the F' framework, and with the Linux drivers and hardware for the memory tiers. Related documentation includes the subsystem SDDs and driver specifications.