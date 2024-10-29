---
title: "The NES Internals Series: chapter 1 - The CPU"
series-name: NES Internals
chapter: 1
date: 2024-03-14 15:13:25 +0300
draft: false
---

## Introduction

The NES has a slightly modified version of the MOS 6502 CPU.
This CPU was quite popular back in the day so there is plenty of detailed documentation about it online.

## Terminology

We won't be using any fancy emulation/NES terminology here, just basic computer stuff.
I will explain some terms very simply, if you want a more in depth explanation you can look these things up online.

- **Registers** - small and fast areas of storage the CPU can fetch data from and write data to.

- **RAM** (Random Access Memory) - a type of memory the CPU can read data to and write data from.
It's used mostly to store the stack and global variables.

> **_NOTE:_** In addition to the stack, global variables and runtime data, modern computers and consoles usually
copy code of the game from the harddisk to memory. The NES doesn't do that - it reads each instruction from the cartridge.

- **MMIO** (Memory mapped I/O) - Roughly speaking, this is mapping a certain memory address to a device, so by reading/writing
that address we can transfer data from/to that device. Look it up online for a more detailed explanation.

- **ISR** (short for interrupt service routine) - the function called to handle an interrupt. Different interrupts have different ISRs,
and each game defines its own ISR.

- **Interrupt Vector** - A memory address which contains the address of the ISR of an interrupt.  When an interrupt occurs, the CPU reads the
address from the interrupt vector to get the address of the ISR, and then jumps to that address (Indirect addressing)

## Part 1 - What does a CPU do?

### The fetch-decode-execute cycle:

At the end of the day, CPU is just a component which receives some data, does some calculations, and outputs some data.

Roughly speaking, each cycle the CPU does one of these operations:

- **Fetch** - fetch an instruction
- **Decode** - figure out what that instruction does
- **Execute** - do that instruction

> **_NOTE:_** These stages could be further divided into more stages (a common division is into 7), but for the NES's emulation purposes,
we can simplify it into 3.

### How should I read this, sir?

The binary code the CPU needs to execute follows the following format:

```psuedo
opcode operand1 operand2
opcode operand1 operand2
opcode operand1 operand2

... etc
```

An **Opcode** is a number unique to each operation the CPU performs, so the CPU knows what it should should do,
and how to get the data used by that instruction. Even similar instructions have different opcodes, because they uses different
addressing modes. (for example, the opcode for `AND X` will be different than)

**Operands**, are data that the operation needs as parameters. Somewhat similar to function arguments in high level languages
such as C, Java, Python, etc. Each operation's operand number range from 0
(the minimum amount) to 2 (the maximum amount)

An a **addressing mode**, dictates a certain way the CPU should "treat" an operand as. (IE treat it as a register, treat it as an immediate
value, treat it as an memory address, or even as a memory address which stores a value the CPU wants to retrieve)

For an example, let's look the following code snippet:

```assembly
LDA #$11
STA $2005
LDA $11
```
All that code does is:
1. load the value `$11` into the accumulator
2. store the value in the accumulator at address `$200`.
3. load the value stored at address `$11` into the accumulator

> **_NOTE:_**
> - `#$` marking means "interpret this as a regular hex-form number"
> - `$` marking means "interpret this as a hex-form address number"
> - `#` marking means "interpret this as a regular decimal-form number"


The assembled code will be:

```psuedo
A9 11 8D 05 20 A5 11
```

- the `A9` is the **opcode** for the `LDA (load accumulator with some value)` instruction, using the _immediate_ addressing mode.
- the `11` is first `LDA`s first **operand**
- the `8D` is the **opcode** for the `STA` instruction, using _absolute_ addressing modes, and both
- the `05` is first `STA`s first **operand**
- the `20` is first `STA`s second **operand**.
- the `A5` is the **opcode** for the second `LDA` instruction, using _zero page_
- the `11` is the second `LDA`s first **operand**

> **_NOTE:_** The reason `2005` is stored reversed, as `05 20` is because the 6502 is _little endian_. Look up online what this means!

