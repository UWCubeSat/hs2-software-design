# Ccsds::TcDeframer

## Overview

`Ccsds::TcDeframer` is an F' passive component that validates and removes CCSDS telecommand framing from received frames.

## HuskySat-2 Use

It is part of the uplink chain between radio reception and space-packet processing. Command dispatch occurs only after the frame has passed the communications validation stages.

## F' Reference

[CCSDS TC deframer source](https://github.com/nasa/fprime/tree/devel/Svc/Ccsds/TcDeframer)

