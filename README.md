# RISC V-Architecture implementation on-Circuitverse
### The GS01, a limited RISC-V Computer that can control a RGB matrix - Created on CircuitVerse

![alt text](GS01.gif).

Made by Glenn Corthout & Stijn Verwoerd 

## Index

* [Datapath](#heading-1 "Goto Datapath")
* [RGB Matrix](#heading-1 "Goto RGB Matrix")
* [Memory extension](#heading-1 "Goto Memory extension")
* [Instruction extension](#heading-1 "Instruction extension")
* [Controller](#heading-1 "Controller")
* [Assembly code](#heading-1 "Assembly code")
---

## Datapath

## RGB Matrix

## Memory extension

## Assembly code

### Registers
To use our computer effectively and be able to program it, we will have to assign certain registers to certain tasks

* x10 - ```0x00000000``` | Here the injected controller value gets stored


To be able  to run the program effectively without filling up our very limited memory we have chosen to preset certain registers at the following values: *

The following values get used to compare the injected controller value with.
* x16 - ```0x00000001``` | Represents the Left button
* x17 - ```0x00000002``` | Represents the Right button
* x18 - ```0x00000003``` | Represents the Up button
* x19 - ```0x00000004``` | Represents the Down button

### Memory

We currently only have 16 32-bit memory adresses. 
Address 0x0000003c (position 16) is being used as the memory address where the controller value gets injected.


### Simple color changing routine, increase or decrease the value in row 4
```mips
lw x10, 60(x0)      # load in ctrmem
blt x10, x16, -4    # if ctrmem < 1 go back to start
blt x10, x17, 8     # if ctrmem < 2 go to add -1
blt x10, x18, 20    # if ctrmem < 3 go to add +1

addi x5, x5, -1     # add -1 to x5
sw x5, 16(x15)      # store x5 in Vmem place 16
sw x0, 60(x0)       # store 0 in controller mem slot
blt x0, x17, -28    # go back to start of program

addi x5, x5, 1      # add 1 to x5
sw x5, 16(x15)      # store x5 in Vmem place 16
sw x0, 60(x0)       # store 0 in controller mem slot
blt x0, x17, -48    # go back to start of program
```

### Refactored above code, this is better for more buttons <- write later

```MIPS
# loop
    lw x10, 60(x0)        # 0       Load control memory into x10
    blt x10, x16, 28      # 4       If ctrmem < 1, go to reset
    beq x10, x17, 12      # 8       If ctrmem == 2, go to add +1
# add_minus1
    addi x15, x5, -4      # 12      Subtract 1 from x5
    j 8                   # 16      Jump to store
# add1        
    addi x15, x5, 4       # 20      Add 1 to x5
# store
    sw x5, 16(x15)        # 24      Store x5 in Vmem at offset 16
# reset
    sw x0, 60(x0)         # 28      Store 0 in control memory
    j -32                 # 32      Jump back to start

```