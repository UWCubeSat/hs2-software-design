# Svc::CmdDispatcher

## Overview

`Svc::CmdDispatcher` is an F' active component that decodes incoming command buffers and dispatches registered commands to the components that implement them.

## HuskySat-2 Use

It is part of the shared command-and-data-handling path. Ground commands and sequenced commands enter the topology through `Svc::CmdDispatcher`; the destination application or manager owns the command implementation.

## F' Reference

[CmdDispatcher software design document](https://fprime.jpl.nasa.gov/latest/Svc/CmdDispatcher/docs/sdd/)

