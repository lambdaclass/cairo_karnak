# Cairo compiled program reference

## JSON structure

> WIP

## Instruction encoding

  - Each instruction is a felt (which fits in 64 bits).
  - The prime number must be at least `1 << 64`, otherwise it cannot encode instructions (due to the modulo).

### Encoding

Each instruction has four parts:

  - Flags: A bitset of 16 bits.
  - Offset #2: An offset (`i16`).
  - Offset #1: An offset (`i16`).
  - Offset #0: An offset (`i16`).

They are encoded like this:

    encoded_instruction = 0
        | (flags << 48)
        | ((offset2 + 32768) << 32)
        | ((offset1 + 32768) << 16)
        | ((offset0 + 32768) <<  0);

> TODO: Which values should the offsets have when not in use?

### Flags

The flags bitset can be further divided into several groups:

**Destination register**

  - Bit #0: Set to `1` when the destination is `fp`-relative.

If not set, then the destination is `ap`-relative.

**Operand #0**

  - Bit #1: Set to `1` when the operand #0 is `fp`-relative.

If not set, then the operand #0 is `ap`-relative.

**Operand #1**

  - Bit #2: Set to `1` when the operand #1 is an immediate.
  - Bit #3: Set to `1` when the operand #1 is `ap`-relative.
  - Bit #4: Set to `1` when the operand #1 is `fp`-relative.

If none are set, operand #0 is used. Otherwise, exactly one bit must be `1`.

Alternatively, the following table summarizes the possible combinations:

| #2 | #3 | #4 |     Meaning     |
|----|----|----|-----------------|
|  0 |  0 |  0 | Use operand #0. |
|  1 |  0 |  0 | Immediate.      |
|  0 |  1 |  0 | Register `ap`.  |
|  0 |  0 |  1 | Register `fp`.  |

**Operation**

  - Bit #5: Set to `1` when performing an addition.
  - Bit #6: Set to `1` when performing a multiplication.

If none are set, only operand #1 is used. Otherwise, exactly one bit must be `1`.
If the opcode is `CALL`, then this bits must be set to `0` (unconstrained).

Alternatively, the following table summarizes the possible combinations:

| #5 | #6 |       Meaning        |              Notes               |
|----|----|----------------------|----------------------------------|
|  0 |  0 | Use only operand #1. | Mandatory when opcode is `CALL`. |
|  1 |  0 | Addition.            |                                  |
|  0 |  1 | Multiplication.      |                                  |

**PC Update**

  - Bit #7: Set to `1` when performing an unconditional absolute jump.
  - Bit #8: Set to `1` when performing an unconditional relative jump.
  - Bit #9: Set to `1` when performing a conditional jump.

If none are set, a regular PC update (`pc += 1`) is executed. Otherwise, exactly one bit must be `1`.

Alternatively, the following table summarizes the possible combinations:

| #7 | #8 | #9 |     Meaning     |
|----|----|----|-----------------|
|  0 |  0 |  0 | Use operand #0. |
|  1 |  0 |  0 | Immediate.      |
|  0 |  1 |  0 | Register `ap`.  |
|  0 |  0 |  1 | Register `fp`.  |

> TODO: The conditional jump is always relative?

**AP Update**

  - Bit #10: Set to `1` when adding an arbitrary value to `ap`.
  - Bit #11: Set to `1` when adding one to `ap`.

If none are set, `ap` is not modified, except if the opcode is `CALL`, which will increment it by two. Otherwise, exactly one bit must be `1`.

Alternatively, the following table summarizes the possible combinations:

| #10 | #11 |         Meaning         |                Notes                 |
|-----|-----|-------------------------|--------------------------------------|
|   0 |   0 | Increments `ap` by two. | Mandatory when opcode is `CALL`.     |
|   0 |   0 | Doesn't modify `ap`.    | Behaviour when opcode is not `CALL`. |
|   1 |   0 | Add some value to `ap`. |                                      |
|   0 |   1 | Add one to `ap`.        |                                      |

**Operation code**

  - Bit #12: Set to `1` when `CALL`ing.
  - Bit #13: Set to `1` when `RET`urning.
  - Bit #14: Set to `1` when `ASSERT`ing equality.

If none are set, then a `nop` is executed. Otherwise, exactly one bit must be `1`.

Alternatively, the following table summarizes the possible combinations:

| #12 | #13 | #14 |     Meaning      |
|-----|-----|-----|------------------|
|   0 |   0 |   0 | No operation.    |
|   1 |   0 |   0 | Function call.   |
|   0 |   1 |   0 | Function return. |
|   0 |   0 |   1 | Equalit assert.  |

### Example

**Assert equals**

Source code:

    [ap] = 1, ap++;

Flags:

  0. ☐ Target is `ap`-relative, therefore not setting this bit.
  1. ☒ Ignored since operand #0 is not used. Seems to default to `1`.
  2. ☒ Operand #1 is an immediate.
  3. ☐ Operand #1 is not `ap`-relative.
  4. ☐ Operand #1 is not `fp`-relative.
  5. ☐ No addition, just operand #1.
  6. ☐ No multiplication, just operand #1.
  7. ☐ No absolute unconditional jump.
  8. ☐ No relative unconditional jump.
  9. ☐ No conditional jump.
  10. ☐ Not adding an arbitrary value to `ap`.
  11. ☒ Adding one to `ap`.
  12. ☐ Not `CALL`ing.
  13. ☐ Not `RET`urning.
  14. ☒ `ASSERT`ing equality.
  16. ☐ (Reserved.)

Offset #0: `0`, as in `[ap + 0] = ...`?
Offset #1: `-1`, TODO: why?
Offset #2: `1`, TODO: why?

    // Encode the instruction
    encoded_instruction = 0
        | (0b0100100000000110 << 48)
        | (( 1 + 0x8000) << 32)
        | ((-1 + 0x8000) << 16)
        | (( 0 + 0x8000) <<  0);

    // Write encoded instruction and immediate.
    stream.write(encoded_instruction);
    stream.write(1);

    // Stream should have:
    //   -> 0x480680017FFF8000
    //   -> 1

More examples at [the source code](https://github.com/starkware-libs/cairo-lang/blob/master/src/starkware/cairo/lang/compiler/encode_test.py).

