# OP_PUSHSTATE Draft Specification

OP_PUSHSTATE is a new opcode for the BCH virtual machine which provides direct access to elements of the virtual machine’s state during evaluation. A `template` represents the state elements bitfield according to the [State Item Identifiers](#state-item-identifiers) table. On evaluation, the value of the requested elements are concatenated and pushed to the stack.

## Deployment

Deployment of this proposal is recommended for the November 2020 upgrade (`BCH_2020_11`).

## Motivation

When designing covenant-style transactions, elements of a transaction’s signing serialization (A.K.A. `sighash` preimage) often must be pushed to the stack in unlocking scripts so they can be validated by the locking script. Because the virtual machine must already maintain this state information over the course of an evaluation, these pushes do not provide any performance or optimization value but force all covenant transactions to duplicate state information (needlessly inflating transaction sizes).

OP_PUSHSTATE allows scripts to request state information directly from the virtual machine during evaluation, reducing wasted transaction space.

## Opcode Description
```
Pop the top item from the stack as a state concatenation template. If each bit of the template is recognized, push the identified state value to the stack, otherwise, error.
```

## Codepoint

The next undefined codepoint (`0xbc`/`188`) is defined as `OP_PUSHSTATE`.

## Algorithm

When the virtual machine encounters an `OP_PUSHSTATE`, the top item is popped from the stack as the `template`.

The `template` is confirmed to be a minimally encoded 1 or 2 bytes long array. If not, error.

The value of each identified state item is concatenated together in the order specified by the `template`. If the length of this concatenation exceeds the maximum push length (currently 520 bytes), error.

The concatenated result is pushed to the stack.

## State Item Identifiers

OP_PUSHSTATE `template` bits are mapped to specific state information as follows:

| Name                              | Bit    | Hex (BE) | Hex (LE) | Description                                                                                                                        |
| --------------------------------- | ------ | -------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Version                           | `1`    | `0x01`   | `0x01`   | The transaction's version number.                                                                                                  |
| Transaction Outpoints             | `2`    | `0x02`   | `0x02`   | The signing serialization of all transaction outpoints.                                                                            |
| Transaction Outpoints Hash        | `3`    | `0x04`   | `0x04`   | The double-sha256 hash (`OP_HASH256`) of `Transaction Outpoints`.                                                                  |
| Transaction Sequence Numbers      | `4`    | `0x08`   | `0x08`   | The signing serialization of all transaction sequence numbers.                                                                     |
| Transaction Sequence Numbers Hash | `5`    | `0x10`   | `0x10`   | The double-sha256 hash (`OP_HASH256`) of `Transaction Sequence Numbers`.                                                           |
| Outpoint Transaction Hash         | `6`    | `0x20`   | `0x20`   | The transaction hash/ID of the outpoint being spent by the current input.                                                          |
| Outpoint Index                    | `7`    | `0x40`   | `0x40`   | The index of the outpoint being spent by the current input.                                                                        |
| Covered Bytecode Length           | `8`    | `0x80`   | `0x80`   | The length of the covered bytecode encoded as a Bitcoin VarInt.                                                                    |
| Covered Bytecode                  | `9`    | `0x0100` | `0x0001` | The bytecode segment covered by the signature (A.K.A. `scriptCode`)                                                                |
| Output Value                      | `10`   | `0x0200` | `0x0002` | The output value of the outpoint being spent by the current input.                                                                 |
| Sequence Number                   | `11`   | `0x0400` | `0x0004` | The sequence number of the outpoint being spent by the current input.                                                              |
| Corresponding Output              | `12`   | `0x0800` | `0x0008` | The signing serialization of the transaction output with the same index as the current input. If none, an empty stack item (`0x`). |
| Corresponding Output Hash         | `13`   | `0x1000` | `0x0010` | The double-sha256 hash (`OP_HASH256`) of `Corresponding Output`.                                                                   |
| Transaction Outputs               | `14`   | `0x2000` | `0x0020` | The signing serialization of all transaction outputs.                                                                              |
| Transaction Outputs Hash          | `15`   | `0x4000` | `0x0040` | The double-sha256 hash (`OP_HASH256`) of `Transaction Outputs`.                                                                    |
| Locktime                          | `16`   | `0x8000` | `0x0080` | The transaction's locktime.                                                                                                        |
## Calculating and pushing template

The `template` is a sum (binary `OR`) of state element bit values. At least one bit must be set.

Once the `template` parameter is determined, it needs to be encoded to bytes, and then minimally pushed the same way as `checkbits` in `OP_CHECKMULTISIG`. While the encoding to bytes is straight forward, it is worth emphasizing that certain length-1 byte vectors must be pushed using special opcodes.

* For `template <= 8`, a length-1 byte array is to be pushed.
  * The byte arrays `{0x01}` through `{0x10}` must be pushed using OP_1 through OP_16, respectively.
  * The byte array `{0x81}` must be pushed using OP_1NEGATE.
  * Other cases will be pushed using no special opcode, i.e., using `0x01 <template>`.
* For `9 <= template <= 16`, a length-2 byte array in Little-Endian order is to be pushed.
  * The push will always be `0x02 LL HH`, where `LL` is the least significant byte of `template`, and `HH` is the remaining high bits.

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

## Ordering of Identifiers

State items have been mapped to identifying numbers/hex values in the order in which they appear in the [Bitcoin Cash signing serialization algorithm](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/replay-protected-sighash.md) (with preimages listed before their hashes). While items could also be ordered alphabetically by their names, this would ossify naming choices, and any future extensions would break the alphabetical ordering.

## Inclusion of Identifiers for Hashed Values

Several state identifiers represent the hash of other state items (`Transaction Outpoints Hash`, `Transaction Sequence Numbers Hash`, `Corresponding Output Hash`, and `Transaction Outputs Hash`). This allows scripts to avoid manually re-hashing the values (e.g. `OP_2 OP_PUSHSTATE OP_HASH256`). This optimization reduces transaction sizes by eliminating the hashing opcode, incentivizes better performance, and makes performance optimizations easier for implementations.

In most cases, the virtual machine will be required to perform these hash functions during a signature checking operation (with a few exceptions, e.g. `Corresponding Output Hash` in an input which doesn't utilize "SIGHASH_SINGLE"). By allowing scripts to request the hashed result directly, scripts are incentivized to avoid harder-to-optimize constructions (e.g. `OP_2 OP_PUSHSTATE OP_SHA256 OP_SHA256`).

Additionally, most "covenant"-style scripts will require each hashed state value (to construct the full signing serialization when validating a signature), while fewer preimage values are needed for validating conformance to the covenant. This implies that state identifiers representing hashes will be more common than those representing their preimages.

## Inclusion of Covered Bytecode Length

The length of the covered bytecode could be directly included in the `Covered Bytecode` as a prefix (and extracted with `OP_SPLIT` if needed), or it could be derived by scripts using `OP_SIZE` and some simple math (to convert from a Script Number to a Bitcoin `VarInt`). Both of these options significantly increase the complexity of scripts and require branching to handle multiple `VarInt` sizes. Providing direct access to the properly-encoded length value dramatically simplifies these operations, saving network bandwidth and eliminating several security "footguns".
 
## Inclusion of Concatenation Functionality

`OP_PUSHSTATE` must often be used in series with resulting state values being concatenated. For example, without multi-value templating, OP_PUSHSTATE-optimized CashChannels would require the following script segment:
```
OP_1 OP_PUSHSTATE // version
OP_4 OP_PUSHSTATE // transaction outpoints hash
OP_16 OP_PUSHSTATE // transaction sequence numbers hash
0x01 0x20 OP_PUSHSTATE // outpoint transaction hash
0x01 0x40 OP_PUSHSTATE // outpoint index
0x01 0x80 OP_PUSHSTATE // covered bytecode length
OP_CAT OP_CAT OP_CAT OP_CAT OP_CAT
```

By including the templating and concatenation functionality in `OP_PUSHSTATE`, this script segment is reduced to:

```
0x01 0xF5 OP_PUSHSTATE
```

## Lazy Hashing & Performance Optimizations

TODO: discuss implementation performance considerations

# Contributing

Feedback and improvements to this draft specification are welcome. Please [open an issue](https://github.com/bitjson/op-pushstate/issues) or join the [OP_PUSHSTATE working group on Telegram](https://t.me/pushstate).