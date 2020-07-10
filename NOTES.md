# Mocha 86k - A 16/32-bit DCPU competitor

**Alpha quality!** - This is a draft specification, and the details of the
instructions, addressing modes, and especially their encoding is subject to
change. There's a summary of ideas for changes to improve the spec at the bottom.
Feel free to send a PR if you have changes to suggest!

This is a prototype design for a 32-bit capable DCPU-16 competitor.
The design is largely inspired by the architecture of the real-world Motorola
68k family.

It aims to be as familiar as possible to the DCPU-16 in terms of programming
model, though there are some important differences.

Generally the operating speed of the Mocha 86k is 1MHz, 1 million instruction
cycles per second. Since even simple instructions usually cost 2-4 cycles, the
operating rate is something like 250k instructions per second, but that can
vary a lot.


## High-level Architecture

Memory address space is a uniform, unsegmented 32-bit space. Each address in
memory refers to a 16-bit word. Note that most machines have a far smaller
memory than the 4GW this allows; most in fact have a 24-bit address bus and are
limited to 16MW of memory.

There are 11 32-bit registers, one 16-bit register, and one single bit flag.

- 8 general-purpose 32-bit registers: `A`, `B`, `C`, `X`, `Y`, `Z`, `I`, `J`.
- 32-bit program counter `PC`
- 32-bit stack pointer `SP`
- 16-bit interrupt address `IA`
- Single flag `Q` to control interrupt queueing

Finally, there is a queue of up to 256 16-bit interrupt messages.


### Endianness

32-bit values are stored in memory in big-endian fashion. That is, if we stored
the 32-bit value `$deadbeef` to memory address `$10000`, then `$10000` contains
`$dead`, and `$10001` contains `$beef`.


### Operand Sizes

Most instructions can work on either 16-bit words or 32-bit "longwords"; this is
indicated with the `.w` and `.l` suffixes on instructions.

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


## Migrating from the DCPU-16

This is a summary of the most substantial differences from programming for the
DCPU-16.

- Most obviously, all registers are 32-bit under the hood. You can do word-sized
  arithmetic without penalty, however.
- Since memory is accessed a word at a time, reading and writing words is faster
  than longwords, if they are sufficient for your purposes.
- Be careful when writing words to registers, as the upper word is not cleared.
  - See "Operand Sizes" above for details.
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
  - Note one downside: binary operands now have only one complex operand; the
    other must always be a simple register.
  - Binary branches, and `SET`, however, still have two complex operands.
- Branching instructions are much more powerful, for several reasons.
  - There are "branching" instructions (eg. `BRE` branch equal) alongside the
    `IFx` "skipping" ones.
  - These are faster, and much more compact than writing
    `IFE blah, blah; SET PC, somewhere_nearby`
  - In addition, there are now four unary branching/skipping instructions, which
    check whether their single operand is zero, nonzero, negative or positive.
    They have the same branching (`BZR` "branch if zero") and skipping (`IZR`
    "if zero") types.
  - Finally, the unary branches have "decrementing" counterparts, that decrement
    the operand's value by one **after** checking the condition. This is great
    for writing loops.
- Hardware is still mostly compatible! `HWN`, `HWQ`, and `HWI` still work the
  same way (mostly; `HWQ` writes the 32-bit IDs to single 32-bit registers).
  Hardware that reads and writes registers touches their lower words, like a `.w`
  instruction.


## Instruction Format

Each instruction begins with a single word. Some instructions have an additional
word following the first, giving further details of the operation. Additionally,
some addressing modes need extra values, which are put after the instruction.

For instructions with two complex operands (`SET` and the `IFx/BRx` family), any
extra words are written for the "right-hand" operand first. That is,
`SET y, x` has the extra words needed for `x`, then those for `y`. Similarly,
`IFE y, x` puts the extras for `x`, then `y`.

Currently instructions take on a few basic shapes:

