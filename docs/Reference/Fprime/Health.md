# Health

## Overview

`Health` is an F' queued component that pings active components, detects missed responses, and provides the framework health-monitoring path.

## HuskySat-2 Use

Critical applications and hardware managers register their ping ports with the shared health service. Missed responses are converted into health events for the command-and-data-handling and fault-response paths.

## F' Reference

[Health software design document](https://fprime.jpl.nasa.gov/latest/Svc/Health/docs/sdd/)

