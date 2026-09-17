# Drv::LinuxGpioDriver

## Overview {.unnumbered .unlisted}

`Drv::LinuxGpioDriver` is an F' Level 1 passive driver for Linux GPIO lines. It exposes digital inputs and outputs without assigning meaning to a particular enable, reset, interrupt, or status signal.

## HuskySat-2 Use {.unnumbered .unlisted}

GPIO lines support hardware enables, reset lines, chip-select support, and discrete status signals throughout the ADCS, EPS, communications, and payload subsystems. The connected manager owns the safe value and the interpretation of each line.

## Component Type and Integration {.unnumbered .unlisted}

The driver is passive and synchronous. Each topology instance is configured for a Linux GPIO line and connects to the manager that owns the associated hardware behavior. Device sequencing and fault recovery remain above the driver layer.

## F' Reference {.unnumbered .unlisted}

[LinuxGpioDriver source and interface](https://github.com/nasa/fprime/tree/devel/Drv/LinuxGpioDriver)
