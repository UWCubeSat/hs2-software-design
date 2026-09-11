# Drv::LinuxSpiDriver

## Overview

`Drv::LinuxSpiDriver` is an F' Level 1 passive driver that exposes Linux SPI transactions through typed F' ports. It owns access to a configured SPI device but does not interpret the bytes exchanged with that device.

## HuskySat-2 Use

The IMU and sun-sensor ADC use SPI-backed interfaces in the current hardware-manager designs. Their Level 2 managers own chip-specific transaction formats and sequencing; the current SSD and microSD storage architecture uses Linux filesystem services rather than an SPI NOR storage path.

## Component Type and Integration

The driver is passive and synchronous. A topology instance configures the Linux SPI device and connects transaction ports to a manager. Chip-select, register interpretation, initialization, and retry behavior are owned by the higher-level component that uses the bus.

## F' Reference

[LinuxSpiDriver source and interface](https://github.com/nasa/fprime/tree/devel/Drv/LinuxSpiDriver)