| Type       | Format              | Notes                                                        |
| :--        | :--                 | :--                                                          |
| 0 argument | `00000000 0000oooo` | eg. `RFI`, `NOP`, `BRK`.                                     |
| Register   | `00000000 00ooorrr` | Register number in `rrr`, eg. `HWN`                          |
| 1 operand  | `0000oooo oLAAAAAA` | `L` usually `1` for longwords, sometimes an extra opcode bit |
| Branches   | `000011oo oLBBBBBB` | Overlaps with the above. `BBBBBB` = left operand             |
|            | `jjjjjjjj jjAAAAAA` | Overlaps with the above. `AAAAAA` = right operand            |
| `SET`s     | `001LBBBB BBAAAAAA` | `L` for longwords; `AAA` moved into `BBB`.                   |
| Binary     | `ooooorrr DLAAAAAA` | `ooooo` opcode, `rrr` register, `AAA` operand                |
|            |                     | `D` set if the complex operand is the destination            |
|            |                     | `L` set for longwords.                                       |


The 6-bit complex operands take many different formats, detailed below.


### Addressing Modes

After the fundamental shift to 32-bit, this is easily the most vital difference
from the DCPU-16. There are now vastly more powerful and flexible ways to access
operands.

This carries two downsides: more to learn, and more bits in the instruction
encoding. Arithmetic operations (eg. `ADD`, `XOR`) are limited to working on a
general-purpose register and a complex operand.

On the other hand, `SET` and the branching instructions can use the full power
of these addressing modes for both their operands.

This table summarizes the addressing modes, and their encoding, and then each
one has a section describing its details. Addressing modes are encoded as
`aaaRRR` where `aaa` is a 3-bit addressing mode number, and `RRR` is either a
register number (`A` = 0, `J` = 7) or a special code.

| Mode                 | Syntax          | Encoding | Register | Width |
| :--                  | :--             | :--      | :--      | :--   |
| Register             | `A`             | `000`    | Reg.     | 0     |
| Reg. indirect        | `[A]`           | `001`    | Reg.     | 0     |
| Reg. postincrement   | `[A]+`          | `010`    | Reg.     | 0     |
| Reg. predecrement    | `-[A]`          | `011`    | Reg.     | 0     |
| Reg. offset          | `[A+lit]`       | `100`    | Reg.     | 1     |
| Reg. index           | `[A,B]`         | `101`    | Reg.     | 1     |
| PC                   | `PC`            | `110`    | `000`    | 0     |
| SP                   | `SP`            | `110`    | `001`    | 0     |
| EX                   | `EX`            | `110`    | `010`    | 0     |
| IA                   | `IA`            | `110`    | `011`    | 0     |
| [SP] (aka `PEEK`)    | `[SP]`          | `110`    | `100`    | 0     |
| Push/pop             | `[SP]+`/`-[SP]` | `110`    | `101`    | 0     |
|                      | `PUSH`/`POP`    |          |          |       |
| Literal 0            | `0`             | `110`    | `110`    | 0     |
| Literal 1            | `1`             | `110`    | `111`    | 0     |
| Absolute word        | `[lit]`         | `111`    | `000`    | 1     |
| Absolute longword    | `[lit]`         | `111`    | `001`    | 2     |
| Immediate word       | `lit`           | `111`    | `010`    | 1     |
| Immediate longword   | `lit`           | `111`    | `011`    | 2     |
| PC-relative indirect | `[PC+lit]`      | `111`    | `100`    | 1     |
| PC-relative index    | `[PC, A]`       | `111`    | `101`    | 1     |
| [SP+word]            | `[SP+lit]`      | `111`    | `110`    | 1     |
| Reserved             |                 | `111`    | `111`    | 0     |

You'll note that many of these addressing modes also exist on the DCPU-16,
though the encoding is different.

#### Concept: Effective Address

Most of these modes defines an "effective address": the address where we are
reading and writing memory.

Some of them, such as `PC` and direct registers, do not have real addresses;
trying to `LEA` or `PEA` one of these operands will load or push `0` instead.

#### Register

The simplest mode: just reading and writing the register. No effective address.

#### Register indirect

Effective address is the value of the register, `r`; reading and writing to `[r]`.

#### Register postincrement

Effective address is `r`. After reading `r` as our effective address, we add 1
(if reading a word) or 2 (for longwords) to `r`.

#### Register predecrement

Before reading `r`, we decrement it by 1 (for a word) or 2 (longword). Then this
new value of `r` is the effective address.

#### Register offset

Read an extra signed word from the instruction. Then the effective address is
`r + lit_sw`.

#### Register indexed

Read an extra word from the instruction, its low 3 bits contain a register
number `r2`. Our effective address is `r + r2`.

