# LinuxUartDriver

## Overview

`LinuxUartDriver` is an F' Level 1 passive driver that exposes a Linux serial device through F' byte-stream ports. It owns operating-system access to a configured UART but does not interpret the protocol carried over that UART.

## HuskySat-2 Use

HuskySat-2 uses UART access for the EnduroSat S-band radio, the GNSS receiver, and other serial peripherals. `TmtcRadioManager` and `GnssManager` own the device-specific protocols above this driver.

## Component Type and Integration

The driver is passive and synchronous. A topology instance configures the Linux device path and serial settings, then connects its byte-stream ports to a hardware manager or communications component. Protocol framing, packet validation, retries, and mission state remain outside the driver.

## F' Reference

[LinuxUartDriver source and interface](https://github.com/nasa/fprime/tree/devel/Drv/LinuxUartDriver)

