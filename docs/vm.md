# Virtual machine

The story engine is built on a custom virtual machine (VM).
It is stack-oriented and has a simple, concise instruction set.

This document is divided into several parts:

**Part 1** gives a background of how the game's bytecode is organized in the ROM.

**Part 2** describes the "syntax" of the VM.
While some nuance is omitted, enough detail should be captured here to be able to
understand how opcodes are encoded and what they do when executed.

**Part 3** describes the "semantics" of the bytecode contained in MOTHER 3.
Here we introduce the calling convention and higher-level abstractions like
local variables and function arguments.
This section is crucial to being able to construct a high-level language representation
of the bytecode and, ultimately, a decompiler for it.

# Part 1: ROM organization

Bytecode is organized into functions (subroutines).
Each function is scoped to a "room" (literally, corresponding 1:1 with an area of the game's overworld).
All of a room's functions are stored contiguously. There is no "data" section: everything is executable code.

There are 1001 rooms in the game.
The first room, "room 0", is special: it does not correspond to an area of the game's overworld, but instead contains a bunch of common functions that other rooms' functions can call into, like a library of sorts.
Other than room 0, no room's code can call into any other room's code.

Details on how things are physically stored and indexed in the ROM can be found on [Datacrystal](https://datacrystal.tcrf.net/wiki/Mother_3/Game_logic) for now.


# Part 2: Machine syntax

You should be generally familiar with machine code to understand this section: it is assumed that you know what registers are, what a stack is, what a program counter is, etc.

The VM uses an evaluation stack as a means of passing information around and performing operations on it.
This stack has a capacity of 1000 values.
Despite being a stack, there are opcodes that allow somewhat arbitrary direct access to values stored anywhere on the stack: Part 3 will describe why.

The VM's unit of memory is the 32-bit integer: this applies to both bytecode (opcodes) and the stack.
The address space is 16 bits.

The VM has the following 16-bit registers:

* `r0`-`r3`: general purpose
* `sp`: stack pointer
* `pc`: program counter

Finally, the VM is invoked in the context of a room and always knows which room it is executing under.

All integers are encoded little-endian.

## Opcode summary

```
00 xx yy yy        load [rx,yyyy]
01 xx xx xx        push xxxxxx
02 xx yy yy        load rx,yyyy
03 xx yy yy        store [rx,yyyy]
04 00 xx xx        syscall xxxx
05 xx yy yy        call0 yyyy
06 xx yy yy        return0 sp-yyyy
07 xx yy yy        call yyyy
08 xx yy yy        return sp-yyyy
09 00 00 00        exit
0A xx xx 00        mov r1,xxxx
0B xx xx xx        add sp,xxxxxx
0C 00 xx xx        jmp xxxx
0D 00 xx xx        jmpz xxxx

0E 00 00 00        neg
0E 01 00 00        add
0E 02 00 00        sub
0E 03 00 00        mul
0E 04 00 00        div
0E 05 00 00        mod
0E 06 00 00        inc
0E 07 00 00        dec
0E 08 00 00        and
0E 09 00 00        or
0E 0A 00 00        eq
0E 0B 00 00        ne
0E 0C 00 00        lt
0E 0D 00 00        gt
0E 0E 00 00        le
0E 0F 00 00        ge
0E 10 00 00        copy
0E 11 00 00        pop
0E 12 00 00        pop
0E 13 00 00        nop
```

## Detailed opcode descriptions

Assume that the stack grows left-to-right: address 0 is the bottom of the stack, address 1 is the value immediately on top of the bottom of the stack, etc.

Pushing will increment the stack pointer and popping will decrement it.

In the sections that follow, the notation `x'` denotes "the new state of `x`".

### `00 xx yy yy`: `load [rx,yyyy]`

Read value from stack memory at location `rx+yyyy`, and push it to the stack.

`yyyy` can be negative.

```
stack'[sp] <- stack[rx+yyyy]
sp' <- sp+1
```

### `01 xx xx xx`: `push xxxxxx`

Push literal value `xxxxxx` (sign-extended to 32 bits).

```
stack'[sp] <- xxxxxx
sp` <- sp+1
```

### `02 xx yy yy`: `load rx,yyyy`

Push literal value `rx+yyyy`. `yyyy` is sign-extended to 32 bits and can be negative.

```
stack'[sp] <- rx+yyyy
sp' <- sp+1
```

### `03 xx yy yy`: `store [rx,yyyy]`

Pop value from stack, and store it to stack memory location `rx+yyyy`.

`yyyy` can be negative.

```
stack'[rx+yyyy] <- stack[sp-1]
sp' <- sp-1
```

### `04 00 xx xx`: `syscall xxxx`

Execute native system code.
For example: set a flag, move a sprite on the screen, trigger a battle, etc.

There are 256 syscall slots: `xxxx` identifies which slot to use.

A syscall can consume 0 or more values from the stack as _arguments_, and can push 0 or 1 _return values_ to the stack upon completion.

To properly evaluate program flow, you need to know ahead of time how many arguments a syscall consumes and whether it returns anything.
Fortunately the game has a table of argument counts: let the argument count for syscall `xxxx` be denoted `a`.
There is no table of return counts however; this needs to be determined manually.
Let the return count for syscall `xxxx` be denoted `n`.

Then:

```
sp' <- sp - a + n
if n > 0:
    stack'[sp' - 1] <- returned value
```

Remark:

* Stupidly, there are three syscalls that have the _wrong_ argument count stored in the ROM: they consume more values from the stack than the ROM thinks they do. TODO: write more about this, there's a trick we can do to work around it.

### `05 xx yy yy`: `call0 yyyy`

Call function at offset `yyyy` in room 0.

The caller's frame (`rx` and `sp`) are stored to the stack before control is transferred to the callee.
Remarkably, the stack pointer is _not_ incremented: the callee is responsible for not clobbering the caller's frame.

`rx` is identified by `xx+1`, not `xx`: `xx==0` means `r1`.

```
stack'[sp] <- rx
stack'[sp+1] <- pc
rx' <- sp
pc' <- yyyy (context is moved to room 0)
```

### `06 xx yy yy `: `return0 sp-yyyy`

Return from room 0 callee to caller.

Like with `call0`, the register `rx` is identified by `xx+1`.

The caller's frame is restored and the stack is adjusted to account for (consume) the arguments that the caller initally pushed.
`yyyy` encodes how much to adjust the stack and corresponds to how many arguments the function expects.

The value on top of the callee's stack gets saved to the top of the caller's stack after returning and popping the arguments.

```
rx' <- stack[rx]
pc' <- stack[rx+1] (context is moved back to caller's room)
sp' <- rx-yyyy+1
stack[sp'-1] <- stack[sp-1]
```

### `07 xx yy yy`: `call yyyy`

Exact same as `05 xx yy yy`, except that the callee is in the same room as the caller.

### `08 xx yy yy`: `return sp-yyyy`

Exact same as `06 xx yy yy`, except that the callee is in the same room as the caller.

### `09 00 00 00`: `exit`

Terminates VM execution.

Will not return to caller first if called deep in the call stack.

### `0A xx xx 00`: `mov r1,xxxx`

Store literal value `xxxx` to `r1`.

```
r1' <- xxxx
```

### `0B xx xx xx`: `add sp,xxxxxx`

Increment `sp` by `xxxxxx`.

`xxxxxx` is sign-extended to 32 bits and can be negative.

```
sp' <- sp+xxxxxx
```

### `0C 00 xx xx`: `jmp xxxx`

Unconditional jump to offset `xxxx`.

```
pc' <- xxxx
```

### `0D 00 xx xx`: `jmpz xxxx`

Pop the stack.
If the value is zero, jump to offset `xxxx`.

```
sp' <- sp-1
if zero:
    pc' <- xxxx
```

### `0E xx 00 00`: math

Pop some values from the stack, perform an operation on them, and push the result (except for `nop`).

For comparison operations, `0` is false and `1` is true.

* `00 neg`
  * pop `x`
  * push `-x`
* `01 add`
  * pop `x`
  * pop `y`
  * push `x+y`
* `02 sub`
  * pop `x`
  * pop `y`
  * push `y-x`
* `03 mul`
  * pop `x`
  * pop `y`
  * push `x*y`
* `04 div`
  * pop `x`
  * pop `y`
  * push `y/x`
* `05 mod`
  * pop `x`
  * pop `y`
  * push `y%x`
* `06 inc`
  * pop `x`
  * push `x+1`
* `07 dec`
  * pop `x`
  * push `x-1`
* `08 and`
  * pop `x`
  * pop `y`
  * push `x&y`
* `09 or`
  * pop `x`
  * pop `y`
  * push `x|y`
* `0A eq`
  * pop `x`
  * pop `y`
  * push `x==y`
* `0B ne`
  * pop `x`
  * pop `y`
  * push `x!=y`
* `0C lt`
  * pop `x`
  * pop `y`
  * push `y<x`
* `0D gt`
  * pop `x`
  * pop `y`
  * push `y>x`
* `0E le`
  * pop `x`
  * pop `y`
  * push `y<=x`
* `0F ge`
  * pop `x`
  * pop `y`
  * push `y>=x`
* `10 copy`
  * pop `x`
  * push `x`
  * push `x`
* `11 pop`
  * pop `x`
* `12 pop`
  * pop `x`
* `13 nop`
  * (do nothing)


# Part 3: Semantics

While the machine has four general purpose registers, only the first two are ever used: `r0` and `r1`.

Recall that the stack is backed by a shared memory region.
We introduce a notion of **room variables** and **local variables**, which are the things that share that memory.

`r0` can be thought of as a pointer to a fixed set of **room variables** (or global variables): pieces of memory that any function or syscall nested anywhere in the call stack can access.
**`r0` always has a value of zero.**

`r1` can be thought of as a pointer to a fixed set of **local variables**: pieces of memory that the current function can access.

The actual "evaluation stack" logically begins after the local variables.
Code should never pop more values off the evaluation stack than were pushed in the first place, otherwise it would "bleed" into the local variable region.

Top-level functions always begin with `add sp,x` / `mov r1,y`.
Effectively, this means "allocate `y` room variables and `x-y` local variables".
All top-level functions belonging to the same room will use the same value for `y`.

For example, if `r1` is 3 and `sp` is 5, then:

* There are 3 room variables (occupying locations 0, 1, 2)
* There are 2 local variables (occupying locations 3, 4)
* The evaluation stack begins at location 5


