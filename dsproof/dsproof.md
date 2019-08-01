# Double Spend Proof Generation and Propagation

@im_uname
WIP
Last revised: 20190729
Author: @im_uname, adapted from Chris Pacia's Double Spend Alert (https://github.com/cpacia/spec/blob/master/double-spend-alerts.md) 

## Abstract

This document describes a new Bitcoin Cash network message that is generated when two transactions spending the same input are detected on a participating node, and related protocol to relay it through the network among participating nodes.

## Design requirements

A transaction that has its inputs all being from P2PKH or P2SH-multisig outputs, follow prevailing standardness rules and has all signatures signed with SIGHASH_ALL without the ANYONECANPAY flag, and in the case of P2SH-multisig containing all unique pubkeys, is hereby referred as "protected transactions".

 
1. The proof message, by itself, must be sufficient to prove the offending output(s) being signed for different transactions, or in the case of p2sh-multisig, signed more than necessary numbers to allow multiple versions of the same transaction to coexist, breaking unconfirmed chains (subject to BIP62 implementation). 
2. The proof message, by itself, must not be sufficient to reconstruct the offending transactions in whole unless additional information is relayed.
3. Generating and relaying proof must not create additional bandwidth and processing requirement for nodes above O(n) where n is existing transaction traffic in the worst case. 
4. Node wallets and SPV wallets should be able to conclude from the absence of a DS proof over a tolerated time t before next block, that a protected transaction has been relayed to all mempools of the "honest" public network following rules no more stringent than prevailing relay rules, and no alternative was able to supercede the transaction in question in terms of first-seen.
5. (Optional, subject to BIP62 implementation): For a DAG (or chain) of unconfirmed protected transactions all lacking DS proofs over time t, we also get the same certainty as in #4. This will require full deployment of BIP62 third-party malleability fixes. (subject to chain-length limits)

## Specification

The `dsproof` message is defined with the following spec;

| Field Size | Description | Data Type  | Comments |
| -----------|:-----------:| ----------:|---------:|
| 32 | prevTxId | sha256 | The txid being spent |
| 4  | prevOutIndex | int | The output being spent |
| | first_spender | spender | the first sorted spender |
| | double_spender | spender | the second spender |

Each `spender` is build from a single input from a transaction. Each
of these inputs spends the same output, indicated by prevTxId/prevOutIndex.
This means that both signatuers can be validated with the same public key.

The details required to validate one input are provided in the spender field;

| Field Size | Description | Data Type  | Comments |
| -----------|:-----------:| ----------:|---------:|
| 4 | tx-version | unsigned int | Copy of the transactions version field |
| 4 | sequence | unsigned int | Copy of the sequence field of the input |
| 4 | locktime | unsigned int | copy of the transactions locktime field |
| 32 | hash-prevoutputs | sha256 | Transaction hash of prevoutputs |
| 32 | hash-sequence | sha256 | Transaction hash of sequences |
| 32 | hash-outputs | sha256 | Transaction hash of outputs |
| 1-9 | list-size | var-int | number of items in the push-data list |
|  | push-data | byte-array | raw byte-array of a push-data. For instance a signature |

The fields hash-prevoutputs, hash-sequence and hash-outputs are defined in the following spec;
https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/replay-protected-sighash.md.

A new inventory type is added:

| Value | Name |
| -----------|:-----------:|
| 0x94a1 | DOUBLE_SPEND_PROOF |

The inventory vector hash is the double-sha256 hash of the serialized double-spend message.

## Message Generation and Relaying

The proof must be only generated when:

1. At least two different valid transactions spending at least one identical output, at least one of them being a protected transaction, are detected (generates type 1 or type 2 proof); or

2. A P2SH-multisig input that requires m of n signatures from unique pubkeys has more than m signatures through any number of transactions spending the same outpoint (generates type 3 proof). 

it must not be forged otherwise. Note that two transactions are considered different if they contain different signatures, even if their transaction digests are exactly the same. In the case of multisignature, any pubkey signing a different transaction spending the same outpoint as a protected transaction is sufficient proof.

When a node detects two transactions that spends the same outpoint, AND at least one of the transactions is a protected transaction, or a P2SH-multisig input containing more than necessary signatures (requires BIP62), it should check to see if it has a double spend proof to that outpoint in inventory.
If not it should construct a `dsproof` message for the outpoint and announce to its peers via `inv` packets.

When receiving an `inv` packet for a double spend proof a node should:

1. Check if the outpoint is spent by any transaction in mempool. If not, record the outpoint and corresponding peer in an index. 
2. Check to see if there is a double spend proof corresponding to the same outpoint in inventory.
3. If not, download, validate, store and announce the dsproof. 
4. If a transaction that matches a recorded outpoint is relayed to the node, request dsproof from previous recorded peer with a "dsproof_req" message. 