As we can see, different instructions have different **opcodes**.
And even when the instructions are the same, the different addressing mode result in different **opcodes** and change completely how the CPU should handle the **operands** passed to it.

In our example, the first `LDA` instruction used `11` as a hex-form number that should be loaded to accumulator "as is".
the second `LDA` instruction used `11` as an address which stores the value to be loaded into the accumulator
These 2 interpretations have very different meanings.

Each addressing mode of the CPU is a different way the CPU should interpret the values of the operands. I won't list and explain them all here, but as we can infer from the example above, _immediate_ addressing means "using the raw value as is" and _absolute_ means "use the 2 operands to make up an address"

> **_NOTE:_** The _zero page_ addressing mode is a bit less obvious at first, but it essentially it tells the CPU to fetch the value from the first memory page (page number zero) at offset specified by the operand. By doing that, you don't need to pass the CPU the 0 page explicitly (ie in our example passing `0011`) so the cpu needs to fetch one operand less - which means less execution cycles. This optimization is very useful and can potentially significantly decrease the execution time of a program.

### To summarize
- The basic for-ever loop of the CPU is:
    1. fetch an opcode + operands
    2. find out what instruction that is, what operands it has and how to read them
    3. execute the instruction
    4. repeat

- An **instruction** is a command you can pass to the CPU to make it do something (store this in that, load this from that, jump to this place, etc)
- **addressing modes** are the different ways the CPU can interpret the operands passed to an instruction (store a value at a certain register, store the value pointed by this address into a certain register, etc)
- A combination of a certain **instruction** with a certain **addressing mode** results in an **opcode** - a unique identifier which tells the CPU exactly what it should do.


