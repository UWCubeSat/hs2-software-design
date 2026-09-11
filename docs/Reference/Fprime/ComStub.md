# Svc::ComStub

## Overview

`Svc::ComStub` is an F' passive communications bridge between packet-oriented F' components and a byte-stream driver.

## HuskySat-2 Use

The CCSDS communications path uses `Svc::ComStub` to connect framing and deframing components to `LinuxUartDriver` through the radio manager and the configured radio interface.

## F' Reference

[ComStub source and interface](https://github.com/nasa/fprime/tree/devel/Svc/ComStub)