#### PC, SP, EX and IA

These four simply read or write the gven register. They do not have an effective
address.

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
and decremented. The result is written to the same effective address, that is at
`[SP]`.)


#### Pick (`[SP+lit]`)

Reads a signed word `lit_sw` from the instruction, and then the effective
address is `SP + lit_sw`. `SP` is unchanged.

#### Literal 0 and 1

As a convenient shorthand, these two common values are available in-line.

No effective address.

#### Absolute word

Read an unsigned word `lit_uw` from the input; it is the effective address.

#### Absolute longword

Read an unsigned longword `lit_ul` from the input; it is the effective address.

#### Immediate word

Read a word `lit_w` from the instruction. If our instruction is 32-bit,
sign-extend `lit_w` to become `lit_l`.

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

Read a word `lit_w` from the instruction. Its low 3 bits contain a register
number `r`. Effective address is `PC + r`.

Note that `PC` will be pointing to just after this word when the effective
address is computed.

### Instruction Timing

Instruction timing is much more complex than in the DCPU-16, due to several
factors. The processor is microcoded internally, and it's really these microcode
operations that account for cycles.

Here are the basic timings:

1 cycle each:

- Reading and writing each **word** of memory.
  - That includes reading instructions, extra words for operands, and any memory
    accesses in the operations.
- An extra for any addition performed by an indexing or offset addressing mode.

Note that there's a 1-cycle "discount" caused by pipelined reading of the next
instruction. However, if the current instruction changes `PC` in any way, that
pipelined read is wasted and the CPU must wait for the fresh read.

Examples:

- `ADD.w B, A` is 2 cycles: 1 to read the operation, and 1 to do the addition.
- `ADD.w B, [A]` is 3 cycles: operation, reading from memory at `[A]`, adding.
- `ADD.w [B], A` is 4 cycles: operation, reading from memory at `[B]`, adding,
  and writing back to memory at `[B]`.
- `ADD.l B, [A]` is 4 cycles: operation, reading from memory at `[A]`, reading at `[A+1]`, adding.
- `ADD.l [B], A` is 6 cycles: operation, reading `[B]`, reading `[B+1]`, adding,
  and writing back to memory at `[B]` and `[B+1]`.


### 0-argument Instructions

Opcode in the bottom 4 bits.

| Opcode | Instruction | Cycles | Details                                  |
| :--    | :--         | :--    | :--                                      |
| `$0`   | `NOP`       | 1      | Does nothing.                            |
| `$1`   | `RFI`       | 4      | Returns from an interrupt handler.       |
| `$2`   | `BRK`       | 2      | Triggers a breakpoint, in dev mode.      |
| `$3`   | `HLT`       | 4      | Halts the CPU until an interrupt occurs. |


### Branch Instructions

There are unary and binary branch operations, and they can either skip the next
non-branch instruction, or simply branch by an offset. In addition, there are
"decrementing" versions of the unary branches.

#### Unary Branches

The unary operations have four conditions.

| Code | Branch | Skip  | Meaning                           |
| :--  | :--    | :--   | :--                               |
| `$0` | `BZR`  | `IZR` | (Branch) if zero                  |
| `$1` | `BNZ`  | `INZ` | (Branch) if nonzero               |
| `$2` | `BPS`  | `IPS` | (Branch) if positive (signed > 0) |
| `$3` | `BNG`  | `ING` | (Branch) if negative (signed < 0) |

The "decrementing" versions (`BZRD`, `IZRD`, etc.) check the condition against
the operand, then decrement it by 1, then branch/skip accordingly.

