# Svc::EventManager

## Overview

`Svc::EventManager` is an F' active component that collects component events, applies event filtering, and packages events for downlink.

## HuskySat-2 Use

Applications and hardware managers publish diagnostic, warning, and fatal events to this shared path. The communications topology carries the resulting event packets to the ground interface.

## F' Reference

[EventManager software design document](https://fprime.jpl.nasa.gov/latest/Svc/EventManager/docs/sdd/)

