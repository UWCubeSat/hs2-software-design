# RateGroupDriver

## Overview

`RateGroupDriver` is an F' passive component that divides the primary system tick into scheduled rate-group signals.

## HuskySat-2 Use

The rate-group driver provides the timing events used by CDH, applications, and hardware managers. Each consumer performs its periodic work through a configured rate-group connection rather than creating an independent timing source.

## F' Reference

[RateGroupDriver software design document](https://fprime.jpl.nasa.gov/latest/Svc/RateGroupDriver/docs/sdd/)

