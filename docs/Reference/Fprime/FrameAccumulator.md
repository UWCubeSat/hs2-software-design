# Svc::FrameAccumulator

## Overview

`Svc::FrameAccumulator` is an F' passive component that accumulates incoming bytes until a complete communications frame is available for deframing.

## HuskySat-2 Use

It sits on the uplink path between the radio byte stream and the configured F' or CCSDS deframer. It handles stream boundaries without owning command interpretation.

## F' Reference

[FrameAccumulator source and interface](https://github.com/nasa/fprime/tree/devel/Svc/FrameAccumulator)

