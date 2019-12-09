# OP_PUSHSTATE Draft Specification

OP_PUSHSTATE is a new opcode for the BCH virtual machine which provides direct access to elements of the virtual machine’s state during evaluation. `State Item Identifiers` are used to select particular state elements according to the [State Item Identifiers](#state-item-identifiers) table. On evaluation, the value of the requested element is pushed to the stack.

## Deployment

Deployment of this proposal is recommended for the November 2020 upgrade (`BCH_2020_11`).

## Motivation

When designing covenant-style transactions, elements of a transaction’s signing serialization (A.K.A. `sighash` preimage) often must be pushed to the stack in unlocking scripts so they can be validated by the locking script. Because the virtual machine must already maintain this state information over the course of an evaluation, these pushes do not provide any performance or optimization value but force all covenant transactions to duplicate state information (needlessly inflating transaction sizes).

OP_PUSHSTATE allows scripts to request state information directly from the virtual machine during evaluation, reducing wasted transaction space.

## Opcode Description
```
Pop the top item from the stack as a state identifier. If the identifier is recognized, push the identified state value to the stack, otherwise, error.
```

## Codepoint

The next undefined codepoint (`0xbc`/`188`) is defined as `OP_PUSHSTATE`.

## State Item Identifiers

OP_PUSHSTATE reads the top element on the stack as a **state item identifier**. Identifiers are mapped to specific state information as follows:

| Name                              | Number | Hex    | Description                                                                                                                        |
| --------------------------------- | ------ | ------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| Version                           | `1`    | `0x01` | The transaction's version number.                                                                                                  |
| Transaction Outpoints             | `2`    | `0x02` | The signing serialization of all transaction outpoints.                                                                            |
| Transaction Outpoints Hash        | `3`    | `0x03` | The double-sha256 hash (`OP_HASH256`) of `Transaction Outpoints`.                                                                  |
| Transaction Sequence Numbers      | `4`    | `0x04` | The signing serialization of all transaction sequence numbers.                                                                     |
| Transaction Sequence Numbers Hash | `5`    | `0x05` | The double-sha256 hash (`OP_HASH256`) of `Transaction Sequence Numbers`.                                                           |
| Outpoint Transaction Hash         | `6`    | `0x06` | The transaction hash/ID of the outpoint being spent by the current input.                                                          |
| Outpoint Index                    | `7`    | `0x07` | The index of the outpoint being spent by the current input.                                                                        |
| Covered Bytecode Length           | `8`    | `0x08` | The length of the covered bytecode encoded as a Bitcoin VarInt.                                                                    |
| Covered Bytecode                  | `9`    | `0x09` | The bytecode segment covered by the signature (A.K.A. `scriptCode`)                                                                |
| Output Value                      | `10`   | `0x0a` | The output value of the outpoint being spent by the current input.                                                                 |
| Sequence Number                   | `11`   | `0x0b` | The sequence number of the outpoint being spent by the current input.                                                              |
| Corresponding Output              | `12`   | `0x0c` | The signing serialization of the transaction output with the same index as the current input. If none, an empty stack item (`0x`). |
| Corresponding Output Hash         | `13`   | `0x0d` | The double-sha256 hash (`OP_HASH256`) of `Corresponding Output`.                                                                   |
| Transaction Outputs               | `14`   | `0x0e` | The signing serialization of all transaction outputs.                                                                              |
| Transaction Outputs Hash          | `15`   | `0x0f` | The double-sha256 hash (`OP_HASH256`) of `Transaction Outputs`.                                                                    |
| Locktime                          | `16`   | `0x10` | The transaction's locktime.                                                                                                        |


Note, all state item identifiers can be interpreted as valid Script Numbers. This ensures maximum future protocol compatibility and implementation flexibility. Identifiers begin at `1` to reserve both empty stack items (`0x`) and zero values (`0x00`/`0x80`) for use in future extensions.

## Implementations

Please see the following reference implementations for examples and test vectors:

[TODO]
<!-- - [bitcoin-ts]() – [opcode](), [tests]() -->

<!-- ## Optimization Example

When used to optimize the CashChannel protocol, average transaction sizes are reduced from ~900 bytes to ~500 bytes. 
- [Current CashChannel Specification]()
- [CashChannel Specification – Optimized with OP_PUSHSTATE]() -->

# Other Considerations

Below are related ideas and discussion. Important rationale from this section will be summarized and included in the final specification.

## Single vs. Multiple Opcodes

Bytecode length of covenant scripts could be further reduced by implementing the functionality of `OP_PUSHSTATE` across more than one opcode, e.g. "`OP_PUSH_OUTPUT_VALUE`". 

This strategy would expend a much larger number of opcodes than the single-opcode strategy. Since a multiple-opcode strategy would only save the additional "State Identifier" byte(s), this would not be the most valuable use of additional opcodes. When common network usage is taken into account, other optimizations (like additional compound opcodes) offer significantly greater space savings network-wide.

Backwards compatibility is also more challenging – any future changes to the signing serialization algorithm would likely require new opcodes or additional bytes.

## Inclusion of State Identifiers for Hashed Values

As specified, several state identifiers represent the hash of other state items (`Transaction Outpoints Hash`, `Transaction Sequence Numbers Hash`, `Corresponding Output Hash`, and `Transaction Outputs Hash`). This allows scripts to avoid manually re-hashing the values (e.g. `<2> OP_PUSHSTATE OP_HASH256`). This optimization reduces transaction sizes by eliminating the hashing opcode, incentivizes better performance, and makes performance optimizations easier for implementations.

In most cases, the virtual machine will be required to perform these hash functions during a signature checking operation (with a few exceptions, e.g. `Corresponding Output Hash` in an input which doesn't utilize "SIGHASH_SINGLE"). By allowing scripts to request the hashed result directly, scripts are incentivized to avoid harder-to-optimize constructions (e.g. `<2> OP_PUSHSTATE OP_SHA256 OP_SHA256`).

Additionally, most "covenant"-style scripts will require each hashed state value (to construct the full signing serialization when validating a signature), while fewer preimage values are needed for validating conformance to the covenant. This implies that state identifiers representing hashes will be more common than those representing their preimages.
 
## Multiple State Identifiers Per Operation

As currently specified, `OP_PUSHSTATE` will often be used in series with resulting state values being concatenated. It would be valuable to develop a syntax for specifying a list of state items to concatenate and push in a single operation.

For example, as specified, OP_PUSHSTATE-optimized CashChannels will include the following script segment:
```
<1> OP_PUSHSTATE // version
<3> OP_PUSHSTATE // transaction outpoints hash
<5> OP_PUSHSTATE // transaction sequence numbers hash
<6> OP_PUSHSTATE // outpoint transaction hash
<7> OP_PUSHSTATE // outpoint index
<8> OP_PUSHSTATE // covered bytecode length
OP_CAT OP_CAT OP_CAT OP_CAT OP_CAT
```

One possible strategy is to allow multiple state identifiers to be concatenated into a single stack item. The `OP_PUSHSTATE` operator would pop the top value from the stack, and interpret it as an array of bytes representing the list and order of state items to concatenate and push to the stack. (Pushes larger than the current limit – 520 bytes – produce an error.) 

This would reduce the above script segment to:

```
<0x010305060708> OP_PUSHSTATE
```

This change would limit the count of state identifiers to `255`, though the byte `0xff` can be reserved to indicate longer identifiers (as in UTF-8).

## Lazy Hashing & Performance Optimizations

TODO: discuss effects on signature/hash caching, other performance considerations

# Contributing

Feedback and improvements to this draft specification are welcome. Please [open an issue](https://github.com/bitjson/op-pushstate/issues) or join the [OP_PUSHSTATE working group on Telegram]().