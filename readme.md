# OP_TX* Draft Specification

OP_TX* is a set of new opcodes for the BCH virtual machine which provides direct access to elements of the virtual machine’s state during evaluation. Each opcode puts a specific state element onto the stack (see [State item opcodes](#state-item-opcodes) for a full list).

This set of opcodes serves as replacement for the previously proposed OP_PUSHSTATE.

## Deployment

Deployment of this proposal is recommended for the November 2020 upgrade (`BCH_2020_11`).

## Motivation

When designing covenant-style transactions, elements of a transaction’s signing serialization (A.K.A. `sighash` preimage) often must be pushed to the stack in unlocking scripts so they can be validated by the locking script. Because the virtual machine must already maintain this state information over the course of an evaluation, these pushes do not provide any performance or optimization value but force all covenant transactions to duplicate state information (needlessly inflating transaction sizes).

The OP_TX* opcodes allow scripts to request state information directly from the virtual machine during evaluation, reducing wasted transaction space.

## Opcode Description
```
1) If the opcode requires indices, pop the required indices from the stack. If any index is not a minimally encoded number, error.
2) Push state value associated with the respective opcode to the stack. Error if the state value exceeds the maximum stack item limit or if the maximum stack size has been reached.
```

## Codepoint

For each of the OP_TX* opcodes, a multi-byte opcode will be used, prefixed by 0x65 and followed by one additional byte, which determines the state item to be pushed (see [State item opcodes](#state-item-opcodes)). This opcode replaces the currently disabled OP_VERIF, which at the time of writing makes scripts totally unspendable, even when within an unexecuted OP_IF branch.

## State Item Opcodes

The following opcodes will be added:

| Name                      | Hex     | Inputs             | Pushed data                                                                                                                        |
| ------------------------- | ------- | ------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| OP_TXVERSION              | `65 01` |                    | Transaction version number as minimally encoded integer.                                                                           |
| OP_TXOUTPOINT             | `65 02` | `nInput`           | Signing serialization of the outpoint at the `nInput`-th input.                                                                    |
| OP_TXHASHOUTPOINTS        | `65 03` |                    | Double-sha256 hash (`OP_HASH256`) of the signing serialization of all transaction outpoints.                                       |
| OP_TXSEQUENCENUMBER       | `65 04` | `nInput`           | Sequence number as minimally encoded integer of the `nInput`-th input.                                                             |
| OP_TXHASHSEQUENCENUMBERS  | `65 05` |                    | Double-sha256 hash (`OP_HASH256`) of the signing serialization of all transaction sequence numbers.                                |
| OP_TXINPUTDATA            | `65 06` | `nInput` `nPushop` | Value pushed by the `nItem`-th pushop of the `nInput`-th input.                                                                    |
| OP_TXOWNOUTPOINT          | `65 07` |                    | Outpoint being spent by the current input.                                                                                         |
| OP_TXOWNOUTPOINTINDEX     | `65 08` |                    | Index of the outpoint being spent by the current input as minimally encoded integer.                                               |
| OP_TXOWNSCRIPTCODE        | `65 09` |                    | Bytecode segment covered by the signature (A.K.A. `scriptCode`)                                                                    |
| OP_TXOWNVALUE             | `65 0a` |                    | Output value of the outpoint being spent by the current input as minimally encoded integer.                                        |
| OP_TXOWNSEQUENCENO        | `65 0b` |                    | Sequence number of the outpoint being spent by the current input as minimally encoded integer.                                     |
| OP_TXOUTPUTVALUE          | `65 0c` | `nOutput`          | Value of the `nOutput`-th output as minimally encoded integer.                                                                     |
| OP_TXOUTPUTSCRIPT         | `65 0d` | `nOutput`          | Script of the `nOutput`-th output.                                                                                                 |
| OP_TXHASHOUTPUTS          | `65 0e` |                    | Double-sha256 hash (`OP_HASH256`) of the signing serialization of all transaction outputs.                                         |
| OP_TXLOCKTIME             | `65 0f` |                    | Transaction locktime as minimally encoded integer. If locktime > 2³¹, error.                                                       |

# TODO: adapt following sections to multi-byte opcode

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

Several state identifiers represent the hash of other state items (`Transaction Outpoints Hash`, `Transaction Sequence Numbers Hash`, `Corresponding Output Hash`, and `Transaction Outputs Hash`). This allows scripts to avoid manually re-hashing the values (e.g. `<2> OP_PUSHSTATE OP_HASH256`). This optimization reduces transaction sizes by eliminating the hashing opcode, incentivizes better performance, and makes performance optimizations easier for implementations.

In most cases, the virtual machine will be required to perform these hash functions during a signature checking operation (with a few exceptions, e.g. `Corresponding Output Hash` in an input which doesn't utilize "SIGHASH_SINGLE"). By allowing scripts to request the hashed result directly, scripts are incentivized to avoid harder-to-optimize constructions (e.g. `<2> OP_PUSHSTATE OP_SHA256 OP_SHA256`).

Additionally, most "covenant"-style scripts will require each hashed state value (to construct the full signing serialization when validating a signature), while fewer preimage values are needed for validating conformance to the covenant. This implies that state identifiers representing hashes will be more common than those representing their preimages.

## Inclusion of Covered Bytecode Length

The length of the covered bytecode could be directly included in the `Covered Bytecode` as a prefix (and extracted with `OP_SPLIT` if needed), or it could be derived by scripts using `OP_SIZE` and some simple math (to convert from a Script Number to a Bitcoin `VarInt`). Both of these options significantly increase the complexity of scripts and require branching to handle multiple `VarInt` sizes. Providing direct access to the properly-encoded length value dramatically simplifies these operations, saving network bandwidth and eliminating several security "footguns".

Because the number pushing opcodes (`OP_1`-`OP_16`) allow for a single-byte push of numbers up to 16, `Covered Bytecode Length` can be included in the initial set without losing optimization (requiring multi-byte pushes for identifiers of single state items).
 
## Inclusion of Concatenation Functionality

`OP_PUSHSTATE` must often be used in series with resulting state values being concatenated. For example, without multi-value templating, OP_PUSHSTATE-optimized CashChannels would require the following script segment:
```
<1> OP_PUSHSTATE // version
<3> OP_PUSHSTATE // transaction outpoints hash
<5> OP_PUSHSTATE // transaction sequence numbers hash
<6> OP_PUSHSTATE // outpoint transaction hash
<7> OP_PUSHSTATE // outpoint index
<8> OP_PUSHSTATE // covered bytecode length
OP_CAT OP_CAT OP_CAT OP_CAT OP_CAT
```

By including the templating and concatenation functionality in `OP_PUSHSTATE`, this script segment is reduced to:

```
<0x010305060708> OP_PUSHSTATE
```

While this limits the count of state identifiers to `255`, the byte `0xff` can be reserved to indicate longer identifiers (as in UTF-8).

## Lazy Hashing & Performance Optimizations

TODO: discuss implementation performance considerations

# Contributing

Feedback and improvements to this draft specification are welcome. Please [open an issue](https://github.com/bitjson/op-pushstate/issues) or join the [OP_PUSHSTATE working group on Telegram](https://t.me/pushstate).
