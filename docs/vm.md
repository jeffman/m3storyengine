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
All of a room's functions are stored contiguously.
There is no "data" section in the ROM: everything is executable code.

Each room has a pointer table that indexes some (not all) of the room's functions: these functions are called _entry functions_ or _entries_.
They are top-level functions that are invoked first when interacting with an NPC, for example.
These entry functions can call into other, non-entry functions from the same room (or room 0).

The first five entries for each room are special in how they are triggered.
(TODO: add more detail)

There are 1001 rooms in the game.
The first room, "room 0", is special: it does not correspond to an area of the game's overworld, but instead contains a bunch of common functions that other rooms' functions can call into, like a library of sorts.
Other than room 0, no room's code can call into any other room's code.

Details on how things are physically stored and indexed in the ROM can be found on [Datacrystal](https://datacrystal.tcrf.net/wiki/Mother_3/Game_logic) for now.


# Part 2: Machine syntax

You should be generally familiar with machine code to understand this section: it is assumed that you know what registers are, what a stack is, what a program counter is, etc.

The VM's word size is 32 bits.
The address space is 16 bits.

The VM has the following 16-bit registers:

* `r0`-`r4`: memory base registers
* `sp`: stack pointer
* `pc`: program counter

The VM has 1000 words of data memory, in a separate address space from code.

Data memory is treated like a stack: most opcodes push and/or pop values to/from the stack.
Accordingly, this memory is named `stack`.
The notation `stack[x]` means "the value stored at location `x` in data memory".

Memory/stack location 0 is the bottom of the stack.
`sp` refers to the location just above the top of the stack: the next pushed value is stored to location `sp`.

Some opcodes can access arbitrary stack values, effectively treating the stack as RAM.
For example, `load [rx,yyyy]` will read a word from `stack[rx+yyyy]` and push it to the top of the stack.
(This is why the registers `r0`-`r4` are called "memory base registers", because they are used as a base offset for such opcodes.)
This mechanism is used to implement variables: Part 3 goes into detail.

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

Pushing will increment the stack pointer and popping will decrement it.

In the sections that follow, the notation `x'` denotes "the new state of `x`".

### `00 xx yy yy`: `load [rx,yyyy]`

Read value from stack memory at location `rx+yyyy`, and push it to the stack.

`yyyy` can be negative.

```
stack'[sp] <- stack[rx + yyyy]
sp' <- sp + 1
```

### `01 xx xx xx`: `push xxxxxx`

Push literal value `xxxxxx` (sign-extended to 32 bits).

```
stack'[sp] <- xxxxxx
sp` <- sp + 1
```

### `02 xx yy yy`: `load rx,yyyy`

Push literal value `rx + yyyy`. `yyyy` is sign-extended to 32 bits and can be negative.

```
stack'[sp] <- rx + yyyy
sp' <- sp + 1
```

### `03 xx yy yy`: `store [rx,yyyy]`

Pop value from stack, and store it to stack memory location `rx + yyyy`.

`yyyy` can be negative.

```
stack'[rx + yyyy] <- stack[sp - 1]
sp' <- sp - 1
```

### `04 00 xx xx`: `syscall xxxx`

Execute native system code.
For example: set a flag, move a sprite on the screen, trigger a battle, etc.

There are 256 syscall slots: `xxxx` identifies which slot to use.

A syscall can consume 0 or more values from the stack as arguments, and can push 0 or 1 return values to the stack upon completion.

To properly evaluate program flow, you need to know ahead of time how many arguments a syscall consumes and whether it returns anything.
Fortunately the game has a table of argument counts at `$8D2D658`: let the argument count for syscall `xxxx` be denoted `a`.
There is no table of return counts however; this needs to be determined manually.
Let the return count for syscall `xxxx` be denoted `n`.

Then:

```
sp' <- sp - a + n
if n > 0:
    stack'[sp' - 1] <- returned value