The validation of the double spend proof fails if:

* The double spent `outpoint` is not in `tx_pre1` or `tx_pre2` of the proof.
* The signature for any of the transaction digests is invalid.
* Ths two halves of the doublespend proof are identical in the case of Type 1 and Type 2.
* In the case of Type 3, any two of the public keys supplied are identical.

If validation is successful the node should announce and relay the `dsproof` message to its peers.

A proof of a given outpoint shall also be retrievable by requesting the corresponding mempool transaction spending the same outpoint. This will be useful for servicing SPV clients (see below). 

Double spend proofs should be removed from inventory once its corresponding transaction is removed from mempool, either via confirmation or timeout.

Note that there can be more than one proofs corresponding to different outpoints for the same transaction.

## SPV Relay

An SPV client should request the transaction relevant to their addresses, as well as all its unconfirmed mempool ancestors, from a full node. It will never construct a new double spend proof. If one or more DS proof is already present at the servicing full node, it can be relayed with the corresponding transaction.

If a DS proof is not already present when a transaction is relayed to an SPV client, as the SPV client likely "subscribes" to certain addresses, whether via BIP37 bloom filters, static addresses (Electrum) or xpub (Copay), the full node can identify the corresponding mempool transaction when a new DS proof comes in, and relay it to appropriate SPV clients that the transaction is relevant to.

## Worst Case Bandwidth and Memory cost

TODO: How big can the set of DSproofs be for a mempool that's almost entirely made up of P2PKH inputs?

## Using Double Spend Alerts: Merchant considerations

A merchant node which utilizes double spend proofs should follow the following general procedure, when receiving a transaction that pays them whether through network or direct connection (e.g. BIP70): 

1. Evaluate the transaction that pays them. If the transaction does not pay sufficient fee or is otherwise unfit as defined by the merchant independent of the criteria below, apply remedial action either by rejecting transaction of custom negotiations with the customer - fast transaction becomes irrelevant. 

2. If the transaction does not fit any of the following criteria, do not rely on doublespend proof, and instead either wait for confirmation or apply more stringent risk management: 

        The transaction must contain all P2PKH or multisig-P2SH inputs.
        The transaction must either be spending only from confirmed UTXOs, or all of its ancestors in mempool must also be all-P2PKH/P2SH-multisig-only transactions (Optional, requires BIP62). 
        All of the inputs in the relevant transaction, and its mempool ancestor chain (Optional, requires BIP62), must be signed SIGHASH_ALL without ANYONECANPAY.
 
3. After affirming that the transaction fits the critera for applying double spend proofs, the merchant then waits T seconds, T being a variable adjusted to risk tolerance, for a proof to arrive. If a double spend proof corresponding to the paying transaction or any of its ancestors arrive, the merchant shall either decline the payment, or wait for confirmation. If no proof arrives within T seconds, the merchant hands out goods or services.

Note that in the case of an unconfirm chain, if BIP62 fixes are not implemented in full, transactions can be malleated by third parties to invalidate descendents without triggering doublespend proof - and any attempt at making proof against this will be vulnerable to false positive attacks. Double spend proof against unconfirmed proofs should therefore only be activated after implementation of the full set of BIP62 fixes (https://github.com/bitcoin/bips/blob/master/bip-0062.mediawiki).

TODO: Simulate merchant risk in both single-tx and unconfirmed chain.

## Limitations

Double spend proofs are useful against an adversary who does not collude with a miner, and instead rely on network latency or miner policy differences to double spend the merchant. It does not address the scenario of miner collusion, which is undetectable before confirmation. In the event of a miner collusion it is possible for a merchant node to use the same mechanism to construct a proof for extraprotocol pursuits of justice, but it is beyond the scope of this document. 

As of current version of this spec, nodes that are more stringent in relay criteria will not be able to generate doublespend proofs on its own if a doublespend transaction outside its relay criteria is present on the network; it will rely on its more permissive peers to do so. This reduces denial of service attack surfaces and may increase the willingness of node operators to relay proof at all, as proof generation criteria is aligned with relay policy. However, this may lead to increased latency and less robust generation in scenarios of low participation (TODO:simulate).

## Tests (WIP)

### Generation

P2PKH

P2SH-multisig

### Verification

### Merchant-end flagging

From confirmed

Unconfirmed parent

Thanks to Tom Zander, Mark Lundeberg, Peter Rizun, Andrew Stone, Dagur, Greg Griffith for feedback and initial reworks; Jonathan Silverblood and Chris Pacia for additional comments; thanks to Tom Harding for providing the inspiration and initial framework in tackling different doublespend types. 
