- Notes on my small journey to learning assembly

#Intro to x86-64
- Basic primitives of Intel's x86-64 assembly language
- Construction of basci programs using loops, functions and procedures.
- Using the r2 reverse engineering framework.
- AT&T syntax

Setting AT&T syntax
```
asm.syntax=att

```
- Before an executable file is produced, the source code is first compiled into assembly (.s files) after which it is converted to an object program (.o files), and operations with a linker finally make it executable

###Some useful commands during analysis

```
 r2 -d intro 	# opens the binary in debugging mode
 aa 	# analyzes all symbols and entry points in the executable
 e asm.syntax=att 	#sets the disassembly syntax to AT&T
 ? # general help ,to get specific help about a command type [command]?

```

- Analysis invovles extracting function names, flow control information etc.
- r2 instructions are usually based on a single character

###Useful commands during analysis

```
afl # when you want to know where to start analysis from. Most programs have an entry point defined as main.
This command displays all list of functions

pdf @main # pdf means print disassembly. Doing so will give the memory addresses of instructions, instructions encoded in bytes(machine code) and the human readable instructions
```

-- The core of disassembly involves using registers to do the following

1. Transfer data between memory and the register and vice versa
2. Perform arithmetic operations on registers and data
3. Transfer control to other parts of the program

- Since the architecture is x86-64 the registers are 64 bit and Intel has a list of 16 registers

```
						64 bit		32 bit
						

						%rax       %eax

						%rbx       %ebx

						%rcx       %ecx

						%rdx       %edx

						%rsi       %esi

						%rdi%       edi

						%rsp       %esp

						%rbp       %ebp

						%r8       %r8d

						%r9%       %r9d

						%r10       %r10d

						%r11       %r11d

						%r12       %r12d

						%r13       %r13d

						%r14       %r14d

						%r15       %r15d

```
-Even though the registers are 64 bit, meaning they can hold up to 64 bits of data, other parts of the registers can also be referenced. In this case, registers can also be referenced as 32 bit values as shown. What isn’t shown is that registers can be referenced as 16 bit and 8 bit(higher 4 bit and lower 4 bit). 

- The first 6 registers are shown as general purpose registers.
- The %rsp is the stack pointer and it points to the top of the stack which contains the most recent memory address. 
- %rbp is a frame pointer and points to the frame of the function currently being executed - every function is executed in a new frame

###Some instructions

```
movq source, destination # moving data between registers

This involves
1. Transferring constants (which are prefixed using the $ operator ) eg movq $3 rax # move the constant 3 to the register
2. Transferring values from a register eg movq %rax (%rbx) which means move value stored in %rax to memory location represented by %rbx_


leaq source, destination: this instruction sets destination to the address denoted by the expression in source

addq source, destination: destination = destination + source

subq source, destination: destination = destination - source

imulq source, destination: destination = destination * source

salq source, destination: destination = destination << source where << is the left bit shifting operator

sarq source, destination: destination = destination >> source where >> is the right bit shifting operator

xorq source, destination: destination = destination XOR source

andq source, destination: destination = destination & source

orq source, destination: destination = destination | source

```
The last letter of the mov instruction represents the size of data 

Intel data type Suffix Size (bytes)
bytes				b  			1
Word 				w 			2
Double word 		l 			4
Quad word 			q 			8
Single Precision	s 			4
Double Precision 	l 			8


- When dealing with memory manipulation using registers, there are other cases that might be considered:
1. (Rb, Ri) = MemoryLocation[Rb + Ri]
2. D(Rb, Ri) = MemoryLocation[Rb + Ri + D]
3. Rb, Ri, S) = MemoryLocation(Rb + S * Ri]
4. D(Rb, Ri, S) = MemoryLocation[Rb + S * Ri + D]



