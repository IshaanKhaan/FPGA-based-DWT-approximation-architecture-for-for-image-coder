# 1-D Linear Convolution Using Verilog HDL

## Overview

This project implements a **1-D linear convolution** system using Verilog HDL.

The design uses a modular hardware architecture consisting of:

* Control Path (FSM)
* Datapath
* RAM
* ROM
* MAC (Multiply-Accumulate) Unit
* Top Module

The design performs a **3-point × 3-point linear convolution**.

## Convolution

The input sequence used for simulation is:

```text
x = [-1, 2, -3]
```

The filter coefficients stored in ROM are:

```text
h = [1, 2, 1]
```

For two 3-point sequences, the linear convolution produces 5 output samples:

```text
y[0] = x[0] × h[0]

y[1] = x[0] × h[1] + x[1] × h[0]

y[2] = x[0] × h[2] + x[1] × h[1] + x[2] × h[0]

y[3] = x[1] × h[2] + x[2] × h[1]

y[4] = x[2] × h[2]
```

For the given input:

```text
x = [-1, 2, -3]
h = [1, 2, 1]
```

the expected output is:

```text
y = [-1, 0, 0, -4, -3]
```

## Hardware Architecture

### Control Path

The control path is implemented using a Finite State Machine (FSM). It controls:

* Writing input samples into RAM
* Selecting RAM and ROM addresses
* Enabling the MAC unit
* Resetting the accumulator
* Generating the `done` signal

### Datapath

The datapath connects:

```text
RAM → MAC
ROM → MAC
```

The RAM stores the input sequence, while the ROM stores the fixed filter coefficients.

### MAC Unit

The MAC unit performs:

```text
acc = acc + (a × b)
```

for the required convolution terms.

## Output Selection

The input `n` selects which convolution output is calculated:

```text
n = 0 → y[0]
n = 1 → y[1]
n = 2 → y[2]
n = 3 → y[3]
n = 4 → y[4]
```

The `done` signal indicates that the selected output is available at `acc`.

---

# Verilog Design Code

```verilog
`timescale 1ns / 1ps

module convolution_topmodule(

input clk,
input reset,
input start,

input [2:0] n,
input signed[7:0] data_in,

output signed [15:0] acc,
output done

);

wire [1:0] addr_x;
wire [1:0] addr_h;

wire we;
wire mac_en;
wire mac_reset;



convolution_controlpath CP(

.addr_x(addr_x),
.addr_h(addr_h),

.we(we),
.mac_en(mac_en),
.mac_reset(mac_reset),
.done(done),

.clk(clk),
.reset(reset),
.start(start),
.n(n)

);



convolution_datapath DP(

.clk(clk),

.mac_reset(mac_reset),

.we(we),
.mac_en(mac_en),

.addr_x(addr_x),
.addr_h(addr_h),

.data_in(data_in),

.acc(acc)

);

endmodule


//================ DATAPATH =================

module convolution_datapath(

input clk,
input mac_reset,

input we,
input mac_en,

input [1:0] addr_x,
input [1:0] addr_h,

input signed [7:0] data_in,

output signed [15:0] acc

);

wire signed [7:0] ram_data;
wire signed [7:0] rom_data;



ram_3x8 RAM(

.clk(clk),
.we(we),
.addr_x(addr_x),
.din(data_in),
.dout(ram_data)

);



rom_3x8 ROM(

.addr_h(addr_h),
.dout(rom_data)

);



mac MAC_UNIT(

.clk(clk),
.reset(mac_reset),
.mac_en(mac_en),

.a(ram_data),
.b(rom_data),

.acc(acc)

);

endmodule


//================ RAM 3x8 =================

module ram_3x8(
input clk,
input we,
input [1:0] addr_x,
input signed [7:0] din,
output reg signed [7:0] dout
);

reg signed [7:0] mem0, mem1, mem2;

initial begin
    mem0 = 8'd0;
    mem1 = 8'd0;
    mem2 = 8'd0;
end

// WRITE
always @(posedge clk)
begin
    if (we)
    begin
        case (addr_x)
            2'd0: mem0 <= din;
            2'd1: mem1 <= din;
            2'd2: mem2 <= din;
            default: ;
        endcase
    end
end

// ASYNCHRONOUS READ
always @(*)
begin
    case (addr_x)
        2'd0: dout = mem0;
        2'd1: dout = mem1;
        2'd2: dout = mem2;
        default: dout = 8'd0;
    endcase
end

endmodule


//================ ROM 3x8 =================

module rom_3x8(

input [1:0] addr_h,

output reg signed [7:0] dout

);

always @(*)

begin

case(addr_h)

2'b00:
    dout = 8'sd1;

2'b01:
    dout = 8'sd2;

2'b10:
    dout = 8'sd1;

default:
    dout = 8'sd0;

endcase

end

endmodule



//================ MAC =================

module mac(

input clk,

input reset,

input mac_en,

input signed [7:0] a,

input signed [7:0] b,

output reg signed [15:0] acc

);

wire signed [15:0] mult;
wire signed [15:0] sum;

assign mult = a*b;

assign sum = acc + mult;



always @(posedge clk)

begin

if(reset)

    acc <= 16'sd0;


else if(mac_en)

    acc <= sum;


else

    acc <= acc;

end

endmodule


//================ CONTROL PATH =================

module convolution_controlpath(

output reg [1:0] addr_x,
output reg [1:0] addr_h,

output reg we,
output reg mac_en,
output reg mac_reset,
output reg done,

input clk,
input reset,
input start,

input [2:0] n

);

