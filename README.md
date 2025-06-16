# Report of Digital Circuits & Systems Final Project

## Design Structure
To reduce cycles spend on accessing DRAM, I use a single way, single line cache to store temporary data, which significantly reduce overall execution cycles.

![Untitled Diagram](https://github.com/user-attachments/assets/ac98606f-a96d-492f-b64b-be657a695323)

## Cache

### Instruction Cache
It a 256 byte single way, single line cache. For instructions, there're no write back needed so it just reads data to the cache. The cache has a block address cooresponeded to Instruction DRAM, which is the upper few bits of the DRAM address. When the program counter points to the addres outside current block address, cache miss happens, and now we need to burst read the instruction DRAM for 128 address.

### DATA Cache
It a 256 byte single way, single line cache. It needs to handle both direction, read & write. So if the store instruction is executed at the current block of cache, the cache is marked as dirty. When next DRAM read or DRAM check is needed, and if the data cache is dirty, we need to execute write back, which then now really write the data back to DRAM. If an data request is outside current block, and now we need to read DRAM. This significantly reduce execution cycles because we only do read/write when needed.

## Instruction pipeline
To reduce overall execution time, we can execute tasks in multiple cycles, but under faster clk speed. We use 1st cycle of an execution to decode the instruction set. For this design, the fastest clk spped avialible is 2.8ns, limited by the speed of reg array.

### ADD & SUB 000_0 / 000_1
```
1: fetch regfile
2: calculation
```
The time to fetch a reg array is actually quite long so I need to give a whole cycle for fetching regfiles and store them into a temporary register to speed up the next calculation.

### Mult 001
```
1: getch regfile
2: partial mult
3: add partial result 1
4: add partial result 2
```
I pipeline the singed value mult with long multiplication, but for signed number it's more complicated then unsigned. The following code block display a simple signed 4-bit (2's complement) multiplication. We can expect that the result should be 8 bits, for the first partial line, the result should be 1110, but we need to add **"sign extentions"** up to MSB, so we add 4 1s in front, the next line we add 3 1s..., so on. For the 0 bit of  the multiplier, just make the partial result all 0. The last line is different, instead of normal addition of partial result, we need to subtract previous result with it, which can be seen as the addition of the 2's complement of the last partial result.
```
           1 1 1 0                  1 1 1 0      = -2
 x         1 0 1 1        x         1 0 1 1      = -5
-------------------      -------------------     
   1 1 1 1 1 1 1 0          1 1 1 1 1 1 1 0      <-  mul_partial_pip[0]
 + 1 1 1 1 1 1 0     ->   + 1 1 1 1 1 1 0        <-  mul_partial_pip[1]
 + 0 0 0 0 0 0            + 0 0 0 0 0 0          <-  mul_partial_pip[2]
 - 1 1 1 1 0              + 0 0 0 1 0   <-  2's  <-  mul_partial_pip[3]
                         -------------------
                            0 0 0 0 1 0 1 0      = 10
```
To realize it in verilog, the following code is used:

Partial result
```
reg [15:0] mul_partial_pip [0:7];
//for every posedge clk
mul_partial_pip[0] <= {{8{reg_file_rs_pip[7] & reg_file_rt_pip[0]}}, reg_file_rs_pip & {8{reg_file_rt_pip[0]}}};
mul_partial_pip[1] <= {{7{reg_file_rs_pip[7] & reg_file_rt_pip[1]}}, reg_file_rs_pip & {8{reg_file_rt_pip[1]}}, 1'b0};
mul_partial_pip[2] <= {{6{reg_file_rs_pip[7] & reg_file_rt_pip[2]}}, reg_file_rs_pip & {8{reg_file_rt_pip[2]}}, 2'b00};
mul_partial_pip[3] <= {{5{reg_file_rs_pip[7] & reg_file_rt_pip[3]}}, reg_file_rs_pip & {8{reg_file_rt_pip[3]}}, 3'b000};
mul_partial_pip[4] <= {{4{reg_file_rs_pip[7] & reg_file_rt_pip[4]}}, reg_file_rs_pip & {8{reg_file_rt_pip[4]}}, 4'b0000};
mul_partial_pip[5] <= {{3{reg_file_rs_pip[7] & reg_file_rt_pip[5]}}, reg_file_rs_pip & {8{reg_file_rt_pip[5]}}, 5'b00000};
mul_partial_pip[6] <= {{2{reg_file_rs_pip[7] & reg_file_rt_pip[6]}}, reg_file_rs_pip & {8{reg_file_rt_pip[6]}}, 6'b000000};
mul_partial_pip[7] <= {{1{reg_file_rs_pip[7] & reg_file_rt_pip[7]}}, reg_file_rs_pip & {8{reg_file_rt_pip[7]}}, 7'b0000000};
```

Add partial result 1
```
reg [15:0] add_partial_pip_a, add_partial_pip_b;
//for every posedge clk
add_partial_pip_a <= (mul_partial_pip[0] + mul_partial_pip[1] + mul_partial_pip[2] + mul_partial_pip[3]) & 16'hFFFF;
add_partial_pip_b <= (mul_partial_pip[4] + mul_partial_pip[5] + mul_partial_pip[6] + {(~mul_partial_pip[7][15:7] + 1) & 9'hFFF, 7'b0000000}) & 16'hFFFF;
```

Add partial result 2
```
reg_file[current_inst_rd] <= $signed(add_partial_pip_a) + $signed(add_partial_pip_b);
//noted the ultil now the number is finally marked as signed
```

### LOAD 010
```
1: fetch regfile
2: data address partial mult
3: add partial mult
4: fetch data cache (or fetch DRAM if cache misses)
5; load data to reg_file
```
We now encounter another reg array, so another cycle for fetching is needed.

### STORE 011
```
1: fetch regfile
2: data address partial mult
3: add partial mult
4: store data to cache (or writeback if cache misses)
```

### BEQ 100
```
1: fetch regfile
2: change programcounter
```

### JUMP 101
```
1: change programcounter
```
