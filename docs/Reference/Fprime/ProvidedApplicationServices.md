## F' Application and Data Services

These F' components support the Level 3 application and data paths. They are grouped here so that short framework-component descriptions do not each force a new PDF page.

### Svc::ComQueue

**Type:** Active component.

`Svc::ComQueue` queues event, telemetry, and other outgoing packets for transmission through the communications topology. It decouples packet producers from radio timing and framing while `ComApplication` controls the mission communication mode around the shared path.

[ComQueue source and interface](https://github.com/nasa/fprime/tree/devel/Svc/ComQueue)

### Svc::ComStub

**Type:** Passive communications bridge.

`Svc::ComStub` connects packet-oriented F' components to a byte-stream driver. The CCSDS communications path uses it to connect framing and deframing components to `LinuxUartDriver` through the radio manager.

[ComStub source and interface](https://github.com/nasa/fprime/tree/devel/Svc/ComStub)

### Svc::FrameAccumulator

**Type:** Passive component.

`Svc::FrameAccumulator` accumulates incoming bytes until a complete communications frame is available for deframing. It handles stream boundaries without owning command interpretation.

[FrameAccumulator source and interface](https://github.com/nasa/fprime/tree/devel/Svc/FrameAccumulator)

### Svc::FprimeRouter

**Type:** Passive component.

`Svc::FprimeRouter` routes decoded packets to the appropriate command, telemetry, or event destination. It connects the communications receive path to `Svc::CmdDispatcher` and other packet consumers after uplink frames have been deframed.

[FprimeRouter source and interface](https://github.com/nasa/fprime/tree/devel/Svc/FprimeRouter)

### Svc::FprimeFramer and Svc::FprimeDeframer

**Type:** Passive components.

`Svc::FprimeFramer` wraps outgoing packets in native F' framing. `Svc::FprimeDeframer` removes native F' framing from received packets before command routing.

[FprimeFramer source and interface](https://github.com/nasa/fprime/tree/devel/Svc/FprimeFramer)  
[FprimeDeframer source and interface](https://github.com/nasa/fprime/tree/devel/Svc/FprimeDeframer)

### Svc::Ccsds::TmFramer and Svc::Ccsds::TcDeframer

**Type:** Passive components.

`Svc::Ccsds::TmFramer` formats outgoing telemetry and event packets into CCSDS telemetry frames. `Svc::Ccsds::TcDeframer` validates and removes CCSDS telecommand framing from received frames before space-packet processing.

[CCSDS TM framer source](https://github.com/nasa/fprime/tree/devel/Svc/Ccsds/TmFramer)  
[CCSDS TC deframer source](https://github.com/nasa/fprime/tree/devel/Svc/Ccsds/TcDeframer)

### Svc::Ccsds::SpacePacketFramer and Svc::Ccsds::SpacePacketDeframer

**Type:** Passive components.

`Svc::Ccsds::SpacePacketFramer` constructs CCSDS space packets from outgoing F' packet data. `Svc::Ccsds::SpacePacketDeframer` validates and extracts CCSDS space packets from received telecommand frames.

[CCSDS space-packet framer source](https://github.com/nasa/fprime/tree/devel/Svc/Ccsds/SpacePacketFramer)  
[CCSDS space-packet deframer source](https://github.com/nasa/fprime/tree/devel/Svc/Ccsds/SpacePacketDeframer)

### Svc::Ccsds::ApidManager

**Type:** Passive component.

`Svc::Ccsds::ApidManager` manages CCSDS application-process identifiers and packet sequence information used by the communications path.

[CCSDS APID manager source](https://github.com/nasa/fprime/tree/devel/Svc/Ccsds/ApidManager)

### Svc::ComAggregator

**Type:** Passive component.

`Svc::ComAggregator` combines outgoing packet streams before they enter the configured communications framing path while preserving packet boundaries.

[ComAggregator source and interface](https://github.com/nasa/fprime/tree/devel/Svc/ComAggregator)

### Svc::FileUplink and Svc::FileDownlink

**Type:** Active components.

`Svc::FileUplink` receives file-transfer packets and reconstructs files in the onboard filesystem. `Svc::FileDownlink` reads files from onboard storage and queues them for downlink. Together they support configuration, parameter, science-product, log, and diagnostic-file transfers.

[FileUplink source and interface](https://github.com/nasa/fprime/tree/devel/Svc/FileUplink)  
[FileDownlink source and interface](https://github.com/nasa/fprime/tree/devel/Svc/FileDownlink)

### Svc::FileManager

**Type:** Active component.

`Svc::FileManager` provides commands and services for creating, deleting, inspecting, and otherwise managing files in the onboard filesystem.

[FileManager source and interface](https://github.com/nasa/fprime/tree/devel/Svc/FileManager)

### Svc::PrmDb

**Type:** Active component.

`Svc::PrmDb` stores and retrieves persistent component parameters. It provides the parameter-persistence path used to retain configured values across deployments and reboot cycles.

[PrmDb source and interface](https://github.com/nasa/fprime/tree/devel/Svc/PrmDb)

### Svc::DpManager, Svc::DpWriter, and Svc::DpCatalog

**Type:** Active components.

`Svc::DpManager` allocates and manages data-product containers. `Svc::DpWriter` writes completed science and diagnostic products to persistent storage. `Svc::DpCatalog` catalogs products and manages their availability for later operations such as downlink.

[DpManager source and interface](https://github.com/nasa/fprime/tree/devel/Svc/DpManager)  
[DpWriter source and interface](https://github.com/nasa/fprime/tree/devel/Svc/DpWriter)  
[DpCatalog source and interface](https://github.com/nasa/fprime/tree/devel/Svc/DpCatalog)
