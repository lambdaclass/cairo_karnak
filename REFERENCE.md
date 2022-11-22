# Cairo compiled program reference

## JSON structure

The compiled JSON program structure is composed of the following fields:

  - `attributes`: ???.
  - `builtins`: List of the used builtins' ids (array of strings).
  - `compiler_version`: Cairo version used to compile the program (`<major>.<minor>.<patch>`).
  - `data`: Program bytecode and immediates (array of `BigInt`s). The instructions (bytecode) fit
      within 64bit integers, but immediates are `BigInt`s.
  - `debug_info`: Has [its own section](#debug-information).
  - `hints`: Has [its own section](#hints).
  - `identifiers`: Has [its own section](#identifiers).
  - `main_scope`: Path to the scope containing the main function.
  - `prime`: `BigInt` containing the prime number used in the program.
  - `reference_manager`: Has [its own section](#references).

**Useful types**

  - Memory address: An object with attributes `group` and `offset`, that correspond to a segment
      index and offset respectively.
  - Source code span: Starting and ending location in a source file. Has the following fields:
    - `end_col`: Final column.
    - `end_line`: Final line.
    - `input_file`: An object with a `filename` attribute which contains the absolute path to the
        source file.
    - `start_col`: Starting column.
    - `start_line`: Starting line.
  - Source code span with parent locations: The same as a source code span, with an optional field
      named `parent_location`. This field is a tuple containing itself (recursive) and a string
      describing the scope (in a human-friendly manner).

### Debug information

The debug information object is composed of the following fields:

  - `file_contents`: ???
  - `instruction_locations`: Explained below.

**Instruction locations**

The field `instruction_locations` is a dictionary whose keys map to the indices of instructions on
the `data` field (never to immediates). Its value has the following fields:

  - `accessible_scopes`: A list of scopes that can be accessed from the instruction's location.
  - `flow_tracking_data`: Not sure why it's necessary. Has the following fields:
    - `ap_tracking`: A memory address.
    - `reference_ids`: A dictionary whose keys are paths to the reference (scope + `.` + name) and
        its values point to a reference from the reference manager.
    - `hints`: An object containing a `location`, in the form of a source code span, and
        `n_prefix_newlines`, which is a number (not sure what it represents).
    - `inst`: Span of the instruction, including parent locations.

### Hints

The `hints` attribute contains a dictionary whose keys map to the indices of instructions on the
`data` field (same as with [instruction locations](#debug-information)), while its value are arrays
specifying the hints used (?) by the associated instruction.

Each hint has the following fields:

  - `accessible_scopes`: The scopes that are accessible from the hint's Python code.
  - `code`: The hint's Python code.
  - `flow_tracking_data`: Not sure why it's necessary. Has the following fields:
    - `ap_tracking`: A memory address.
    - `reference_ids`: A dictionary whose keys are paths to the reference (scope + `.` + name) and
        its values point to a reference from the reference manager.


### Identifiers

The `identifiers` attribute is a dictionary whose keys are the identifier IDs (as referenced by the
`flow_tracking_data.reference_ids`'s items). Its values may represent be one of several things. The
field `type` may be used to distinguish between the different identifier kinds.

**Aliases**

Its `type` is `alias`.

An alias just forwards an identifier into a different one. It has a single extra field named
`destination`, whose value is an identifier ID (string).

**Constant values**

Its `type` is `const`.

A constant value identifier has a single extra field named `value`, which contains the integer
(bigint) value.

**Functions**

Its `type` is `function`.

The `function` identifier only declares the function's entrypoint (`pc`) and its decorators. Its
arguments, implicit arguments, return type and size of its locals are all independent identifiers
with IDs `<own-id>.Args`, `<own-id>.ImplicitArgs`, `<own-id>.Return` and `<own-id>.SIZEOF_LOCALS`
respectively. Therefore, it only has the following fields:

  - `decorators`: ???.
  - `pc`: A number indicating the function's first instruction offset (as is in the `data` field).

**Memory references**

Its `type` is `reference`.

Memory references group the `let` bindings, `let local`, `tempvar`... They have the following
format:

  - `cairo_type`: The binding type (TODO: TODO: is this field the same `cairo_type` as in a
      `type_definition`?).
  - `full_name`: Full path to the binding (`<scope-path>.<binding-name>`).
  - `references`: List of referenced values.

The `references` list contains every referenced value for this identifier. Normally, the list is
only one item long; however, creating a reference with the same name (therefore same identifier)
will cause it to grow. Each reference has the following fields:

  - `ap_tracking_data`: Not sure why it's necessary. A memory address.
  - `pc`: The `pc` after which said reference is valid in the current scope.
  - `value`: A Cairo string defining both the value and the type.

**Structures**

Its `type` is `struct`.

Has the following extra fields:

  - `full_name`: Full path to the struct type, as defined by the language.
  - `members`: Dictionary mapping field names to its types.
  - `size`: Size (in felts) of the struct.

Each member is an object composed of:

  - `cairo_type`: A string specifying the field's type (TODO: is this field the same `cairo_type` as
      in a `type_definition`?).
  - `offset`: The offset where the field begins.

**Type definitions**

Its `type` is `type_definition`.

A type definition is used to define types that are not structs, such as tuples. It has a single
extra field named `cairo_type`, which describes the type itself.


### References

The `reference_manager` field contains a single attribute named `references`, which is a list of
references. Each item has the following format:

  - `ap_tracking_data`: Not sure why it's necessary. A memory address.
  - `pc`: The `pc` after which said reference is valid in the current scope.
  - `value`: A Cairo string defining both the value and the type.

It's used by the `flow_tracking_data` objects through the JSON file.


## Instruction encoding

  - Each instruction is a felt (which fits in 64 bits).
  - The prime number must be at least `1 << 64`, otherwise it cannot encode instructions (due to the modulo).

### Encoding

Each instruction has four parts:

  - Flags: A bitset of 16 bits.
  - Offset #2: An offset (`i16`). Applied to operand #1.
  - Offset #1: An offset (`i16`). Applied to operand #0.
  - Offset #0: An offset (`i16`). Applied to the destination.

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

| #7 | #8 | #9 |      Meaning      |
|----|----|----|-------------------|
|  0 |  0 |  0 | `pc` += 1         |
|  1 |  0 |  0 | Absolute jump.    |
|  0 |  1 |  0 | Relative jump.    |
|  0 |  0 |  1 | Conditional jump. |

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
|   0 |   0 |   1 | Equality assert. |

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


## Instruction usage

### Asserts

The `assert` instruction can be invoked using either two or three operands. The
rightmost operand (op1) can be an immediate, or an `ap` or `fp`-relative offset.

When specifying an immediate, the offset must be equal to `1` since that's the
offset of the immediate starting from the instruction's `pc`.

If using only two operands, then `op0` is unused and must be set to an
`fp`-relative with an offset of `-1`.

Although the first operand is named `dst`, it doesn't have to be the unknown
value. An assertion such that the unknown value is on the right is perfectly
valid and should be handled accordingly.

This instruction may increment `ap` by one if the corresponding flags are set.

### An `ap` update

An `ap` update instruction increments `ap` by an arbitrary value.

Since its `dst` operand is unused, it must be an `fp`-relative with offset `-1`.
Expressions are supported and follow the same rules as the `assert` instruction.

> Note: Although `ap` is a `felt`, it's unrealistic that a program would
    allocate that much memory. Technically, the Python VM will run a program
    with holes spanning more than the space available in 64-bit integers,
    however since such program would be practically impossible to prove, it's
    probably sensible to throw an error if an `ap` overflow is detected.

### Jumps

Jumps are used to add non-deterministic behaviour to Cairo.

They can be absolute, relative or conditional. If conditional, the jump is
automatically relative, and the condition is always `x != 0`.

Unconditional jumps don't use `dst`, therefore it must be set to `fp`-relative
with offset `-1`. Conditional jumps use `dst` as the comparison pointer (the `x`
in `x != 0`).

Unconditional jumps support expressions, which also follow the same rules as the
`assert` instruction. Conditional jumps, however, can only use `op1` with either
an immediate, or a register-relative value. As before, `op0` will have to be
`fp`-relative with offset `-1`.

> Note: Inter-function non-conditional jumps are allowed and supported, but are
    probably not a good idea.
> Note: I still haven't found a snippet that doesn't throw an error at runtime
    when executing an absolute jump. Maybe it only works with pointer cells
    instead of `felt`s? Or something about a double dereference?

### Calls

Function calls push two values into the execution segment before jumping to the
target function's entry `pc`:

    [ap] = <value-of-fp>, ap++;
    [ap] = <value-of-pc>, ap++;
    <jump to function's entry `pc`>

Function calls use `dst` as the `fp` pointer destination, and therefore must be
set to `ap-relative` with offset 0. The `op0` operand is used as the return `pc`
and must be `ap-relative` with offset 1. The other operand (`op1`) indicates the
target function's entry `pc`.

The `ap` pointer is always incremented by two (its flags have a special
meaning).

> Note: In reality, the target `pc` doesn't really have to be the function's
    entry `pc`. Any valid `pc` works as well.
> Note: I still haven't found a snippet that doesn't throw an error at runtime
    when executing an absolute jump. Maybe it only works with pointer cells
    instead of `felt`s? Or something about a double dereference?

### Returns

Function returns use `dst` as the new `fp` value, and therefore must be set to
`fp-relative` with offset `-2`. Operand #0 is unused (and must be set as
mentioned earlier), and op #1 is `ap`-relative with offset -1. Of course,
expressions are not supported.

The `pc` update field is set to an absolute jump, and `ap` cannot be updated.

> TODO: Why is `op1` `ap`-relative with offset -1? If it was the return `pc`,
    shouldn't it be `fp`-relative? Or is this the default value when `op1` is
    unused?
