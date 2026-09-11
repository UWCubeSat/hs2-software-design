# Svc::TlmChan

## Overview

`Svc::TlmChan` is an F' active component that stores the most recent telemetry-channel values and makes them available to the telemetry collection path.

## HuskySat-2 Use

Level 2 managers and Level 3 applications publish device and mission telemetry through the standard F' telemetry interface. `Svc::TlmChan` retains the current values for collection and downlink.

## F' Reference

[TlmChan software design document](https://fprime.jpl.nasa.gov/latest/Svc/TlmChan/docs/sdd/)