```

Remark:

* Stupidly, there are three syscalls that have the _wrong_ argument count stored in the ROM: 0x43, 0x4C, 0xCF.
Each one of them consumes one more argument from the stack than what the game thinks it does.
Because the game tries to adjust the stack upon return by popping `a` arguments, you'll end up with the first pushed argument still on the stack.
This is effectively the same as pretending that each of those syscalls returns their first argument.


### `05 xx yy yy`: `call0 yyyy`

Call function at offset `yyyy` in room 0.

The caller's frame (`rx` and `sp`) are stored to the stack before control is transferred to the callee.
Remarkably, the stack pointer is _not_ incremented: the callee is responsible for not clobbering the caller's frame.

`rx` is identified by `xx + 1`, not `xx`: `xx == 0` means `r1`.

```
stack'[sp] <- rx
stack'[sp + 1] <- pc
rx' <- sp
pc' <- yyyy (context is moved to room 0)
```

### `06 xx yy yy `: `return0 sp-yyyy`

Return from room 0 callee to caller.

Like with `call0`, the register `rx` is identified by `xx + 1`.

The caller's frame is restored and the stack is adjusted to account for the arguments that the caller initally pushed.
`yyyy` encodes how much to adjust the stack and corresponds to how many arguments the function expects.

The value on top of the callee's stack gets saved to the top of the caller's stack after returning and popping the arguments.

```
rx' <- stack[rx]
pc' <- stack[rx + 1] (context is moved back to caller's room)
sp' <- rx - yyyy + 1
stack[sp' - 1] <- stack[sp - 1]
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
sp' <- sp + xxxxxx
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
sp' <- sp - 1
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
  * push `x + y`
* `02 sub`
  * pop `x`
  * pop `y`
  * push `y - x`
* `03 mul`
  * pop `x`
  * pop `y`
  * push `x * y`
* `04 div`
  * pop `x`
  * pop `y`
  * push `y / x`
* `05 mod`
  * pop `x`
  * pop `y`
  * push `y % x`
* `06 inc`
  * pop `x`
  * push `x + 1`
* `07 dec`
  * pop `x`
  * push `x - 1`
* `08 and`
  * pop `x`
  * pop `y`
  * push `x & y`
* `09 or`
  * pop `x`
  * pop `y`
  * push `x | y`
* `0A eq`
  * pop `x`
  * pop `y`
  * push `x == y`
* `0B ne`
  * pop `x`
  * pop `y`
  * push `x != y`
* `0C lt`
  * pop `x`
  * pop `y`
  * push `y < x`
* `0D gt`
  * pop `x`
  * pop `y`
  * push `y > x`
* `0E le`
  * pop `x`
  * pop `y`
  * push `y <= x`
* `0F ge`
  * pop `x`
  * pop `y`
  * push `y >= x`
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

## Data memory layout

TL;DR:

* `load/store [r0,x]` means "load/store room variable `x`"
* `load/store [r1,x]` means "load/store local variable `x`"
* `load [r1,-x]` means "load argument `x`" (right-to-left order)
* `load r0,x`/`load r1,x` means "load pointer to room/local variable `x`"

---

While the machine has five memory base registers, only the first two are ever used: `r0` and `r1`.

Recall the following:

* The stack can be accessed at arbitrary locations by the `load` and `store` opcodes
* The locations being accessed are always relative to `r0` or `r1`
* You can modify `r1` with `mov`, but you cannot modify `r0`
* When calling a function:
  * The caller's `r1` and `pc` are saved to the stack, but `sp` is not incremented
  * The callee's `r1` is initialized to `sp`

In fact, `r0` is always zero: the VM initializes it to zero and there's no way for an opcode to modify it.
So `load [r0,yyyy]` and `store [r0,yyyy]` effectively mean "load/store the `yyyy`th word on the stack".

So what does `r1` mean?
When looking through the game's code, the following observations are made:

1. Every entry function begins with the pattern `add sp,x; mov r1,y`, where `x >= y`
1. Every non-entry function begins with the pattern `add sp,z`, where `z >= 2`
1. `mov r1` only appears in entry functions
1. In any given room:
    1. All entry functions have the same value for `y`
    1. For every `load/store [r0,u]`: `u < y`
    1. For every `load/store [r1,v]` in an entry function: `v < (x - y)`
    1. For every `load/store [r1,w]` in a non-entry function, where `w > 0`: `2 <= w < z`
    1. For every `load/store [r1,w]` in a non-entry function, where `w < 0`: `1 <= -w <= t` where `t` is the number of arguments to that function, and that function returns with `return sp-t`

We can draw the following conclusions:

1. Within the scope of a function, the stack has two implied regions of memory whose base pointers are `r0` and `r1` respectively
1. Any function can access variables based in the `r0` region, since `r0` is always zero and never changes
1. Load/store opcodes relative to `r0` never "cross over" into the region based at `r1`, and vice versa
1. Each function gets its own `r1` that's initialized to `sp` at the time of the function getting called
1. No function can access variables in the `r1` region of another function
1. `load [r1,-x]` refers to values that were pushed to the stack immediately before calling a non-entry function

Finally, we define three convenient abstractions:

1. `r0` points to **room variables**: every function in a room can access them, and every function in a room has the same notion of how many room variables have been allocated
1. `r1` points to **local variables**: every function has its own `r1`, and every function allocates local variables with `add sp`
1. In a non-entry function, `load` relative to `r1` with a negative offset refers to **function arguments**

The actual "evaluation stack" logically begins after the local variables.
Code should never pop more values off the evaluation stack than were pushed in the first place, otherwise it would "bleed" into the local variable region.

### Example

Consider this entry function snippet:

```
0:  add   sp,5
1:  mov   r1,3
2:  push  a
3:  store [r0,0]
4:  push  b
5:  store [r0,1]
6:  push  c
7:  store [r0,2]
8:  push  d
9:  store [r1,0]
A:  push  e
B:  store [r1,1]
C:  push  aa
D:  push  ab
E:  push  ac
```

After running this snippet:

* `r0`: `0`
* `r1`: `3`
* `sp`: `8`
* `pc`: `F`
* Stack memory: `{ [r0] a, b, c, [r1] d, e, aa, ab, ac, [sp] }`
* Room variables: `{ a, b, c }`
* Local variables: `{ d, e }`
* Evaluation stack: `{ aa, ab, ac }`

Now suppose it calls a function at `0xABCD`:

```
F:  call  0xABCD
```

Immediately after `call` is executed and control has been transferred to the callee:

* `r0`: `0`
* `r1`: `8`
* `sp`: `8`
* `pc`: `ABCD`
* Stack memory: `{ [r0] a, b, c, d, e, aa, ab, ac, [r1] [sp] 3, 10 }`
* Room variables: unchanged
* Local variables: `{ }`
* Evaluation stack: `{ }`

This isn't a good state to be in: the `3` and `10` in stack memory, which are the caller's stack frame, are liable to be overwritten if the callee at `0xABCD` starts pushing stuff!

By convention, the callee is responsible for preserving the `3` and `10` sitting in stack memory by allocating them as local variables and never accessing them.
Therefore the callee at `0xABCD` must start with `add sp,x` where `x >= 2`, and every `load/store [r1,y]` must have `2 <= y < x`.

Suppose the callee starts like this:

```
ABCD:  add sp,6
```

Now the VM is in a good state:

* `r0`: `0`
* `r1`: `8`
* `sp`: `E`
* `pc`: `ABCE`
* Stack memory: `{ [r0] a, b, c, d, e, aa, ab, ac, [r1] 3, 10, _, _, _, _, [sp] }`
* Room variables: unchanged
* Local variables: `{ 3, 10, _, _, _, _ }`
* Evaluation stack: `{ }`

The caller's stack frame are effectively the same as local variables 0 and 1.
So a non-entry function should never attempt to access those.
If a non-entry function wants `N` local variables of its own, it must start with `add sp,(N+2)`: in this case, `N` is 4.