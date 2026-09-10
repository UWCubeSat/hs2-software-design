# TlmPacketizer

## Overview

`TlmPacketizer` is an F' active component that groups telemetry channels into packets according to configured packet definitions and rate logic.

## HuskySat-2 Use

`ComApplication` changes packet rate logic as the spacecraft communication mode changes. The packetizer supplies the resulting telemetry packets to the communications path while the applications and managers remain responsible for publishing channel values.

## F' Reference

[TlmPacketizer source and interface](https://github.com/nasa/fprime/tree/devel/Svc/TlmPacketizer)

