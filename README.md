# Report of Digital Circuits & Systems Final Project

## Design Structure
To reduce the number of cycles spend on accessing DRAM, I use a single way, single line cache to store temporary data, which significantly reduce overall execution cycles.

![Untitled Diagram](https://github.com/user-attachments/assets/ac98606f-a96d-492f-b64b-be657a695323)

## Cache

### Instruction Cache
It is a 256-byte single-way, single-line cache. For instructions, no write-back is needed, so it only reads data into the cache. The cache has a block address corresponding to the instruction DRAM, which is derived from the upper bits of the DRAM address. When the program counter points to an address outside the current block, a cache miss occurs, and a burst read of 128 addresses from instruction DRAM is triggered.

### DATA Cache
It is a 256-byte single-way, single-line cache. It needs to handle both read and write operations. If a store instruction is executed within the current cache block, the cache is marked as dirty. When the next DRAM read or check is needed, and if the data cache is dirty, we perform a write-back to DRAM. If a data request falls outside the current block, DRAM access is triggered. This significantly reduces execution cycles because DRAM read/write operations are only performed when necessary.

## Instruction pipeline
To reduce overall execution time, we can execute tasks over multiple cycles while using a higher clock speed. The first cycle of execution is used to decode the instruction. For this design, the fastest available clock speed is 2.8 ns, limited by the speed of the register array.

### ADD & SUB 000_0 / 000_1
```
1: fetch regfile
2: calculation
```
Fetching from the register array takes a relatively long time, so a full cycle is dedicated to fetch the registers and storing them in a temporary register to speed up the subsequent calculation.

### Mult 001
```
1: fetch regfile
2: partial mult
3: add partial result 1
4: add partial result 2
```
I pipeline the singed value mult using long multiplication, but for signed number it's more complicated than for unsigned numbers. The following code block display a simple signed 4-bit (2's complement) multiplication. We can expect that the result should be 8 bits, for the first partial line, the result should be 1110, but we need to add **"sign extentions"** up to MSB, so we add 4 1s in front, the next line we add 3 1s..., so on. If the multiplier bit is 0, the corresponding partial result is all zeros. The last line is different, instead of normal addition of partial result, we need to subtract previous result with it, which can be seen as the addition of the 2's complement of the last partial result.
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
We now encounter another reg array, so another cycle for fetching is needed. And the addres decoding need signed multiplication, but different from before, the multiplier is now an unsigned, or positive, number, and the result is truncated to 11 bits the long multiplication can be simplify as
```
        S 1 1 1 1 1 1 0  <- regfile[rs] (signed 8bit)
 x            0 0 1 1 1  <- imm
------------------------
  S S S S 1 1 1 1 1 1 0  -> mul_data_addr_A (3 sign extention)
  S S 1 1 1 1 1 1 1 0    -> mul_data_addr_B (2 sign extention)
  S 1 1 1 1 1 1 1 0      -> mul_data_addr_C (1 sign extention)
  0 0 0 0 0 0 0 0        -> mul_data_addr_D
  0 0 0 0 0 0 0          -> mul_data_addr_E
```

### STORE 011
```
1: fetch regfile
2: data address partial mult
3: add partial mult
4: store data to cache (or writeback if cache misses)
```

### BEQ 100
```
1: instruction decode
2: change programcounter
```


### JUMP 101
```
1: change programcounter
```

## CNN Accelerator
### Input
To balance between speed and dev time, instead of using cache, which may only be a part of the kernel, weight or image, I read kernel & weight, image1, image2 for total of 3 times. Before the read, if the data cache is smaller than the first 7 blocks of RAM (0-6, which is related to CNN data) and the cache is dirty, writeback is needed. First I input the kernel and weight. This way as the image is valid the accelerator would already have the kernel inside, and since the kernel data is very close to weight data in RAM, we only need single read request. If the data is conjunct, we can use all the data of the read burst, if not, we can just dump data in the middle, only transfer the front and the end.
![Untitled Diagram](https://github.com/user-attachments/assets/51edbcb2-bb14-4aed-b7ea-c6c27a75937f)

### Convolution
I wrote the code in HW3 as what the original convolution definition did in sequence. which is a very inefficient way. This time I learned from the best code and do the calculation as the input sequence, input 0 1 2 3 multiply by the 1st entry of the kernel, added to the 1st row of the result, input 2 3 4 5 multiply by the 2nd entry of the kernel, added to the 1st row of the result, and so on. Which make the register needed for image input to just 4 byte, and the calculation can be separated to multiple cycles easily. The calculation of a single chennel is as such.
<img width="1626" alt="image" src="https://github.com/user-attachments/assets/226528c1-795f-455f-ba37-ea69fb458832" />

### Out of Order Execution & Cache Coherence
Since the result regfile would only be used after thousands of instructions after CNN, we only need to do the input (fetching RAM) with order to ensure cache coherence, and the result will be produced within way less than 100 cycles, so the detection of the result regfile is used or not is not needed. We can execute the following instructions without any problem.

## Further optimization
### Faster way of multiplication
The partial signed value multiplication itself, which is only AND result of 16 bit change the needed clk period from 2.8 ns to 3.4 ns, if we want even faster clock, we can dig more into signed value multiplication and do retiming to perfectly optimize the process to desired clk speed. For the original system without signed value multiplication, the critical path is:
```
program_counter_reg_3_/CK (QDFFRBT)    0.00     0.00
program_counter_reg_3_/Q (QDFFRBT)     0.55     0.55
U8809/O (OR2T)                         0.24     0.79 
U8808/O (NR2F)                         0.17     0.96 
U8004/O (INV12)                        0.10     1.06 
U8804/O (OR2T)                         0.24     1.30 
U8800/O (INV12)                        0.20     1.50  
U12215/O (AOI22S)                      0.14     1.64 
U7536/O (AN4)                          0.30     1.94 
U7297/O (ND3HT)                        0.11     2.05 
U8587/O (NR2T)                         0.11     2.16 
U8585/O (ND3HT)                        0.12     2.29 
U7792/O (ND3HT)                        0.10     2.39 
U7802/O (OAI12H)                       0.09     2.48 
current_inst_pip_reg_11_/D (QDFFP)     0.00     2.48 
```
which is limited by cache, a huge reg array, which needs huge muxes to control it.

### Dual clk system
I tried to implement this in verilog code but failed at gate simulation. I used a reg as the clk signal of the CNN accelerator, which flipped its bit at every posedge clk, making it 2 times slower than the clk signal. I've done this on an FPGA board before but in ASIC design, clk signal must be specified in file so that the edge detection could work perfectly, the only way to do this is using IPs that can change clk speed but they are not allowed here. But this is still a very good approach since this way I can spend lees time on CNN pipelining, keep the 2.8 ns clk period and use less area for CNN calculation.
![Screenshot 2025-06-11 at 02 59 18](https://github.com/user-attachments/assets/9ad725da-2d6c-488e-b352-f71f37393626)
As you can see the edge detection is not working correctly at the edges of clk_NPU, the result may leads to unknown value and further creates hazard. The following code is how I divide the clk speed by a factor of 2.
```
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        clk_NPU <= 1'b1;
    end 
    else begin
        clk_NPU <= ~clk_NPU;
    end   
end
```

## Tests
All three tests are passed and the performance is:
* CLK Period 3.4 ns
* Ext Cycles 122728
* SYN Area   951707.843153
<img width="870" alt="Screenshot 2025-06-14 at 16 19 15" src="https://github.com/user-attachments/assets/01ecc010-b289-43de-885c-bcb8c462e47e" />

