# Mocha 86k - A 16/32-bit DCPU competitor

**Beta quality!** - This is a draft specification, and the details of the
instructions, addressing modes, and especially their encoding is subject to
change. Ideas and suggestions are welcome!

This is a prototype design for a 32-bit DCPU-16 successor.
The design is largely inspired by the architecture of the real-world Motorola
68k family.

It aims to be as familiar as possible to the DCPU-16 in terms of programming
model, though there are some important differences.

## Try it!

My multi-architecture assembler [drasm](https://github.com/shepheb/drasm)
supports Mocha 86k with the `-arch mocha` flag.

Similarly, my multi-architecture emulator
[tc-dcpu](https://github.com/shepheb/tc-dcpu) supports Mocha 86k with the same
`-arch mocha` flag.

## Performance

Generally the operating speed of the Mocha 86k is 2MHz, 2 million clock
cycles per second. Since even modest instructions usually cost 2-6 cycles, the
operating rate is something like 500k instructions per second, but that can
vary a lot.

## High-level Architecture

Memory is a uniform, unsegmented 32-bit address space. Each address in
memory refers to a 16-bit word. Note that most machines have a far smaller
memory than the 4GW this allows. Most in-universe, shipping Mocha 86k machines
have a 24-bit address bus, and so have up to 16MW of memory.

There are 11 32-bit registers, one 16-bit register, and one single bit flag.

- 8 general purpose 32-bit registers: `A`, `B`, `C`, `X`, `Y`, `Z`, `I`, `J`.
- 32-bit program counter `PC`
- 32-bit stack pointer `SP`
- 16-bit interrupt address `IA`
- Single flag `Q` to control interrupt queueing

Finally, there is a queue of up to 256 16-bit interrupt messages.


### Endianness

32-bit values are stored in memory in big-endian fashion. That is, if we stored
the 32-bit value `$deadbeef` to memory address `$10000`, then `[$10000]`
contains `$dead`, and `[$10001]` contains `$beef`.


### Size Suffixes

Every instruction specifies either 16-bit words or 32-bit "longwords"; this is
indicated by most instructions ending in `W` or `L`. Nullary instructions don't
have these suffixes.

A few other instructions (`SWP`, `EXT`) only make sense on longwords, but they
can still be assembled in word form. They don't do anything in that case.

### Words and Longwords

But what happens when reading and writing a 16-bit value to a 32-bit register?

For **reading**, the lower (ie. less significant) 16 bits are read. If register
`C` contains `$deadbeef`, a 16-bit operation would read `$beef`. This applies to
all the registers.

For **writing**, it depends on the register type. For our example, the register
contains `$deadbeef` and we're doing a 16-bit write of `$face`:

- General purpose: writes lower 16 bits, leaves upper 16 bits untouched: `$deadface`
- `PC` and `SP`: Value is zero-extended to 32-bits and overwrites: `$0000face`.
  - This is because `PC` and `SP` are less flexible "address" registers.
- `IA`: This register doesn't actually have an upper 16 bits, so the low 16 bits
  gets written: `$face`.


Note that writing a 32-bit value to `IA` is legal. The upper 16 bits are simply
discarded.

**Be careful!** Reading a 16-bit address into a general-purpose register without
clearing the upper word (typically with `CLRL C`) is a common source of
programming bugs.

## Migrating from the DCPU-16

This is a summary of the most substantial differences from programming for the
DCPU-16.

- Most obviously, all registers are 32-bit under the hood. You can do word-sized
  arithmetic without penalty, however.
- Since memory is accessed a word at a time, reading and writing words is faster
  than longwords, if they are sufficient for your purposes.
- Be careful when loading words into registers, as the upper word is not cleared.
  - See "Words and Longwords" above for details.
- There are no `MOD` and `MDI` instructions; `DIV` and `DVI` put the quotient in
  the destination and remainder in `EX`.
- Multiplication and division are realistically more expensive than addition.
  - Note also that 32-bit multiplication and division is slower than 16-bit.
  - Since the ALU is 32-bit, most other operations (`ADD`, `BOR`, etc.) are as
    fast on longwords as on words.
- All the operands on the DCPU-16 (`A`, `[A]`, `[A + offset]`, `[SP + offset]`,
  etc.) are available, plus several powerful new ones.
  - Of particular mention are `[A]+` and `-[A]`, which read memory based on a
    register with postincrement or predecrement respectively. These replace
    `STI` and `STD` and are much more flexible.
- Branching instructions are much more powerful, for several reasons.
  - There are "branching" instructions (eg. `BRE` branch equal) alongside the
    `IFx` "skipping" ones.
  - These are slightly faster and much more compact than writing
    `IFE blah, blah; SET PC, somewhere_nearby`
  - In addition, there are now four unary branching/skipping instructions, which
    check whether their single operand is zero, nonzero, negative or positive.
    They only have the branching form, not "skipping".
  - Finally, the unary branches have "decrementing" counterparts, that decrement
    the operand's value by one **after** checking the condition. This is great
    for writing loops counting down a length.
- DCPU hardware is fully supported! `HWN`, `HWQ`, and `HWI` still work the
  same way (mostly; `HWQ` writes the 32-bit IDs to single 32-bit registers).
  Hardware that reads and writes registers touches their lower words, like a
  word-sized `setw` instruction.
- Take advantage of the powerful, compact `PSH` and `POP` opcodes for pushing
  and popping multiple registers.


## Instruction Format

Each instruction begins with a single word. Some instructions have an additional
word following the first, giving further details of the operation. Additionally,
some addressing modes need extra values, which are put after (all words of) the
instruction.

Instructions are written here as `SET dest, src`.

To be perfectly clear, the order of words in an instruction is always:

```
main opcode words
(any extra words required by the opcode)
(extra words for the source operand)
(extra words for the destination operand)
```

Branches don't have a "source" and "destination", but they are spelled the
same way: `IFE dest, src`. For non-commutative operations like `IFL`, the
operands are in this same order, as with `SUB`: `IFL dest, src` means
`dest < src`.

There are three basic formats for instructions: nullary, unary and binary. The
unary branches (eg. `BZR`) have an extra word. Finally, some binary instructions
are assembled in "long form", with an extra word giving the opcode.

| Type             | Format              | Notes                                 |
| :--              | :--                 | :--                                   |
| Nullary          | `L0000000 00oooooo` | eg. `RFI`, `NOP`, `BRK`               |
| Unary            | `L000oooo oossssss` | `ssssss` is the single operand        |
| Unary Branch     | `L000oooo oossssss` | Second word is 16-bit signed offset   |
|                  | `bbbbbbbb bbbbbbbb` |                                       |
| Binary           | `Looodddd ddssssss` | `s` is src, `d` is dest               |
| Binary Long Form | `L111dddd ddssssss` | Binary with opcode 7                  |
|                  | `bbbbbbbb bbbooooo` | Real opcode is `o`; branch offset `b` |

The 6-bit operands take many different formats, detailed below.


### Operands

After the fundamental shift to 32-bit, this is easily the most vital difference
from the DCPU-16. There are now more powerful and flexible ways to access
operands.

This table summarizes the addressing modes, and their encoding, and then each
one has a section describing its details. Addressing modes are encoded as
`mmmRRR` where `mmm` is a 3-bit addressing mode number, and `RRR` is either a
register number (`A` = 0, `J` = 7) or a special code.

| Mode                  | Syntax          | Encoding | Register | Extra Words |
| :--                   | :--             | :--      | :--      | :--         |
| Register              | `A`             | `000`    | Reg.     | 0           |
| Reg. indirect         | `[A]`           | `001`    | Reg.     | 0           |
| Reg. postincrement    | `[A]+`          | `010`    | Reg.     | 0           |
| Reg. predecrement     | `-[A]`          | `011`    | Reg.     | 0           |
| Reg. offset           | `[A+lit_sw]`    | `100`    | Reg.     | 1           |
| Reg. index            | `[A,B]`         | `101`    | Reg.     | 1           |
| PC                    | `PC`            | `110`    | `000`    | 0           |
| SP                    | `SP`            | `110`    | `001`    | 0           |
| EX                    | `EX`            | `110`    | `010`    | 0           |
| IA                    | `IA`            | `110`    | `011`    | 0           |
| [SP] (aka `PEEK`)     | `[SP]`          | `110`    | `100`    | 0           |
| Push/pop              | `[SP]+`/`-[SP]` | `110`    | `101`    | 0           |
|                       | `PUSH`/`POP`    |          |          |             |
| Literal 0             | `0`             | `110`    | `110`    | 0           |
| Literal 1             | `1`             | `110`    | `111`    | 0           |
| Absolute word         | `[lit_uw]`      | `111`    | `000`    | 1           |
| Absolute longword     | `[lit_ul]`      | `111`    | `001`    | 2           |
| Immediate word        | `lit_uw`        | `111`    | `010`    | 1           |
| Immediate longword    | `lit_ul`        | `111`    | `011`    | 2           |
| Immediate signed word | `lit_sw`        | `111`    | `100`    | 1           |
| PC-relative indirect  | `[PC+lit_sw]`   | `111`    | `101`    | 1           |
| PC-relative index     | `[PC, A]`       | `111`    | `110`    | 1           |
| [SP+word]             | `[SP+lit_sw]`   | `111`    | `111`    | 1           |

You'll note that many of these addressing modes also exist on the DCPU-16,
though the encoding is different.

Note also that the number of extra words does not depend on the `L` bit for the
instruction! There's nothing wrong (at least, in terms of the encoding) with
writing `ADDW B, 123456`. That will be encoded as a 16-bit add with a 32-bit
literal, which will be read in, and then the lower 16 bits of the literal will
be added to (the low word of) `B`.

In particular, the literal offsets used by the Register offset, and PC- and
SP-relative indirect modes, are always 1 word. The absolute and immediate
word/longword modes are always exactly that. It never depends on the `L` bit.

(For clarity, `ADDW B 123456` is encoded as:
```
00100000 01111011  opcode
00000000 00000001  high word
11100010 01000000  low word
```)

On the other hand, the effect of the incrementing `[A]+` and decrementing `-[A]`
modes **are** impacted by the `L` bit. Since this is a 32-bit read or write,
it reads and writes longwords, and increments/decrements by 2.


#### Concept: Effective Address

Most of these addressing modes define an "effective address": the address where
we are reading and writing memory.

Some of them, such as `PC` and direct registers like `A`, do not have real
addresses; trying to `LEA` or `PEA` one of these operands will load or push `0`
instead.

**Effective address calculations are always 32-bit.** For example,
`SETW [A + 15], 700` does a 32-bit add of `A` with `15` (encoded as a signed
word, which is sign-extended to 32 bits) to produce the effective address, then
writes a word to that address. The instruction's width (`W` or `L`) only applies
to its actual operation.


#### Register

The simplest mode: just reading and writing the register. No effective address.

#### Register indirect

Effective address is the value of the register, `r`; reading and writing to `[r]`.

#### Register postincrement

Effective address is `r`. After reading `r` to construct our effective address,
`r` is incremented by 1 (if reading a word) or 2 (for longwords).

#### Register predecrement

First decrement `r` by 1 (for a word) or 2 (longword). Then this new value of
`r` is the effective address.

#### Register offset

Read an extra signed word `lit_sw` from the instruction. Sign-extend that value
to 32 bits `lit_sl`. Then the effective address is `r + lit_sl` (32-bit).

#### Register indexed

Read an extra word from the instruction, its low 3 bits contain a register
number `r2`. Our effective address is `r + r2` (32-bit).

#### PC, SP, EX and IA

These four simply read or write the given register. They do not have an
effective address.

Note that `SP` and `PC` do not handle 16-bit writes to their low 16 bits. Before
a 16-bit value is written to these two, it is zero-extended to 32 bits.

#### Peek (`[SP]`)

Effective address is `SP`. That is, it manipulates the topmost value on the
stack without modifying `SP`.

#### SP push/pop

When reading, our effective address is `SP` and `SP` is incremented after being
read.

When writing, `SP` is decremented first, then this new `SP` value is the
effective address.

Note that if this mode is used as the destination operand for a read/write
arithmetic instruction (such as `AND`), this is equivalent to `[SP]`/`PEEK`.
(The effective address is that of the read, ie. `SP`. Then `SP` is incremented,
then decremented. The result is written to the same effective address, that is
at `[SP]`, and `SP` is left unchanged.)


#### Pick (`[SP+lit]`)

Reads a signed word `lit_sw` from the instruction, sign-extends to `lit_sl`, and
then the effective address is `SP + lit_sl`. `SP` is unchanged.

#### Literal 0 and 1

As a convenient shorthand, these two common lieral values are available in-line.

No effective address.

#### Absolute word

Read an unsigned word `lit_uw` from the input. Zero-extend to 32 bits; it is the
effective address.

#### Absolute longword

Read an unsigned longword `lit_ul` from the input; it is the effective address.

#### Immediate word

Read a word `lit_uw` from the instruction. If our instruction is 32-bit,
zero-extend `lit_uw` to become `lit_ul`.

No effective address.

#### Immediate signed word

Read a word `lit_sw` from the instruction. If our instruction is 32-bit,
sign-extend `lit_sw` to `lit_sl`.

No effective address.

#### Immediate longword

Read a longword `lit_l` from the instruction. If our instruction is 16-bit, use
the lower 16 bits of `lit_l` as `lit_w`.

#### PC-relative indirect

Read a signed word `lit_sw` from the instruction. Effective address is
`PC + lit_sw`.

Note that `PC` will be pointing to just after this word when the effective
address is computed! This is particularly important for `SET` instructions.

#### PC-relative index

Read a word from the instruction. Its low 3 bits contain a register number `r`.
Effective address is `PC + r`.

Note that `PC` will be pointing to just after this extra word when the effective
address is computed. Normally this is the assembler's problem.

### Instruction Timing

Instruction timing is much more complex than in the DCPU-16, due to several
factors. The processor is microcoded internally, and it's really these microcode
operations that account for cycles.

The fundamental timings are that it takes 1 cycle to:

- Read and write each **word** of memory.
  - That includes reading instructions, extra words for operands, and any memory
    accesses in the operations.
- Basic 32-bit ALU operations (eg. addition, subtraction, bitwise and).
  - That includes operations to compute the effective address for an operand.

The "cycles" counts given in these tables are the fundamental, intrinsic costs
of each instruction, usually 1 or 2. Any other operations to compute effective
addresses, read and write memory, etc. are extra.

Note that there's a 1-cycle "discount" caused by pipelined reading of the next
instruction. However, if the current instruction changes `PC` in any way, that
pipelined read is wasted and the CPU must wait for the fresh read.

Some examples illustrate:

- `ADDW B, A` - 2 cycles (read instruction, 1 ALU op)
- `ADDL B, A` - 2 cycles (read instruction, 1 ALU op)
  - 16- and 32-bit ALU operations are equally fast.
- `ADDW [B], 10` - 5 cycles
  - Read instruction, read literal 10, read `[B]`, add, write `[B]`.
- `ADDL [B], 10` - 7 cycles
  - Read instruction, read literal 10, read `[B]`, read `[B+1]`, add, write
    `[B]`, write `[B+1]`.
  - Note that the shifting the high word from `[B]` into the high word of the
    ALU is built-in and doesn't count for an ALU operation.
- `ADDL [B, 9], 10` - 9 cycles
  - Read instruction, read literal 10, read offset 9, add `B + 9`, read `[B+9]`,
    read `[B+9+1]`, add, write `[B+9]`, write `[B+9+1]`.
  - The effective address for the destination is only computed once, so we only
    add `B+9` once.


### Nullary Instructions

Opcode in the bottom 6 bits.

```
L0000000 00oooooo
```

| Opcode | Mnemonic | Cycles | Details                                          |
| :--    | :--      | :--    | :--                                              |
| `$0`   | `NOP`    | 1      | Does nothing (NB: `NOP` is `$0000`)              |
| `$1`   | `RFI`    | 4      | Returns from an interrupt handler                |
| `$2`   | `BRK`    | 2      | Triggers a breakpoint, in dev mode               |
| `$3`   | `HLT`    | 4      | Halts the CPU until an interrupt occurs          |
| `$4`   | `ULK`    | 2      | C-style frame pointer pop; see `LNK` for details |


#### RFI

To expand, `RFI` reverses the operations the Mocha 86k performs when an
interrupt is triggered (push `PC`, push `A`, `Q` set to block further interrupts).

So executing `RFI` does the reverse: pops `A`, pops `PC`, clears `Q`.


#### HLT

See below on Halt and interrupt queueing for more about halting.


### Unary Instructions

Opcode in bits 11:6, "`src`" operand in the bottom 6:

```
L000oooo oossssss
```

Note that though it's called `src` the operand is sometimes the destination.

| Opcode | Mnemonic | Cycles | Details                                                                              |
| :--    | :--      | :--    | :--                                                                                  |
| `$01`  | `SWP`    | 1      | Swap low and high words of 32-bit op. (If `L=0`, this does nothing.)                 |
| `$02`  | `PEA`    | 1      | Push effective address of operand to stack                                           |
| `$03`  | `NOT`    | 1      | Bitwise negation, swap all bits                                                      |
| `$04`  | `NEG`    | 1      | 2's complement signed negation                                                       |
| `$05`  | `JSR`    | 1      | Push 32-bit `PC`, then `PC := src`                                                   |
| `$06`  | `LOG`    | 2      | Emit `src` to the debug logging console                                              |
| `$07`  | `LNK`    | 2      | C-style stack frame helper, see below                                                |
| `$08`  |          |        |                                                                                      |
| `$09`  | `HWN`    | 2      | Set `src` to the number of connected devices                                         |
| `$0a`  | `HWQ`    | 4      | Queries device `src`; sets `A` to device ID, `X` to manufacturer ID, `C` to version  |
| `$0b`  | `HWI`    | 4      | Sends a hardware interrupt to device `src`                                           |
| `$0c`  | `INT`    | 4      | Software interrupt with message `src` added to the queue                             |
| `$0d`  | `IAQ`    | 1      | Interrupt queueing on if `src != 0`, off if `src == 0`                               |
| `$0e`  | `EXT`    | 2      | Sign-extend the low word of `src` to make a signed 32-bit value (unchanged if `L=0`) |
| `$0f`  | `CLR`    | 0      | Clears `src` (ie. sets it to 0)                                                      |
| `$10`  | `PSH`    | 1      | Push registers based on bitmap argument (see below)                                  |
| `$11`  | `POP`    | 1      | Pop registers based on bitmap argument (see below)                                   |
| `$20`  | `BZR`    | 2/3    | Branch if zero                                                                       |
| `$21`  | `BNZ`    | 2/3    | Branch if not zero                                                                   |
| `$22`  | `BPS`    | 2/3    | Branch if positive (signed > 0)                                                      |
| `$23`  | `BNG`    | 2/3    | Branch if negative (signed < 0)                                                      |
| `$24`  | `BZRD`   | 3/4    | Branch if zero, with decrement                                                       |
| `$25`  | `BNZD`   | 3/4    | Branch if not zero, with decrement                                                   |
| `$26`  | `BPSD`   | 3/4    | Branch if positive (signed > 0), with decrement                                      |
| `$27`  | `BNGD`   | 3/4    | Branch if negative (signed < 0), with decrement                                      |

See below for more details on `LNK` (and `ULK`), `PSH` and `POP`, and the
branches.

#### Push and Pop

These instructions, borrowed from the ARM family, push and pop several registers
from the stack in sequence. All the general purpose registers, `EX` and `PC` are
eligible for pushing, in that order.

The unary argument is interpreted as `000000PE JIZYXBCA`, where `E` is the `EX`
bit, `P` the `PC` bit. Those registers with bits set are the ones pushed and
popped.

That is, `A` is at the lowest address, then `B`, etc. `PC` is always at the
highest address. That enables function calls with `JSR` to be bracketed like this:

```
PSHW {X, Y, A}
; ...
POPW {A, X, Y, PC}
```

Note that the order of arguments in the list is ignored.

Note also that `L` controls whether 16- or 32-bit registers are pushed and
popped. You almost certainly want `PSHL` and `POPL` for normal use, in order to
save and restore the entire register.

In particular, `POPW {..., PC}` isn't safe as a general "return", because it
effectively discards the high word of the original `PC`.

Costs 1 cycle by default, though the runtime is dominated by reading or writing
to the stack.


#### LNK and ULK

This pair of instructions is designed to make C-style frame pointers easy and
efficient. They use `J` as the frame pointer, as follows:

`LNK src`, "link", does the following:

1. Decrements `SP` by 2.
2. Stores the current (32-bit) value of `J`, the frame pointer, at the new `[SP]`.
3. Stores this new `SP` in `J`.
4. Reads the operand `src`, as a signed offset (sign-extended if `L=1`).
5. Adds it to `SP` (making space for local variables, usually).

`ULK`, "unlink", reverses this process.

1. Restores the value from `J` to `SP`.
2. Reads the old `J` value from `[SP]`.
3. Increments `SP` by 2.

There's a few important things to note here:

- `J` becomes your "frame pointer", and you can use `[J - 2]` style indexing
  based on it.
- Don't overwrite `J`, or restore it first if you do.
- Your calling convention should probably make `J` callee-saved.
  - Then `LNK` and `ULK` take care of saving the old `J` for you.
  - And any downstream function calls will save/restore your `J` value (whether
    they use `LNK`/`ULK` or not).
- Since the old `SP` is saved, you don't need to pop any other values pushed
  to the stack in the meantime. `ULK` implicitly pops them all by loading the old
  `SP`.

#### Unary Branches

These branch (ie. conditionally jump to nearby addresses) based on the value of
`src`. The next word in the instruction gives a signed 16-bit offset, relative
to `PC` after reading that offset word.

The "decrementing" versions (`BNZD` etc.) check the condition first, and only
decrement `src` if the condition succeeds. This is very handy for loops that
count down a length. (Start the loop with `BZRDL C, loop_done`, and it will loop
`C` times, leaving `C = 0` at the end.)

Note that the branches take 1 more cycle on failure (ie. not branching) than on
branching, due to breaking the pipeline.

### Binary Instructions

These come in two forms: short and long. The most commonly used ALU operations
(`SET`, `ADD`, `SUB`, `AND`, `BOR`, `XOR`) are available in short form, which
helps keep these ubiquitous operations fast and compact.

The first word has the form

```
Looodddd ddssssss
```

With a 3-bit operand `o`, and two full addressing modes `dst` and `src`.

Long form uses a second opcode word following the first.

#### Short Form Instructions

| Opcode | Mnemonic | Cycles | Details                                               |
| :--    | :--      | :--    | :--                                                   |
| `$0`   |          |        | Placeholder, for the unary ops                        |
| `$1`   | `SET`    | 1      | Set `dst` to `src`                                    |
| `$2`   | `ADD`    | 2      | `dst := dst + src`, `EX := 1` if carry, 0 otherwise   |
| `$3`   | `SUB`    | 2      | `dst := dst - src`, `EX := -1` if borrow, 0 otherwise |
| `$4`   | `AND`    | 1      | `dst := dst AND src`                                  |
| `$5`   | `BOR`    | 1      | `dst := dst OR  src`                                  |
| `$6`   | `XOR`    | 1      | `dst := dst XOR src`                                  |
| `$7`   |          |        | Long form, see below                                  |

#### Long Form Instructions

These have an extra word, where the low 5 bits is the opcode: `bbbbbbbb bbbooooo`.

The upper 11 bits is reserved (all 0s for future expansion), except for the
`IFx`/`BRx` branch instructions, where it's the branch offset (more below).

| Opcode | Mnemonic | Cycles | Details                                                 |
| :--    | :--      | :--    | :--                                                     |
| `$00`  | `ADX`    | 3      | `dst := dst + src + EX`, `EX` set as `ADD`              |
| `$01`  | `SBX`    | 3      | `dst := dst - src + EX`, `EX` set as `SUB`              |
| `$02`  | `SHR`    | 2      | `dst:EX := dst >> src`, shifting in 0s                  |
| `$03`  | `ASR`    | 2      | `dst:EX := dst >> src`, preserving top bit              |
| `$04`  | `SHL`    | 2      | `EX:dst := dst << src`, shifting in 0s                  |
| `$05`  | `MUL`    | 4/8    | `EX:dst := dst * src`, unsigned multiply                |
| `$06`  | `MLI`    | 4/8    | `dst := dst * src`, signed multiply (`EX` unchanged)    |
| `$07`  | `DIV`    | 12/18  | `dst := dst / src`, `EX := dst % src`, unsigned         |
| `$08`  | `DVI`    | 12/18  | `dst := dst / src`, `EX := dst % src`, signed           |
| `$09`  | `LEA`    | 1      | Set `dst` to the effective address of `src` (see below) |
| `$0a`  | `BTX`    | 1      | Toggle bit `src` in `dst`                               |
| `$0b`  | `BTS`    | 1      | Set bit `src` in `dst`                                  |
| `$0c`  | `BTC`    | 1      | Clear bit `src` in `dst`                                |
| `$0d`  | `BTM`    | 1      | Set `dst` to a bit mask with only bit `src` set         |
| `$0e`  |          |        | (Reserved)                                              |
| `$0f`  |          |        | (Reserved)                                              |

| Opcode | Branch | If    | Details                                                |
| :--    | :--    | :--   | :--                                                    |
| `$10`  | `BRB`  | `IFB` | Branch/run next if `dst & src != 0` ("Bits set")       |
| `$11`  | `BRC`  | `IFC` | Branch/run next if `dst & src == 0` ("bits Clear")     |
| `$12`  | `BRE`  | `IFE` | Branch/run next if `dst == src` ("Equal")              |
| `$13`  | `BRN`  | `IFN` | Branch/run next if `dst != src` ("Not equal")          |
| `$14`  | `BRG`  | `IFG` | Branch/run next if `dst > src`, unsigned ("Greater")   |
| `$15`  | `BRA`  | `IFA` | Branch/run next if `dst > src`, signed ("Above")       |
| `$16`  | `BRL`  | `IFL` | Branch/run next if `dst < src`, unsigned ("Less than") |
| `$17`  | `BRU`  | `IFU` | Branch/run next if `dst < src`, signed ("Under")       |

All the branches take 2 cycles on success, 3 on failure. They treat the upper 11
bits of the long form word as a signed 11-bit offset, relative to the `PC` after
reading the long form word and before reading the operands.

If that offset is -1 (all bits set), the instruction is interpreted as a DCPU-16
style `IFx` instruction. If the condition is true, the next instruction is
executed. If the condition is false, the next instruction is skipped.

When "skipping" thus, any further branches (unary or binary) are also skipped,
so only the next "real" instruction is executed. This is a good way to encode
"and" conditions:

```
IFG A, 7
  IFL A, 19
    SET PC, somewhere
```

or equivalently, and more compactly:

```
IFG A, 7
  BRL A, 19, somewhere
```

#### Multiplying

Unsigned multiplies work with `EX`, signed ones don't. (Because you can't split
a 32-bit signed result into two 16-bit words without breaking the signing.)

Signed 16-bit multiply puts the signed 16-bit result into the low word of `dst`.

Unsigned 16-bit creates an unsigned 32-bit result, and places its low word in
the low word of `dst`, and its high word in the low word of `EX`.

Unsigned 32-bit multiply creates an unsigned 64-bit result. The low longword goes
in `dst`, the high longword in `EX`. Thus a full 64-bit multiplication is possible.

Note that 16-bit multiply takes 4 cycles, 32-bit takes 8.


#### Division

Both signed and unsigned division puts the quotient in `dst` and the remainder
in `EX`.

Signed division is tricky, since there are two solutions to the division
identity. If `b / a = q` with remainder `r`, we must have `b = a * q + r`.

But what is `-7 / 2`? There are two solutions:

1. `-3` remainder `-1`: `-7 = 2 * (-3) - 1`
2. `-4` remainder `1`: `-7 = 2 * (-4) + 1`

The two principles are:
- Quotients are positive if the input signs match, negative if they differ.
  - This is the same as multiplication.
- Quotients are rounded towards zero, not negative infinity.
  - This implies the sign of the the remainder matches the dividend.

This corresponds to option 1 above.

Considering all four combinations of signs, we have:

1. `7 / 2 = 3 R 1`; `7 = 2 * 3 + 1`
  - This is trivial, and note that unsigned `DIV` is always this.
2. `(-7) / 2 = -3 R -1`; `-7 = 2 * (-3) - 1` as above.
3. `7 / (-2) = -3 R 1`; `7 = (-2) * (-3) + 1`
4. `(-7) / (-2) = 3 R -1`; `-7 = (-2) * 3 - 1`

So in all four cases the division identity is satisfied. We also see that the
quotient is positive **iff** the signs of the operands match, and that the sign
of the remainder matches the dividend.


### Load Effective Address

The instructions `PEA` and `LEA` take an operand, but rather than actually
accessing its value, they capture the "effective address" of that operand. See
the section on Operands, and "Concept: Effective Address" above.

- `SETL C, operand` puts the *value* of the operand into `C`.
- `LEAL C, operand` puts the *effective address itself* into C.

Put another way, `SETW C, operand` is equivalent to

```
LEAL C, operand
SETW C, [C]
```

(Though it's pointless to write it that way; this is just an illustration.)

Note that if the operand passed to `LEA` and `PEA` has side effects, such as
adjusting the stack pointer, these effects **do not** happen. The only effect is
to load the address they would operate on into the register.

For example `LEAL C, POP` copies the effective address (`SP`) into `C` without
changing `SP`.

Finally, note that `LEAW` is perfectly valid. It loads the low word of the
(32-bit) effective address into the destination's low word.

### Reserved areas

Large portions of the instruction space are not yet defined. This is great for
the future, since they can be added as an expansion later.


## Interacting with Hardware

Nearly identical to the DCPU-16. This is by design.

When DCPU-16 hardware talks about using registers like `A` and `X`, that is
understood to be the low word of `A` and `X` on the Mocha 86k. The high word is
not changed when hardware sets a register. (This is the same behavior as
`SETW A, ...`.)

You can query the number of connected hardware devices with `HWN`, which
will write the number of devices (up to `$ffff`) to the operand.

`HWQ` expects a device number (0-based) and populates registers with device
data. This is the main difference from the DCPU-16. The 32-bit device ID is
written to `A`, the 16-bit device version to `C`'s low word, and the 32-bit
manufacturer ID to `X`.

`B` and `Y` (which hold the high words of the 32-bit values on DCPU-16)
are unchanged here.


You can signal a hardware device with `HWI operand` giving its device number.
Most hardware devices expect various values in registers; the low words of the
registers correspond to the DCPU-16 registers.

## Interrupts

Interrupts are generated by hardware, or by `INT`. Every interrupt has a 16-bit
*message*. When an interrupt is generated, it goes into the queue (even if
queueing is off). This allows for multiple devices to generate interrupts at the
same time without dropping any.

When an instruction is about to be executed, first interrupts are checked. If
interrupt queueing is off, and there is at least one interrupt in the queue, that
interrupt is triggered.

When an interrupt is triggered, the CPU checks `IA`. If `IA = 0`, the interrupt
is discarded and execution continues normally. If `IA != 0`, the interrupt is
processed as follows:

1. Turn on interrupt queueing.
2. Push `PC` (the instruction about to be executed)
3. Push register `A` (32-bit)
4. Set `PC = IA` (zero extended)
5. Set `A` to the interrupt message (zero extended).

Execution then continues at the interrupt handler whose address was in `IA`.

### Notes

- The `RFI` instruction is designed to neatly reverse the above, by popping `A`
  and `PC`, and turning queueing off again.
- Conversely, `RFI` is not magic. You can return manually by doing the same
  process with other instructions.
- Only one interrupt is popped at a time. If `IA = 0`, one interrupt is discarded
  for each instruction executed.
- The maximum queue length is 256. When the 257th interrupt is added, behavior
  is undefined.
- Interrupts are not checked while "skipping" due to an `IFx` instruction, even
  if several instructions are skipped in a row. Interrupts are only checked when
  an instruction is about to be actually executed.
- The interrupt process above consumes 4 cycles, since it writes 2 longwords to
  the stack.

### Nesting Interrupts

Possible, with care. You are free to turn interrupt queueing off from inside an
interrupt handler.

Similarly, you are free to return (without `RFI`) from an interrupt handler
without turning queueing off.

## Debugging

`BRK` triggers a debug breakpoint. If debugging hardware is connected, it will
notice this hardware signal and halt processing so the state of the CPU can be
examined.

`LOG operand` will emit a value to the debug logging hardware, if any. It
silently does nothing if there is no debug hardware.

## Halt and Interrupt Queueing

What happens when `HLT` is executed while interrupt queueing is on?

Short answer: the CPU is blocked forever, and must be hardware reset.

In more detail, `HLT` blocks the CPU until an interrupt is **triggered**, not
**generated**. Hardware devices and `INT` **generate** interrupts that go into
the queue, but it's only when interrupts are **triggered** on the way out of the
queue that the CPU breaks out of `HLT` state.

Since interrupts are never triggered when queueing is on, the CPU is halted
forever.

Note that `HLT` state ends when an interrupt is triggered, even if `IA = 0` and
it gets discarded.

## Initial State

On power up, the state of the CPU is:

- All registers are 0, including `PC`, `IA`, `EX` and `SP`.
  - This implies the first instruction executed is at `$0`.
  - Note that `SP` is decremented before storing, the first value pushed would
    go at `[$ffffffff]`. Since you probably don't have 4GW of memory, you'll
    want to set `SP` to some other value.
- ROM is copied to the beginning of memory.
- All other memory is undefined - it may be 0, but it also may not.
- Interrupt queueing is off, interrupts will flow. But since `IA = 0`, they will
  be discarded.
- All hardware devices are in their initial state, so no memory is mapped.

