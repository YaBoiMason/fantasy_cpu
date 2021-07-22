# Unnamed CPU
This is just a shitty fantasy CPU project, I'm writing this out of sheer boredom because a nasty storm just blew in and knocked out internet. 

# Basic Architecture
This is a 16-bit CPU, little endian. It has 8 registers, SC1 (src 1), SC2 (src 2) DS (dest), SP (stack pointer), BP (base pointer), PC(program counter), GP (general-purpose), GP2 (general-purpose 2).
There is the unmodifiable FL dual-word register, which the CPU can only modify the state of (there are instructions that can modify some of these).
The layout of this register is as such:
```
| 0  
| x  
| CARRY FLAG (cf)  
| ZERO FLAG (zf)  
| SIGN FLAG (0 = pos, 1 = negative) (sf)  
| SINGLE-STEP FLAG (calls the trap interrupt defined in your interrupt vector table) (ssf)  
| INTERRUPT ENABLE FLAG (if)  
| PARITY FLAG (pf)  
| Reserved   
| Reserved  
```

There are also the PS registers which show the state of the 16 I/O ports, but this will be covered later.

# Instructions
```
m(register name) [sc1, sc2, ds, sp, bp, pc, gp, gp2, or value literal] ; moves value into register
pin [sc1, sc2, ds, gp, gp2, or value literal] ; gets data from port and puts into DS
put [sc1, sc2, ds, gp, gp2, or value literal] ; gets data from SC1 and sends to port
add ; adds SC1, SC2 and puts it into DS
sub ; subtracts SC1 from SC2 and puts it into DS
mul ; multiplies SC1, SC2 and puts it into DS
div ; divides SC1 by SC2 and puts it into DS
g(flag name) ; gets flag and puts it into DS, 0xF if it's set, otherwise 0x0
gnb [value literal] gets nth bit from SC1 and puts into DS, 0xFFFF if it's set, otherwise 0x0000
mov ; copies the value of address in SC1 to the address of SC2
bor ; bitwise ORs SC1 and SC2 and puts the result in DS
bad ; bitwise ANDs SC1 and SC2 and puts the result in DS
ret ; returns
inz ; if SC1 is not zero, execute next instruction, otherwise skip it
iz  ; if SC1 is zero, execute next instruction, otherwise skip it
s(if, ssf) [0x0, 0xF]; sets flag 
cmp ; if SC1 == SC2 then execute next instruction, otherwise skip it
gth ; if SC1 > SC2 then execute next instruction, otherwise skip it
lth ; if SC1 < SC2 then execute next instruction, otherwise skip it
hlt ; halts until an interrupt is fired
mmem [memory address] ; moves from memory address into DS
```

# Bootstrapping
C is the most widely used language for writing an operating system, and C requires a stack.
This is a small stub program that will set up the CPU and prepare it to call a C function.
```
_loop:
    hlt
    jmp _loop

msp 0x400
; stack top, stack grows downwards
; add additional setup you need here 
call kmain ; calls your kernel main which is linked with this assembly
; if the kernel ever exits, just infinite loop
sif 0x0
jmp _loop
```

# I/O ports
This CPU has 16 I/O ports. You can check the state of each port with the PS registers.
There are two instructions for interfacing with these ports, `pin` and `put`. 
To test the state of port 6:
```
mgp1 0x0
_wait_ready:
    ms1 gp1
    iz
        ret
    ; else
    ms1 mp2
    gnb 0x2
    ms1 ds
    inz
        jmp _ready
    ; else
    jmp _wait_ready
```
To send 0xFFFF to port 6:
```
mgp1 0x0
_send_ready:
    ms1 0xFFFF
    put 0x6
    mgp1 0x1

_wait_ready:
    ms1 gp1
    iz
        ret
    ; else
    ms1 mp2
    gnb 0x2
    ms1 ds
    inz
        jmp _send_ready
    ; else
    jmp _wait_ready
```
To get a byte from port 6:
```
mgp1 0x0
_read_ready:
    ms1 0xFFFF
    pin 0x6
    mgp1 0x1

_wait_ready:
    ms1 gp1
    iz
        ret
    ; else
    ms1 mp2
    gnb 0x2
    ms1 ds
    inz
        jmp _read_ready
    ; else
    jmp _wait_ready
```
