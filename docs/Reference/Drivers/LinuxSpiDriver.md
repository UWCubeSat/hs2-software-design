# LinuxSpiDriver

## Overview

`LinuxSpiDriver` is an F' Level 1 passive driver that exposes Linux SPI transactions through typed F' ports. It owns access to a configured SPI device but does not interpret the bytes exchanged with that device.

## HuskySat-2 Use

The IMU, sun-sensor ADC, camera interfaces, and external flash may use SPI depending on the selected hardware configuration. Their Level 2 managers or Level 3 applications own chip-specific transaction formats and sequencing.

## Component Type and Integration

The driver is passive and synchronous. A topology instance configures the Linux SPI device and connects transaction ports to a manager. Chip-select, register interpretation, initialization, and retry behavior are owned by the higher-level component that uses the bus.

## F' Reference

[LinuxSpiDriver source and interface](https://github.com/nasa/fprime/tree/devel/Drv/LinuxSpiDriver)

