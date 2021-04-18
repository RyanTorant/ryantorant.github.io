---
layout: post
title: Float to int, the fast way
---

# From float to int (when you know it's an int)

As a fun crazy project I've been designing and implementing a raytracing GPU (on an FPGA). While I'll probably do an in depth series of posts soon, for now I just wanted to share a cool snippet that could be useful to someone.
I wanted a relatively generic code design for my RT GPU so I went with programable shader cores, but to keep it realistic, they only process 32 bits floats. There's no hardware for integer operations because you don't really need them on graphics, except... when you want to read memory.

While you can accurately operate on ints up to 2^24 as floats, when you actually interface with the memory subsystem you need to convert the binary representation of the float to a two's complement integer. The FPU unit I'm using for my design does have an operation for float to int conversion, but it takes 4 clocks and handles many more cases than I care about (rounding, numbers > 2^24, etc). Considering I assume my numbers are already integers on the [0,2^24) range then it got me thinking if there wasn't some bit hackery that could get this more efficiently. Turns out there is!

## The magic
... it's actually quite simple :D Whole integers up to 2^24 (I'm ignoring the sign as I don't care about it on my use) are actually stored on the mantissa, plus the implicit leading bit, so getting the number back is just a matter of shifting the mantissa based on the exponent, and adding the leading bit. Remember that the exponent is biased by 127 so you need to subtract that. 

## Let's see the code!
Well, just for reference, here's a C++ implementation, though it makes little sense to use this, you are way better off just doing a cast
```cpp
inline uint32_t as_uint24(float x)
{
	assert(x >= 0 && x < (1 << 24));

	uint32_t raw = *(uint32_t*)&x;

	uint32_t mantissa = raw & ((1 << 23) - 1);
	uint32_t exp = raw & (255 << 22);

	return (mantissa >> (23 - exp - 127)) | (1 << (exp - 127));
}
```
It first extracts the mantisa and exponent, then shifts back the mantissa to it's proper place, and adds back the leading bit. On a CPU, where you actually have dedicated hardware for this operation, and have no nice way to change individual bits, this is useless, BUT when you go to the HDL world, this becomes much more useful. An example implementation on verilog could be as follows
```verilog
module as_uint24 (
    input wire [31:0] float_in,
    output reg [23:0] uint_out
);

wire [4:0] exp;
assign exp = float_in[30:23] - 8'd127;

always @(float_in) begin
    if(float_in == 0) begin
        uint_out = 0;
    end else begin
        case (exp)
            0 : uint_out = 24'd1;
            1 : uint_out = {22'b0, 1'b1, float_in[22:22]};
            2 : uint_out = {21'b0, 1'b1, float_in[22:21]};
            3 : uint_out = {20'b0, 1'b1, float_in[22:20]};
            4 : uint_out = {19'b0, 1'b1, float_in[22:19]};
            5 : uint_out = {18'b0, 1'b1, float_in[22:18]};
            6 : uint_out = {17'b0, 1'b1, float_in[22:17]};
            7 : uint_out = {16'b0, 1'b1, float_in[22:16]};
            8 : uint_out = {15'b0, 1'b1, float_in[22:15]};
            9 : uint_out = {14'b0, 1'b1, float_in[22:14]};
            10 : uint_out = {13'b0, 1'b1, float_in[22:13]};
            11 : uint_out = {12'b0, 1'b1, float_in[22:12]};
            12 : uint_out = {11'b0, 1'b1, float_in[22:11]};
            13 : uint_out = {10'b0, 1'b1, float_in[22:10]};
            14 : uint_out = {9'b0, 1'b1, float_in[22:9]};
            15 : uint_out = {8'b0, 1'b1, float_in[22:8]};
            16 : uint_out = {7'b0, 1'b1, float_in[22:7]};
            17 : uint_out = {6'b0, 1'b1, float_in[22:6]};
            18 : uint_out = {5'b0, 1'b1, float_in[22:5]};
            19 : uint_out = {4'b0, 1'b1, float_in[22:4]};
            20 : uint_out = {3'b0, 1'b1, float_in[22:3]};
            21 : uint_out = {2'b0, 1'b1, float_in[22:2]};
            22 : uint_out = {1'b0, 1'b1, float_in[22:1]};
            23 : uint_out = {1'b1, float_in[22:0]};
            default: uint_out = 0;
        endcase
    end
end
endmodule
```
This synthesizes to just an array of muxes, so it's way faster than the bit manipulations you need on c++, and it can execute on a single clock, so it's better than the operation from the FPU I'm using. Plus, by having separate HW I can let the core continue it's execution while I process this conversion, as I'm designing memory accesses to be async.