# 1-D Linear Convolution Using Verilog HDL

## Overview

This project implements a **1-D linear convolution** system using Verilog HDL. The design is implemented using a modular hardware architecture consisting of a control path, datapath, RAM, ROM, and MAC (Multiply-Accumulate) unit.

The design performs convolution between a 3-point input sequence and a fixed 3-point filter sequence.

## Architecture

The design consists of the following modules:

* **Control Path** – FSM that controls memory writing, address selection, MAC operation, and completion.
* **Datapath** – Connects the RAM, ROM, and MAC units.
* **RAM** – Stores the 3 input samples.
* **ROM** – Stores the fixed filter coefficients.
* **MAC Unit** – Performs multiplication and accumulation.
* **Top Module** – Integrates the complete design.

## Convolution

The input sequence is:

```text
x = [x0, x1, x2]
```

The filter coefficients stored in ROM are:

```text
h = [1, 2, 1]
```

The resulting 1-D linear convolution contains five output samples:

```text
y = [y0, y1, y2, y3, y4]
```

The output samples are calculated as:

```text
y0 = x0 × h0

y1 = x0 × h1 + x1 × h0

y2 = x0 × h2 + x1 × h1 + x2 × h0

y3 = x1 × h2 + x2 × h1

y4 = x2 × h2
```

## Hardware Implementation

The input samples are first stored in the RAM. The control-path FSM then selects the appropriate RAM and ROM addresses for each multiplication.

The MAC unit performs the required multiplication and accumulation operations. The `n` input selects which convolution output is calculated.

```text
n = 0 → y[0]
n = 1 → y[1]
n = 2 → y[2]
n = 3 → y[3]
n = 4 → y[4]
```

The `done` signal indicates that the selected convolution output is available at `acc`.

## Simulation

The design is verified using a Verilog testbench.

Test input:

```text
x = [-1, 2, -3]
```

Filter:

```text
h = [1, 2, 1]
```

Expected output:

```text
y[0] = -1
y[1] = 0
y[2] = 0
y[3] = -4
y[4] = -3
```

## Files

```text
design/
└── convolution.v

simulation/
└── tb_convolution.v
```

The RTL design and testbench can be added to the respective folders in this repository.

## Tools

* Verilog HDL
* Xilinx Vivado
* FPGA-based RTL simulation
