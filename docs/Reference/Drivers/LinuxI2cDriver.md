# Drv::LinuxI2cDriver

## Overview

`Drv::LinuxI2cDriver` is an F' Level 1 passive driver that exposes Linux I2C transactions through typed F' ports. It provides bus access while leaving register maps, device initialization, and recovery policy to the connected hardware manager.

## HuskySat-2 Use

The EPS board, temperature sensors, sun-sensor support hardware, and any I2C-configured IMU use this driver. Their managers validate device responses and translate register values into subsystem data.

## Component Type and Integration

The driver is passive and synchronous. A topology instance selects the Linux I2C bus and target address, then connects read, write, or combined transaction ports to a Level 2 manager. It does not contain device-specific commands or telemetry.

## F' Reference

[LinuxI2cDriver source and interface](https://github.com/nasa/fprime/tree/devel/Drv/LinuxI2cDriver)

