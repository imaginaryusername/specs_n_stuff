# Double Spend Proof Generation and Propagation

@im_uname
WIP
Last revised: 20190412

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


The `dsproof` command generates the following, with two different transactions spending the same outpoint as input:

| Field Size | Description | Data Type  | Comments |
| -----------|:-----------:| ----------:|---------:|
| 36     | outpoint | outpoint | The outpoint that was double spent |
| ?      | proof | double_spend_proof | Proof a double spend took place |

If the two transaction share more than one outpoint, then a corresponding number of proofs shall be generated for each outpoint shared. This is important for peers that do not have either of the transactions that generated the proof.

`double_spend_proof`:

| Field Size | Description | Data Type  | Comments |
| -----------|:-----------:| ----------:|---------:|
| 1 | version | uint8 | version byte; 1 for p2pkh |
| ? | tx_dig1 | tx_dig | Transaction digest of the one transaction before final double SHA256 hash, corresponding to the doublespent outpoint|
| ? | tx_dig2 | tx_dig | Transaction digest of a different transaction before final double SHA256 hash, corresponding to the doublespent outpoint|
| ? | sig1 | char | signature of the double-SHA256 of tx_dig1, extracted from tx1 |
| ? | sig2 | char | signature of the double-SHA256 of tx_dig2, extracted from tx2 |

A second type that addresses p2sh-multisignature transactions is structured as follows: 

| Field Size | Description | Data Type  | Comments |
| -----------|:-----------:| ----------:|---------:|
| 1 | version | uint8 | version byte; 2 for p2sh-multisig double-signing proof |
| 1 | pk_seq1 | uint8 | "sequence" number that points to which pubkey in checkmultisig the first signature pertains to |
| 1 | pk_seq2 | uint8 | "sequence" number that points to which pubkey in checkmultisig the second signature pertains to. May be identical to pk_seq1. |
| ? | tx_dig1 | tx_dig | Transaction digest of the one transaction before final double SHA256 hash, corresponding to the doublespent outpoint|
| ? | tx_dig2 | tx_dig | Transaction digest of a different transaction before final double SHA256 hash, corresponding to the doublespent outpoint|
| ? | sig1 | char | signature of the double-SHA256 of tx_dig1, extracted from tx1 and signed by pubkey from pk_seq1 |
| ? | sig2 | char | signature of the double-SHA256 of tx_dig2, extracted from tx2 and signed by pubkey from pk_seq2 |

For tx_dig, refer to https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/replay-protected-sighash.md.

A third type of proof that also pertains to p2sh-multisignature transactions is structured as follows. It contains strictly m+1 signatures where the associated transaction is m-of-n p2sh-multisig. This type is only relevant upon complete implementation of BIP62.

| Field Size | Description | Data Type  | Comments |
| -----------|:-----------:| ----------:|---------:|
| 1 | version | uint8 | version byte; 3 for p2sh-multisig m+1 proof |
| 1 | pk_seq1 | uint8 | "sequence" number that points to which pubkey in checkmultisig the first signature pertains to |
| 1 | pk_seq2 | uint8 | "sequence" number that points to which pubkey in checkmultisig the second signature pertains to. May **not** be identical to pk_seq1. |
| ... | ... | ... | ... | 
| 1 | pk_seq m+1 | uint8 | "sequence" number that points to which pubkey in checkmultisig the m+1 signature pertains to. All pubkeys must be unique. |
| ? | tx_dig1 | tx_dig | Transaction digest of one transaction before final double SHA256 hash, corresponding to the doublespent outpoint |
| ? | tx_dig2 | tx_dig | Transaction digest of one transaction before final double SHA256 hash, corresponding to the same outpoint. If identical to another tx_dig before it, replaced by an uint8 number that refers to the sequence of the previous tx_dig. |
| ... | ... | ... | ... | 
| ? | tx_dig m+1 | tx_dig | Transaction digest of one transaction before final double SHA256 hash, corresponding to the same outpoint. May be identical to any of previous tx_dig. If identical to another tx_dig before it, replaced by an uint8 number that refers to the sequence of the previous tx_dig. |
| ? | sig1 | char | signature of the double-SHA256 of tx_dig1, signed by pubkey from pk_seq1 |
| ? | sig2 | char | signature of the double-SHA256 of tx_dig2, signed by pubkey from pk_seq2 |
| ... | ... | ... | ... | 
| ? | sig m+1 | char | signature of the double-SHA256 of tx_dig m+1, signed by pubkey from pk_seq m+1 |

A message type 'dsproof_req' is structured as follows:

| Field Size | Description | Data Type  | Comments |
| -----------|:-----------:| ----------:|---------:|
| 36     | outpoint | outpoint | The outpoint that was double spent |

A new inventory type is added:

| Value | Name |
| -----------|:-----------:|
| 4     | DOUBLE_SPEND_PROOF |

The inventory vector hash shall be set to the hash of the serialized outpoint which was double spent. This implies that it is possible for two different double spend proofs to share the same hash, but this is OK since we only care about relaying one double spend alert per input - any will do.

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

* Any one of the tx_digs in proof do not contain the double spent `outpoint`.
* The signature for any of the transaction digests is invalid.

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

TODO: Explanation for specs, attack scenarios

Thanks to Tom Zander, Mark Lundeberg, Peter Rizun, Andrew Stone, Dagur, Greg Griffith for feedback and initial reworks; Jonathan Silverblood and Chris Pacia for additional comments; thanks to Tom Harding for providing the inspiration and initial framework in tackling different doublespend types. 
