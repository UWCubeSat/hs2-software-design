# AssertFatalAdapter

## Overview

`AssertFatalAdapter` is an F' passive component that converts framework assertions into FATAL events for the CDH fault-handling path.

## HuskySat-2 Use

The adapter gives assertion failures the same event and fault-routing behavior as other fatal software conditions. Reset and recovery policy remains a system-level deployment decision.

## F' Reference

[AssertFatalAdapter software design document](https://fprime.jpl.nasa.gov/latest/Svc/AssertFatalAdapter/docs/sdd/)

