# mips-simulator

This is a C++ program that simulates the execution of a MIPS program. The simulator reads a "binary" file that contains hexadecimal representations of the text and initialized data segments. The program utilizes only a subset of the MIPS R2000 instruction set. In general, the program performs the fetch-execute cycle, reading instructions from memory, decoding them, and performing the appropriate operation on "simulated" registers and/or memory.

Since memory cannot be allocated in the simulator as a single linear array, three separate arrays have been allocated for the text, static data and stack segment, respectively. Instructions and static (initialized) data are read from the "binary" file into the corresponding simulator array. It is assumed that the text segment will not exceed 2 kilobytes (2*1024), the static data segment will not exceed 4 kilobytes, and the stack segment will not exceed 2 kilobytes. From the program's perspective, the text segment begins at address `0x00400000` and the static data segment begins at address `0x10010000`. The program expects a valid stack pointer at address `0x7fffefff`, (so the first valid aligned stack address is `0x7fffeffc`, which is the value the stack pointer has when the program begins execution). When the program accesses one of these segments, the appropriate offset into the "simulated" segment is computed.

The hex code accepted by the program may be generated with QtSpim. Also, the results produced by QtSpim may be compared with those produced by this simulator program for evaluating it. These binary input files containing the hex code need to have a marker (DATA SEGMENT) that indicates where the code (text) segment ends and the static data segment begins; the simulator jumps to loading the data into address `0x10010000` at that point. Also, to avoid including all 4K entries of the static data segment, the input file need only list those which are not 0; it needs to list their addresses followed by the data in word-aligned format. As an example, the assembly program "sum.s", which computes the sum of 9 numbers, lists as follows:

```
0x27bdffd8
0xafbf0024
0xafb30020
0xafb2001c
0xafb10018
0xafb00014
0x00001021
0x00008821
0x3c011001
0x34300000
0x3c011001
0x34320024
0x3c011001
0x34330024
0x8e0e0000
0x022e8821
0x34020004
0x00122021
0x0000000c
0x34020001
0x00112021
0x0000000c
0x34020004
0x3c011001
0x34240030
0x0000000c
0x26100004
0x1613fff3
0x00001021
0x8fb00014
0x8fb10018
0x8fb2001c
0x8fb30020
0x8fbf0024
0x27bd0028
0x03e00008
DATA SEGMENT
0x10010000 0x00000023
0x10010004 0x00000010
0x10010008 0x0000002a
0x1001000c 0x00000013
0x10010010 0x00000037
0x10010014 0x0000005b
0x10010018 0x00000018
0x1001001c 0x0000003d
0x10010020 0x00000035
0x10010024 0x54686520
0x10010028 0x73756d20
0x1001002c 0x69732000
0x10010030 0x0a000000
```

The corresponding assembly code is as follows. Note that pseudoinstructions are translated into actual instructions.

```
# Example for CPS 104
# Program to add together list of 9 numbers
.text                           # Code
.align 2
.globl main
main:                           # MAIN procedure Entrance
        subu $sp, 40            #\ Push the stack
        sw $ra, 36($sp)         # \ Save return address
        sw $s3, 32($sp)         #  \
        sw $s2, 28($sp)         #   > Entry Housekeeping
        sw $s1, 24($sp)         #  / save registers on stack
        sw $s0, 20($sp)         # /
        move $v0, $0            #/ initialize exit code to 0
        move $s1, $0            #\
        la $s0, list            # \ Initialization
        la $s2, msg             # /
        la $s3, list+36         #/
                                # Main code segment
again:                          # Begin main loop
        lw $t6, 0($s0)          #\
        addu $s1, $s1, $t6      #/ Actual work
                                # SPIM I/O
        li $v0, 4               #\
        move $a0, $s2           # > Print a string
        syscall                 #/
        li $v0, 1               #\
        move $a0, $s1           # > Print a number
        syscall                 #/
        li $v0, 4               #\
        la $a0, nln             # > Print a string (eol)
        syscall                 #/
        addu $s0, $s0, 4        #\ index update and
        bne $s0, $s3, again     #/ end of loop
                                # Exit Code
        move $v0, $0            #\
        lw $s0, 20($sp)         # \
        lw $s1, 24($sp)         #  \
        lw $s2, 28($sp)         #   \ Closing Housekeeping
        lw $s3, 32($sp)         #   / restore registers
        lw $ra, 36($sp)         #  / load return address
        addu $sp, 40            # / Pop the stack
        jr $ra                  #/ exit(0) ;
.end    main                    # end of program
                                # Data Segment
.data                           # Start of data segment
list:   .word 35, 16, 42, 19, 55, 91, 24, 61, 53
msg:    .asciiz "The sum is "
nln:    .asciiz "\n"
```

## Instructions Simulated

syscall codes 1,4,5,8,10, (read/write int, read/write string, and exit) have been simulated. Floating point and coprocessor instructions have not been simulated. As actual hardware is being simulated, pseudoinstructions (like LI: load immediate, or MOVE) which are translated by the assembler into actual instructions (usually ORI: or immediate in the case of LI) are not simulated — only "real" instructions with an opcode have been simulated. Pseudoinstructions may be used in the input binary files themselves, as the assembler will continue to make all the translations QtSpim would make (please use corresponding opcodes and instruction formats).

The actual gate-level hardware execution of each individual operation has also not been simulated. For example, to simulate ADD r3, r4, r5, the + operator in C++ has been used to perform r3 = r4 + r5, where r3, r4, and r5 are "simulated" registers. Similarly, *, /, -, ^, <<, >>, etc., have been used to "simulate" instruction execution.

Input binaries terminate by using the exit syscall. The entire binary is executed and all the register contents are printed at the end.
