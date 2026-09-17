# Svc::ComQueue

## Overview

`Svc::ComQueue` is an F' active component that queues event, telemetry, and other outgoing packets for transmission through the communications topology.

## HuskySat-2 Use

It decouples packet producers such as `Svc::EventManager`, `Svc::TlmChan`, and `Svc::TlmPacketizer` from radio timing and framing. `ComApplication` controls the mission communication mode around this shared path.

## F' Reference

[ComQueue source and interface](https://github.com/nasa/fprime/tree/devel/Svc/ComQueue)