This is helpful for writing loops. (Checking before decrementing is handy for
loops counting down

- `BZR.w [b], some_label`
- `IZR.l [b]`
- `BZRD.l [b], some_label`
- `IZRD.w [b]`

#### Binary Branches

The binary operations have eight conditions.

| Code | Branch | Skip  | Meaning                               |
| :--  | :--    | :--   | :--                                   |
| `$0` | `BRB`  | `IFB` | (Branch) if `b & a` is nonzero        |
| `$1` | `BRC`  | `IFC` | (Branch) if `b & a` is zero (clear)   |
| `$2` | `BRE`  | `IFE` | (Branch) if `b == a`                  |
| `$3` | `BRN`  | `IFN` | (Branch) if `b != a`                  |
| `$4` | `BRG`  | `IFG` | (Branch) if `b > a`                   |
| `$5` | `BRA`  | `IFA` | (Branch) if `b > a`, signed ("above") |
| `$6` | `BRL`  | `IFG` | (Branch) if `b < a`                   |
| `$7` | `BRU`  | `IFA` | (Branch) if `b < a`, signed ("under") |

All branches occupy two words at least. They all use the same format for the
second word: `oooooooo ooAAAAAA`.

That's a 10-bit signed offset, and a complex operand. It's the only argument for
unary branches, and the right-hand argument for binary ones.

The brancing flavour simply adjusts `PC` by the signed offset, and continues
from there. The CPU lacks any branch prediction, so it loses the pipelining
"discount" when `PC` changes.

If the offset is `0`, that signals that this is the skipping flavour. Any
further branch or skip instructions (unary or binary) are skipped at a cost of 1
further cycle each. The next non-branch instruction is skipped, and execution
continues after it.


### Register Instructions

| Opcode | Instruction | Cycles | Details                                     |
| :--    | :--         | :--    | :--                                         |
| `$0`   | `LNK`       | 2      | "Link" process for stack frames; see below. |
| `$1`   | `ULK`       | 2      | Unlink, undoes a previous `LNK`.            |
| `$2`   | `HWN`       | 2      | Writes the number of hardware devices.      |
| `$3`   | `HWQ`       | 4      | Queries the hardware details, see below.    |

See the section on interacting with hardware later, about `HWN` and `HWQ`.

#### Link and Unlink

This pair of instructions seems quite arbitrary at first glance, but they are
designed to support C-style stack frames efficiently and concisely.

`LNK`, "link", does the following:

1. Decrements `SP` by 2.
2. Stores the current value of `rrr`, the register, at the new `[SP]`.
3. Stores this new `SP` in `rrr`.
4. Reads a signed word from the instruction.
5. Adds it into `SP` (making space for local variables, usually).

`ULK`, "unlink", reverses this process.

1. Restores the value from `rrr` to `SP`.
2. Reads the old `rrr` value from `[SP]`.
3. Increments `SP` by 2.

There's a few important things to note here:

- `rrr` becomes your "frame pointer", and you can use `SP`-relative indexing
  based on it.
- Don't overwrite `rrr`, or restore it first if you do.
- Downstream function calls will save your `rrr` for you if they also use `LNK`.
- Since the old `SP` is saved, you don't need to pop any other values pushed
  to the stack in the meantime.


### Unary Instructions

There are four types of instructions which have one complex operand. They all
have the shape `0000ffoo oLAAAAAA`, where `ff` is the "family" of operations,
`ooo` is the specific operation, `L` is the word/longword flag, and `AAAAAA` is
the operand.

#### Family 0: Immediate arithmetic

Five key arithmetic operations have "immediate" versions, that allow adding an
inline literal (following the instruction) to the addressed operand. Most of
these operations are commutative; for `SUB` the computation is
`operand - immediate`.

Either way the immediate value comes before and extra words for the operand.

If `L` is set, reads a longword from the instruction and does 32-bit arithmetic.
If `L` is clear, reads a word and does 16-bit arithmetic.

| Opcode | Instruction | Cycles | Details                                 |
| :--    | :--         | :--    | :--                                     |
| `$0`   | (none)      | -      | Placeholder for the Reg type above.     |
| `$1`   | `ADD`       | 1      | Adds immediate value to operand.        |
| `$2`   | `SUB`       | 1      | Subtracts immediate vaule from operand. |
| `$3`   | `AND`       | 1      | Bitwise AND.                            |
| `$4`   | `BOR`       | 1      | Bitwise inclusive OR.                   |
| `$5`   | `XOR`       | 1      | Bitwise exclusive OR.                   |
| `$6`   | (reserved)  | -      | Reserved for future use.                |
| `$7`   | (reserved)  | -      | Reserved for future use.                |

#### Family 1: Bit twiddling

Each of these operations reads a word-sized immediate value, regardless of `L`.

The "register" operations expect a register number in the low 3 bits of that
word; the "immediate" ones expect an unsigned word literal.

Either the register or the immediate gives the bit number for the operation. If
`L` is set, the bit number is interpreted modulo 32; if `L` is clear, modulo 16.

These all require 1 cycle.

| Opcode | Mnemonic | Bit Number | Details                                           |
| :--    | :--      | :--        | :--                                               |
| `$0`   | `BTX`    | Immediate  | Toggles the specified bit (`X` for `XOR`)         |
| `$1`   | `BTS`    | Immediate  | Sets the specified bit ("bit set")                |
| `$2`   | `BTC`    | Immediate  | Clears the specified bit ("bit clear")            |
| `$3`   | `BTM`    | Immediate  | Clears the whole operand, so only this bit is set |
| `$4`   | `BTX`    | Register   | As above.                                         |
| `$5`   | `BTS`    | Register   | As above.                                         |
| `$6`   | `BTC`    | Register   | As above.                                         |
| `$7`   | `BTM`    | Register   | As above. ("bit mask")                            |


#### Family 2: Other unary operations

Most of these use the `L` bit normally, but a few don't need it, since their
size is implied by their operation.

| Opcode | `L` | Operation | Details                                                 |
| :--    | :-- | :--       | :--                                                     |
| `$0`   | `0` | `SWP`     | Swaps the low and high words of operand                 |
| `$0`   | `1` | `PEA`     | Pushes effective address of operand to stack            |
| `$1`   | `0` | `EXT`     | Sign-extends the low word of operand to 32 bits         |
| `$1`   | `1` | `INT`     | Triggers a software interrupt with operand message      |
| `$2`   | `L` | `NOT`     | Bitwise negation - reverse all bits                     |
| `$3`   | `L` | `NEG`     | 2's complement negation                                 |
| `$4`   | `L` | `JSR`     | Jump to subroutine - push `PC`, then `PC := operand`    |
| `$5`   | `L` | `IAQ`     | Interrupt queueing bit `:= operand == 0`                |
| `$6`   | `L` | `LOG`     | Log the operand's value to any relevant debug system    |
| `$7`   | `L` | `HWI`     | Send hardware interrupt for the device number `operand` |

The implied sizes are:

- `SWP`: Longword, so it has two words to swap.
- `PEA`: Doesn't actually read the operand; pushes 32-bit effective address.
- `EXT`: Expects a 32-bit operand, ignores its high word and sign-extends the low word.
- `INT`: Reads just a word, since that's the size of interrupt messages.
  - **NB**: If you do `INT [some_longword_var]`, it'll read the high word!

#### Family 3: Binary branches

Branch instructions are discussed in more detail above; this is where they fall
in the instruction encoding.


### `SET` instructions

Set instructions are encoded as: `001LBBBB BBAAAAAA`. That is, if the high
nybble is `0010`, the instruction is a `SET.w`. If `0011`, `SET.l`.

The destination operand is `BBBBBB`, the source is `AAAAAA`. Remember that the
source is processed first, and its extra immediate values (if any) come first.

The source operand is fully processed and the value read, before the effective
address is computed and written for the destination. That can be relevant if
the operands depend on values that are changing as the instruction is processed,
such as `PC` or `SP`.


### Binary instructions

This family of instructions all have the form: `ooooorrr DLAAAAAA`. `ooooo` is
the 5-bit opcode, `rrr` the register operand's number, `D` the "destination" bit,
`L` the common longword bit, and `AAAAAA` the complex operand.

If `D` is set, the complex operation is the destination, and the register the
source. If `D` is clear, the complex operand is the source.

Note that for non-commutative operations (eg. `SUB`, `DIV`) the destination is
the left operand and source the right.

Note that opcode `00000` is a placeholder for all the unary, register, and
nullary ops described above. `0001x` is reserved for future use. `001xx` are the
`SET` opcodes above. The binary operations currently run from `01000` to `10101`.

| Opcode  | Instruction | Cycles | Notes                                          |
| :--     | :--         | :--    | :--                                            |
| `00000` | (reserved)  | -      | Unary, register and nullary operations.        |
| `0001x` | (reserved)  | -      | Reserved for future expansion.                 |
| `001xx` | `SET`       | -      | See above.                                     |
| `01000` | `ADD`       | 1      | `dst := dst + src`, `EX := 1` if carry         |
| `01001` | `ADX`       | 2      | `dst := dst + src + EX`, `EX := 1` if carry    |
| `01010` | `SUB`       | 1      | `dst := dst - src`, `EX := -1` if borrow       |
| `01011` | `SBX`       | 2      | `dst := dst - src + EX`, `EX := -1` if borrow  |
| `01100` | `MUL`       | 4/8    | `EX:dst := dst * src` unsigned multiply        |
| `01101` | `MLI`       | 4/8    | `dst := dst * src` signed multiply             |
| `01110` | `DIV`       | 12/18  | `dst := dst / src`, `EX := dst % src` unsigned |
| `01111` | `DVI`       | 12/18  | `dst := dst / src`, `EX := dst % src` signed   |
| `10000` | `AND`       | 1      | `dst := dst & src`                             |
| `10001` | `BOR`       | 1      | `dst := dst OR src`                            |
| `10010` | `XOR`       | 1      | `dst := dst ^ src`                             |
| `10011` | `SHR`       | 1      | `dst:EX := dst >> src` shifting in 0s          |
| `10100` | `ASR`       | 1      | `dst:EX := dst >>> src` preserving top bit     |
| `10101` | `SHL`       | 1      | `EX:dst := dst << src`                         |

For the shifts, the bits shifted out go into `EX`. Right shifts push the bits
into the top bits of `EX`; treating `EX` here as 32-bit. For left shifts, `EX`
gets the bits shifted out of the (16- or 32-bit) result in its low bits.


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

This corresponds to option 1 above.

Considering all four combinations of signs, we have:

1. `7 / 2 = 3 R 1`; `7 = 2 * 3 + 1`
  - This is trivial, and note that unsigned `DIV` is always this.
2. `(-7) / 2 = -3 R -1`; `-7 = 2 * (-3) - 1` as above.
3. `7 / (-2) = -3 R 1`; `7 = (-2) * (-3) + 1`
4. `(-7) / (-2) = 3 R -1`; `-7 = (-2) * 3 - 1`

So in all four cases the division identity is satisfied, and so is the property
that the quotient is positive **iff** the signs of the operands match.


### Load Effective Address

Sometimes would want to do something complex to memory at the location of an
operand. Perhaps multi-word arithmetic, or accessing a structure. Rather than
forcing the programming to duplicate logic that's already built into the CPU, or
to repeatedly specify expensive indexing operands, you can use `LEA`.

Memory operands have an "effective address" (see above) which they compute,
and then use for both reading and writing that operand as needed.

`SET.w C, operand` loads the **value** at that effective address into `C`.
`LEA C, operand` loads the **effective address itself** into `C`.

Put another way, these two sequences have the same result:

```
SET.w C, operand
```

```
LEA C, operand
SET.w C, [C]
```

(Of course, if that's all you meant to do there's no need to use `LEA`.)


`LEA` is encoded as `11000rrr 00AAAAAA`, which resembles a binary operation but
is missing the `L` and `D` bits. They aren't required here - the effective
address is always 32 bits, and the operand is always the source.


Note that if the operand passed to `LEA` and `PEA` has side effects, such as
adjusting the stack pointer, these effects **do not** happen. The only effect is
to load the address they would operate on into the register.

For example `LEA C, POP` copies the effective address (`SP`) into `C` without
changing `SP`.

### Reserved areas

Large portions of the instruction space are not yet defined. This is great for
the future, since they can be added as an expansion later.


## Interacting with Hardware

This is very similar, by design, to the DCPU-16.

When DCPU-16 hardware talks about using registers like `A` and `X`, that is
understood to be the low word of `A` and `X`. The high word is not changed when
hardware sets a register. (This is the same behavior as `SET.w A, ...`.)

You can query the number of connected hardware devices with `HWN reg`, which
will write the number of devices (up to `$ffff`) to the register.

`HWQ reg` expects a device number (0-based) and populates registers with device
data. This differs slightly from the DCPU-16. The 32-bit device ID is written to
`A`, the 16-bit device version to `C`'s low word, and the 32-bit manufacturer ID
to `X`. `B` and `Y` (which hold the high words of the 32-bit values on DCPU-16)
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

Similarly, you are free to return (manually) from an interrupt handler without
turning queueing off.

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

## Change Ideas

- `SHL` and friends are kind of wasting top-level slots. They could use the same
  encoding as the bit twiddlers, but there's not really space for them.