reg [3:0] state, next_state;

parameter
S0  = 4'd0,
S1  = 4'd1,
S2  = 4'd2,
S3  = 4'd3,
S4  = 4'd4,
S5  = 4'd5,
S6  = 4'd6,
S7  = 4'd7,
S8  = 4'd8,
S9  = 4'd9,
S10 = 4'd10,
S11 = 4'd11,
S12 = 4'd12,
S13 = 4'd13,
S14 = 4'd14;

always @(posedge clk)
begin

if(reset)

state <= S0;

else

state <= next_state;

end

always @(*)
begin

next_state = state;

we = 0;
mac_en = 0;
mac_reset = 0;
done = 0;

addr_x = 0;
addr_h = 0;

case(state )

S0:
begin

    if(start)
    begin

        mac_reset = 1;

        next_state = S1;

    end

end

S1:
begin

    we = 1;

    addr_x = 0;

    next_state = S2;

end

S2:
begin

    we = 1;

    addr_x = 1;

    next_state = S3;

end

S3:
begin

    we = 1;

    addr_x = 2;

    next_state = S14;

end

S4:
begin

    addr_x = 0;

    addr_h = 0;

    mac_en = 1;

    next_state = S13;

end

S5:
begin

    addr_x = 0;

    addr_h = 1;

    mac_en = 1;

    next_state = S6;

end

S6:
begin

    addr_x = 1;

    addr_h = 0;

    mac_en = 1;

    next_state = S13;

end

S7:
begin

    addr_x = 0;

    addr_h = 2;

    mac_en = 1;

    next_state = S8;

end

S8:
begin

    addr_x = 1;

    addr_h = 1;

    mac_en = 1;

    next_state = S9;

end

S9:
begin

    addr_x = 2;

    addr_h = 0;

    mac_en = 1;

    next_state = S13;

end

S10:
begin

    addr_x = 1;

    addr_h = 2;

    mac_en = 1;

    next_state = S11;

end

S11:
begin

    addr_x = 2;

    addr_h = 1;

    mac_en = 1;

    next_state = S13;

end

S12:
begin

    addr_x = 2;

    addr_h = 2;

    mac_en = 1;

    next_state = S13;

end

S13:
begin

    done = 1;

    mac_reset = 1;

    next_state = S0;

end

S14:
begin

    case(n)

        3'd0: next_state = S4;

        3'd1: next_state = S5;

        3'd2: next_state = S7;

        3'd3: next_state = S10;

        3'd4: next_state = S12;

        default: next_state = S14;

    endcase

end

default:

    next_state = S0;

endcase

end

endmodule
```

---

# Testbench

The following testbench verifies all five outputs of the 3-point linear convolution.

```verilog
`timescale 1ns / 1ps

module tb_convolution;

reg clk = 0;
reg reset = 1;
reg start = 0;
reg [2:0] n = 0;
reg signed [7:0] data_in = 0;

wire signed [15:0] acc;
wire done;


convolution_topmodule DUT(

.clk(clk),
.reset(reset),
.start(start),
.n(n),
.data_in(data_in),
.acc(acc),
.done(done)

);


always #5 clk = ~clk;


initial begin

    @(posedge clk);

    reset = 0;


    // ---------------- n = 0 ----------------

    @(negedge clk);

    start   = 1;
    n       = 0;
    data_in = -1;

    @(posedge clk);
    @(posedge clk);

    @(negedge clk);

    data_in = 2;
    start = 0;

    @(posedge clk);

    @(negedge clk);

    data_in = -3;

    @(posedge clk);

    wait(done);

    $display("y[0] = %0d", acc);

    #10;


    // ---------------- n = 1 ----------------

    @(negedge clk);

    start   = 1;
    n       = 1;
    data_in = -1;

    @(posedge clk);
    @(posedge clk);

    @(negedge clk);

    data_in = 2;
    start = 0;

    @(posedge clk);

    @(negedge clk);

    data_in = -3;

    @(posedge clk);

    wait(done);

    $display("y[1] = %0d", acc);

    #10;


    // ---------------- n = 2 ----------------

    @(negedge clk);

    start   = 1;
    n       = 2;
    data_in = -1;

    @(posedge clk);
    @(posedge clk);

    @(negedge clk);

    data_in = 2;
    start = 0;

    @(posedge clk);

    @(negedge clk);

    data_in = -3;

    @(posedge clk);

    wait(done);

    $display("y[2] = %0d", acc);

    #10;


    // ---------------- n = 3 ----------------

    @(negedge clk);

    start   = 1;
    n       = 3;
    data_in = -1;

    @(posedge clk);
    @(posedge clk);

    @(negedge clk);

    data_in = 2;
    start = 0;

    @(posedge clk);

    @(negedge clk);

    data_in = -3;

    @(posedge clk);

    wait(done);

    $display("y[3] = %0d", acc);

    #10;


    // ---------------- n = 4 ----------------

    @(negedge clk);

    start   = 1;
    n       = 4;
    data_in = -1;

    @(posedge clk);
    @(posedge clk);

    @(negedge clk);

    data_in = 2;
    start = 0;

    @(posedge clk);

    @(negedge clk);

    data_in = -3;

    @(posedge clk);

    wait(done);

    $display("y[4] = %0d", acc);


    #10;

    $finish;

end

endmodule
```

## Expected Simulation Output

```text
y[0] = -1
y[1] = 0
y[2] = 0
y[3] = -4
y[4] = -3
```

## Tools Used

* Verilog HDL
* Xilinx Vivado
* RTL Simulation