You can check out all the 6502 instructions and the available addressing modes [here](https://www.masswerk.at/6502/6502_instruction_set.html)

## The ALU

ALU (Arithmetic Logic Unit) - that is the part of the CPU that actually performs the calculations. It takes in data some data, and outputs it out to a register called the accumulator.

<!-- FINISH WRITING HERE -->

## Registers

The 6502 has a few registers:

- **X** - A 1 byte wide utility register the programmer can use to store some data, to be later used in calculations, writing to RAM or MMIO, etc. The main use of this register is as an offset from some fixed address. For example, a programmer could write a simple loop to iterate over every element in a certain address range, with very few lines of code.

- **Y** - Exactly the same as X, although some addressing modes require the use Y instead of X and vise versa.

- **A** (short for Accumulator) - We've already seen that one the section above. A 1 byte register used by the accumulator to output the result of a calculation. The programmer can also use this register for various needs (temporarily storing values, loading these values to memory, etc)

- **PC** (short for Program Counter) - A 16 bit register used to hold the address for the next instruction to be executed. After each instruction the counter gets incremented some amount to point to the next instruction. The incrementation amount differs depending on the opcode, as different instructions and addressing modes accommodate different amounts of space.
(For example in the example above the `lda` instruction only used 1 operand, which is 1 byte and the `sta` instruction used 2 operands, which make up to be 2 bytes)

    Architecually, This register is divided  into 2:
    - **PCL** - the low 8 bits (byte) of **PC**
    - **PCH** - the high 8 bits (byte) of **PC**


- **S** (short for Stack Pointer) - This register holds the address of the memory stack.
The stack is a data structure (don't know what that is? Look it up!) used by the 6502 to save temporary data, or for saving the return address to jump to when a subroutine ends.
Similarly to other systems, the 6502 stack grows downwards.
    - A value is _saved_ on the stack by writing to the address pointed by **S** and _decrementing_ **S** by the size of the written data. Saving a value on the stack is commonly called "_Pushing_"
    - A value is _retrieved_ from the stack by _decrementing_ **S**, and copying the value currently pointed at by **S**. Retrieving a value on the stack is commonly called "_Pulling_" or "_Popping_"

    > **_NOTE:_** When _popping_ a value, we only need to decrement **S** because we don't care that the value is technically "still there" - setting it to 0 is just more work. We simply decrement **S**, thus loosing reference to that value and effectively "deleting" it. (A good analogy would be: If you want to get rid of a banana peel, you don't go through the hassle of burning it or ripping it to shreds - you simply throw it in the garbage and "loose reference" of it)

- **P** (status register) - This registers bits are used as flags to keep track of certain stuff an instruction done. For example, if an `inc` caused A to overflow out of its bounds and caused a reset. If interrupts to the CPU are disabled, if the current instruction caused a value to become negative, etc. For more information, check out [this page](https://www.nesdev.org/wiki/Status_flags)

- **IR** (short for Instruction Register) - This register isn't necessary for emulation behavior, but it is part of the CPU's architecture so I thought I'll include it. This one is very simple - it's a 1 byte register holding the opcode of the current instruction the CPU is handling.

## The clock

The clock is the component of the CPU which tells it what it should do now. The clock ticks indefinitely, and each click tick the CPU moves to the next stage of the fetch-decode-execute cycle.

So it looks something like:

- **fetch**
- _tick_
- **decode**
- _tick_
- **execute**
- _tick_
- **fetch**
- _tick_
- **decode**

... etc

## Interrupts

Say the user pressed a button, or another component of the machine has finished doing something and needs to tell the CPU, or some other important and urgent event that happened that the CPU should respond to.

What can we do?

Well, one of the ways is using interrupts.
interrupts are essentially, as their name suggests - signals sent to the CPU to interrupt its current execution.
they cause the CPU to stop whatever its currently doing, and respond to the new event that just happened.

Interrupts could be very costly, as they make the CPU drop everything its doing, save what it was doing (so later when it finished handling the event it knowns where it stopped off), and take care of the event. Then when its done, it reads the data it saved to "remember" what it was doing before the interrupt, and continues execution.
So interrupts should used only when really necessary.

The 6502 has 2 different kinds of interrupts:

- IRQ (Interrupt ReQuest) - A kind of interrupt used by mappers for example. This pin is mostly used for lower priority events, as the CPU can ignore them by setting the _Interrupt Disable_ flag in the P register.
It pushes the P and PC registers onto the stack, and writes the IRQ interrupt vector (which is on addresses `$fffe` and `$ffff`) to PC.

- NMI (Non Maskable Interrupt) - As the name implies, this is an interrupt the CPU cannot ignore. It is used for more urgent and important events. A use of this interrupt is by the PPU for example: when it enters the first vblank scanline (don't worry, we will talk about it in the PPU internals write up)
It pushes the P and PC registers onto the stack, and writes the NMI interrupt vector (which is on addresses `$fffa` and `$fffb`) to PC.

> **_NOTE:_** I'm writing this clarification, because I remember it made me confused: Some people say there is another interrupt, called BRK (or software interrupt). This is not entirely true. BRK is an instruction of the 6502, which lets the programmer generate an interrupt from the games code, but in reality, all it does is cause an IRQ interrupt, with the special addition of setting the `B` flag on the P register to 1.

## The internal buses

This section isn't really relevant for good emulation, but it is part of the architecture and I think it's important enough to include it here.

In general, computer buses connect some components together and allows them to communicate with each other.

So far, we've talked about the many components of the NES (The registers, the clock, the ALU, etc) which together form the entity called "The CPU". As you might have guessed, this is the job of the CPUs internal buses.
The 6502 has 2 buses:
- The **Internal Data Bus** - which is used to transfer data between the different components of the CPU. A component writes to that bus, and a different component reads from it.
- The **Internal Address Bus** - which is used to transfer addresses between the different components of the CPU. A component writes to that bus, and that address is read by a different component.

The internal buses are part of the process of communicating with the system bus that connects all devices on the system. (the internal data bus is connected to the system data bus, and the internal address bus is actually PART of the system address bus)

## Final words

Hope this writeup helped understanding how the CPU of the NES works. I know I say this every writeup, but even though it is very hard to understand as a beginner, I can't recommend the NESdev wiki enough. It contains everything emu devs or game devs of the NES need to know
