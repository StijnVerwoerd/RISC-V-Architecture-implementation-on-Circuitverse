# RISC-V Architecture implementation on Circuitverse
### A limited RISC-V Computer that can control a RGB matrix
Made by Glenn Corthout & Stijn Verwoerd 

![alt text](GS01_T.gif)


## Index

* [Dataflow](#dataflow "Goto Dataflow")
* [Instructions](#instructions "Goto Instructions")
* [RGB Matrix](#rgbmatrix "Goto RGB Matrix")
* [Registers](#registers "Goto Registers")
* [Memory](#memory "Goto Memory")
* [Controller](#controller "Goto Controller")
* [Assembly code](#assembly "Goto Assembly code")

### <a id="dataflow"></a>
## Dataflow 
```mermaid

graph TD

    subgraph computer
    reg[[Registers]]
    mem[(512B Memory)]
    ctr[Control]
    io(Controller 
    Input/Output)
    PC(Program Counter)
    ins[Instruction decoder]
    alu[\ALU/]
    end

    subgraph Controller
    but((buttons))
    oi(Controller
    Input/Output)
    end

    subgraph Screen
    pc(Program 
    Counter)
    decode[Position and
    color decoder]    
    vmem[(64B 
    memory)]
    rgb[# RGB #
    Matrix]
    rd(row decoder)
    end

    reg -- jump --> PC
    io --data--> mem
    ctr -- Enable --> reg
    ctr -- Enable --> mem
    ins -- type of instruction --> ctr
    ins -- rs1, rs2, rd --> reg
    ins -- immediate --> PC
    ins -- immediate --> alu
    ctr -- operation --> alu
    mem -- raw instruction --> ins
    PC -- position in memory --> mem
    alu -- value --> reg
    alu -- Data in --> mem
    reg -- rs1, rs2 --> alu
    reg -- address & value --> mem

    io <-- reset & data ---> oi
    but -- L, R, UP, DOWN --> oi

    pc --> vmem
    pc --> rd -- row --> rgb
    reg -- address & value --> vmem
    vmem -- raw data --> decode -- display values --> rgb


```
### <a id="instructions"></a>

## Instructions 

We implemented the following instruction set:


* ADD
* ADDI
* AND
* OR
* BEG
* BEQ
* BLT
* BLTU
* SRL
* SRA
* SLL
* SW
* LW

### <a id="rgbmatrix"></a>

## RGB Matrix 
We use a 16x16 RGB matrix that is supplied by CircuitVerse.

The matrix can be driven by inserting a value into the address range 0x00000200 - 0x0000023c. Every 4 bytes represents 1 row of the screen 
and uses a simple protocol for the data. The data is structured as follows, imagine a 32 bit string divided into 16 2-bit 'cells' that each
tell something about the location and the color assigned to it.

`00000001000000110000000010000000`

| 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| 00 | 00 | 00 | 01 | 00 | 00 | 00 | 11 | 00 | 00 | 00 | 00 | 10 | 00 | 00 | 00 |

In this case pixels 4,8 & 13 would be turned on and 
we defined the colors as such:
* `00` - Black
* `01` - Red
* `10` - Green
* `11` - Blue

each of these colors is then encoded into their respective 24 bit value that the matrix takes as an input for color.

* `111111110000000000000000` - Red
* `000000001111111100000000` - Green
* `000000000000000011111111` - Blue

To cycle through the rows that should be inserted we have created a small program counter that counts 4 at a time (to jump 4 bytes in memory) and that also splits off and decodes into a row selector for the matrix, this way the same row and value in memory are selected simultaneously. 

Let's say we want to set pixel 2 on row 3 to the colour blue, we would use the following instructions:
```t
addi x4, x0, 805306368      # sets the register x4 to the value 00110000000000000000000000000000
sw x4, 12(x15)              # stores the value in x4 to the 3th row in the screen memory
```

### <a id="registers"></a>

## Registers

To use our computer effectively and be able to program it, we will have to assign certain registers to certain tasks

controller:
* x10 - ```0x00000000``` | Here the injected controller value gets stored temporarily
* x16 - ```0x00000001``` | Represents the Left button
* x17 - ```0x00000002``` | Represents the Right button
* x18 - ```0x00000003``` | Represents the Up button
* x19 - ```0x00000004``` | Represents the Down button

Video memory:
* x15 - ```0x00000200``` | This is the starting address of video memory

### <a id="memory"></a>

## Memory 

Currently the computer has total of 128 4-byte addresses with an extra 16 adresses for video memory at address 512+.
This means the computer has a grand total of 18.4kb memory.

```mermaid

graph TD
    subgraph Memory Addresses
    Instructions[[Instructions
    0x00000000
    ---------------->
    0x0000018c]]
    Controller[[Controller
    -----------------
    0x00000190
    -----------------]]
    Presets[[Presets
    0x00000194
    ---------------->
    0x000001FC]]
    Video[[Video memory
    0x00000200
    ---------------->
    0x0000023C]]
    end
    
```









### <a id="controller"></a>

## Controller 

We use a standard address for the controller.

Address ```0x00000190``` (byte address 404) is being used as the memory address where the controller value gets injected.












### <a id="assembly"></a>

## Assembly code 

### Simple color changing routine, increase or decrease the value in row 4
```t
# start 
    lw x10, 60(x0)      # load in ctrmem
    blt x10, x16, -4    # if ctrmem < 1 go back to start
    blt x10, x17, 8     # if ctrmem < 2 go to add -1
    blt x10, x18, 20    # if ctrmem < 3 go to add +1
# left button pressed
    addi x5, x5, -1     # add -1 to x5
    sw x5, 16(x15)      # store x5 in Vmem place 16
    sw x0, 60(x0)       # store 0 in controller mem slot
    blt x0, x17, -28    # go back to start of program
# right button pressed
    addi x5, x5, 1      # add 1 to x5
    sw x5, 16(x15)      # store x5 in Vmem place 16
    sw x0, 60(x0)       # store 0 in controller mem slot
    blt x0, x17, -44    # go back to start of program
```

### Code that can make a pixel move up, down, left or right

```t
# start of the program
    addi x15, x0, 512       # 0         Starting address of video memory
    addi x14, x15, 28       # 4         This will be the starting /address/ of the first pixel
    addi x13, x13, 65536    # 8         This will be the starting /value/ for the first pixel
    addi x4, x0, 4          # 12
# checks if button pressed
    lw x10, 508(x0)         # 16        Load control memory into x10
    beq x10, x0, -4         # 20        If ctrmem == 0, go to start of loop
    beq x10, x16, 16        # 24        If ctrmem == 1, go to Left
    beq x10, x17, 24        # 28        If ctrmem == 2, go to Right
    beq x10, x18, 32        # 32        If ctrmem == 3, go to Up
    beq x10, x19, 56        # 36        If ctrmem == 4, go to Down
# Left    
    sll x13, x13, x4        # 40        Shift pixel left
    sw x13, 0(x14)          # 44        Save the value
    beq x0, x0, 52          # 48        jump to reset
# Right   
    srl x13, x13, x4        # 52        Shift pixel right
    sw x13, 0(x14)          # 56        save the value
    beq x0, x0, 40          # 60        jump to reset
# Up   
    addi x14, x14, -4       # 64        Add -4 to pixel address (move one row up)
    sw x0, 4(x14)           # 68        save 0 in the old row
    sw x13, 0(x14)          # 72        save the pixel value in the new row
    beq x0, x0, 24          # 76        jump to reset

# Down          
    addi x14, x14, 4        # 80        Add 4 to pixel address (move one row down)
    sw x0, -4(x14)          # 84        Save 0 in the old row
    sw x13, 0(x14)          # 88        save the pixel value in the new row
    beq x0, x0, 4           # 92        jump to reset
# reset 
    sw x0, 508(x0)          # 96        Store 0 in control memory
    beq x0, x0, -84         # 100       Jump back to start
```

### Maze game

Place all the maze values directly into memory to be able to load up the game

The following values are in hexadecimal

'MAZE' text:
```
20256A54 
28A44240 
22254850 
20246040 
20246A54
```
Maze:
```
CC000003 
CC003FCF 
CCFF30C3 
C00C3CC3 
FFCCF0CF 
FFCC03CF 
F00FFF03 
F3FC0333 
F000F333 
FFFFF333 
C00F0333 
CCCF3F33 
CCC03033 
CCFFF3F3 
CC0003C3 
FFFFFFCF
```
End of Maze value:
```
DC000003
```
'WIN!' text:
```
101230C4 
10103CC4 
111233C4 
145230C0 
101230C4
```
Maze program
```t
# beginning of screen
    addi x15, x0, 512               # beginning of screen
# maze letters
    lw x31, 472(x0)     #row 5
    sw x31, 16(x15)
    lw x31, 476(x0)     #row 6
    sw x31, 20(x15)
    lw x31, 480(x0)     #row 7
    sw x31, 24(x15)
    lw x31, 484(x0)     #row 8
    sw x31, 28(x15)
    lw x31, 488(x0)     #row 9
    sw x31, 32(x15)
# wait 2 seconds
    addi x2, 10
    addi x1, x1, 1
    blt x1, x2, -8
# values
    add x5, x15, 60                 # set end of screen
    addi x6, x0, 408                # set beginning of maze place in memory
    addi x4, x0, 2                  # steps for bitshift
    lw x3, 404(x0)                   # load up the number that equals end of maze
# load up maze
    add x7, x0, x6                  # start of maze in mem
    add x23, x0, x15                # add the start row of screen memory to x23
    lw x31, 0(x7)                   # load maze value into x31
    sw x31, 0(x23)                  # store maze value on row x23
    addi x23, x23, 4                # make x23 the next row
    addi x7, x7, 4                  # next value in memory of the maze
    blt x23, x5, -16                # if current row is smaller than last row of the screen, go back to load the next value
# load up player position
    addi x21, x31, 0                # load current row
    addi x22, x0, 16                # pos x
    add x21, x21, x22               # add pixel to row value
    addi x23, x15, 60               # pos y
    sw x21, 0(x23)                  # render pixel on screen
# button pressed loop 
    lw x10, 400(x0)                 # Load control memory into x10
    beq x10, x0, -4                 # If ctrmem == 0, go to start of loop
    beq x10, x16, 16                # If ctrmem == 1, go to Left
    beq x10, x17, 36                # If ctrmem == 2, go to Right
    beq x10, x18, 56                # If ctrmem == 3, go to Up
    beq x10, x19, 84                # If ctrmem == 4, go to Down
# left
    lw x27, 0(x23)                  # loads current value into reg x27
    sub x27, x27, x22               # subtracts current position from x27 
    sll x22, x22, x4                # shift the value of the player position 2 bits to the left
    add x27, x22, x21               # add the value of the position and the row together
    sw x27, 0(x23)                  # store the value in the current row
    beq x0, x0, 92                  # jump to reset
# right
    lw x27, 0(x23)                  # loads current value into reg x27
    sub x27, x27, x22               # subtracts current position from x27 
    srl x22, x22, x4                # shift the value of the player position 2 bits to the right
    add x24, x22, x21               # add the value of the position and the row together
    sw x24, 0(x23)                  # store the value in the current row
    beq x0, x0, 68                  # jump to reset
# up
    lw x27, 0(x23)                  # loads current row value into register x27
    sub x27, x27, x22               # subtracts postion x from the current row value
    addi x23, x23, -4               # moves pos y one row up
    lw x21, 0(x23)                  # loads in new row into x21
    add x21, x21, x22               # adds postition x into x21
    sw x27, 4(x23)                  # stores the removed pos x back into vmem
    sw x21, 0(x23)                  # stores the added pos x back into vmem
    beq x0, x0, 36                  # jump to reset
# down
    lw x27, 0(x23)                  # loads current row value into register x27
    sub x27, x27, x22               # subtracts postion x from the current row value
    addi x23, x23, 4                # moves pos y one row down
    lw x21, 0(x23)                  # loads in new row into x21
    add x21, x21, x22               # adds postition x into x21
    sw x27, -4(x23)                 # stores the removed pos x back into vmem
    sw x21, 0(x23)                  # stores the added pos x back into vmem
# reset
    beq x21, x3, 12                 # jump to reset screen
    sw x0, 508(x0)                  # Store 0 in control memory
    beq x0, x0, -140                # Jump back to start
# reset screen
    add x23, x0, x15                # add the start row of screen memory to x23
    sw x0, 0(x23)                   # make the screen black on row x23
    addi x23, x23, 4                # make x23 the next row
    blt x23, x5, -8                 # if current row is smaller than last row of the screen, go back to reset screen
# finish
    lw x31, 492(x0)     #row 5
    sw x31, 16(x15)
    lw x31, 496(x0)     #row 6
    sw x31, 20(x15)
    lw x31, 500(x0)     #row 7
    sw x31, 24(x15)
    lw x31, 504(x0)     #row 8
    sw x31, 28(x15)
    lw x31, 508(x0)     #row 9
    sw x31, 32(x15)
    beq x0, x0, -56                 # jump back to reset screen
```