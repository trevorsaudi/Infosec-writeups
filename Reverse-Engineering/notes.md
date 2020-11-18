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



###If statements

The general format of an if statement is

```
if(condition){

  do-stuff-here

}else if(condition) //this is an optional condition {


  do-stuff-here

}else {


  do-stuff-here

}
```
If statements use 3 important instructions in assembly

1. ``` cmpq source2, source1```: it is like computing a-b without setting the destination
2. ```testq source2, source1```: it is like computing a&b without setting destination 
3. Jump instructions which are used to transfer control to different instructions and there are different types of jumps

```
Jump type  		Descriptions
jmp 				unconditional
je 					equal/Zero
jne 				Not equal/Not Zero
js 					Negative
jns 				Nonnegative
jg 					Greater
jge 				Greater or Equal
jl   				Less					
jle 				Less or Equal
ja 					Above(unsigned)
jb 					Below(unsigned)
```
- the last 2 values cannot be negative. while signed integers represent both positive and negativ values. 
- Signed integers are represented in the two's complement while unsigned use normal binary calculations


#Analysing megabeets_0x1

##Binary info
- Getting information from the binary using ```rabin2```

```
rabin2 allows extracting information from binary files inlcuding sections, headers, imports, strings, entrypoints, etc. It can then export the output in several formats. rabin2 is able to understand many file formats such as ELF, PE, Mach-O, Java class
```
- Calling rabin2 with the -I flag prints the binary information suchas the OS, language, endianness, architecture, mitigations and more

- We use r2 to examine the program. Verify the entry point address using ie

##Analysis

-radare2 doesn't analyse the file by default because analysis is a complex process that can take a long time especially with large file
- we can use aaa

##Flags

- After the analysis,r2 associates names to interesting offsets in the file such as Sections, Functions, Symbils, and Strings. Thos names are called Flags and can be grouped into 'flag spaces'
- A flag space is a namespace for flags of similar characteristics or type.

- ```fs``` lists flag spaces
- We can choose a flag space using fs <flagspace> and print the flags it contains using f.

- ```fs imports; f``` flags the imports used by the binary
- ``` fs * ``` going back to the default selection of flagspaces(all of them)

##Strings 

- We see r2 flagged some offsets as strings, some sort of variable names. We can have a look at the strings themselves using :
	``` iz ``` - List strings in data sections
	``` izz ``` - Search for strings in the whole binary

 - ``` axt @@ str.*``` axt stands for *analyze x-refs to*
 axt is used to "find data/code references to this address"
 - The special operator ```@@``` is like a foreach iterator sign, used to repeat a command over a list of offsets (see @@?) and ```str.*```is a wild card for all the flags that start with str.

- This combination helps us to list the strings flags and the function name where they are used and the referencing instruction. Make sure to select the strings flagspace(default use ``` fs *```)

##Seeking

- At the start we were at the entry point of the program. The strings listed are all referenced by main. To move on we need to use the seek command represented by ``` s``` .
- Before moving on we can check for other funtions that r2 flagged for us using the ``` afl ``` command. (analyze functions list)
- We see some interesting functions like sym.beet and sym.rot13

##Disassembling

#main function

- Time to look at some sweet assembly. we need to seek to the function using ``` s main``` and then disassemble using 	``` pdf``` 