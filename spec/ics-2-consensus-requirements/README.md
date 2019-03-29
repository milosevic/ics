---
ics: 2
title: Consensus Verification
stage: proposal
category: ibc-core
requires: 23
required-by: 3
author: Juwoon Yun <joon@tendermint.com>, Christopher Goes <cwgoes@tendermint.com>
created: 2019-02-25
modified: 2019-03-05
---

## Synopsis

Consensus verification requires the properties that the chains on the network are 
expected to satisfy. The properties are needed for efficient and safe verification on the 
higher level mechanics, such as connection and channel semantics. The algorithm who uses these 
properties to verify another chain is referred as "lightclient", which is embedded on the chains.

## Specification

### Motivation

`Header` defines required properties of the blockchain on the network. The implementors can 
check whether the consensus that they are using is qualified to be connected to the network or 
not. If not, they can modify the algorithm or wrap it with additional logic to make it 
compatible with the specification. It also provides base layer for the protocol that the other 
components can rely on.

### Desired Properties

* Blockchains, defined as an infinite list of `Commit` starting from its genesis, is linear; no 
conflicting `Commit`s can be both validated, thus no data can be rewritten after it has been 
committed. Two `Commit`s are conflicting when both has same height but not equal.

* Verifiers can verify future `Commit`s using an existing `TrustedCommit`. When the verifier 
validates it, the verified header is in the canonical blockchain.

* `TrustedCommit`s contains an accumulator root(ICS23) that the other logics can verify whether 
key-value pairs exists or not with it.

### Technical Specification

#### Definitions

* `TrustedCommit` is a blockchain commit which can be used to prove future commits, stored in
  one blockchain to verify the state of the other.
  Defined as 3-tuple `(v :: Commit -> (Error|TrustedCommit), r :: AccumulatorRoot)`, where
    * `v` is the verifier, proves child `Commit.p` and returns the updated `TrustedCommit`
    * `r` is the `AccumuatorRoot`, used to prove internal state

* `Commit` is a blockchain header which provides information to update `TrustedCommit`, 
  submitted to one blockchain to update the stored `TrustedCommit`.
  Defined as 2-tuple `(p :: CommitProof, tc :: TrustedCommit`, where
    * `p` is the commit proof used by `TrustedCommit.v` to verify
    * `tc` is the new `TrustedCommit` which will replace the existing one after the verification
 
* `Consensus` is a blockchain consensus algorithm which generates valid `Commit`s.
  Defined as a function `Commit -> [Transaction] -> Commit`

* `Blockchain` is a subset of `(TrustedCommit, [Commit])`, generated by a `Consensus`.

#### Requirements

Consensus verification requires the following accumulator primitives with datastructures and
properties as defined in ICS23:

* `AccumulatorRoot`

#### Subprotocols

##### Verifier

Verifiers prove new `Commit` generated from a blockchain. It is expected to prove 
a `Commit` efficiently; efficient then replaying `Consensus` logic for given parent `Commit`
and the transactions. 

##### Accumulator Root

`TrustedCommit` contains an accumulator root, which identifies the whole state of the corresponding
blockchain at the point of time that the commit is generated. It is expected that the verifying 
inclusion or exclusion of certain data in the accumulator is done efficient. See ICS23 for the details
about the `AccumulatorRoot`s.

##### CommitProof

`CommitProof` is a type dependent on concrete implementations of `Commit`. `TrustedCommit.v` uses
the information stored in `Commit.p` to check `Commit.tc`. 

##### Consensus 

`Consensus` is a blockchain protocol which actually generates a list of `Commit`. 

1. Headers have no more than one direct child
 
* Satisfied if: deterministic safety
* Possible violation scenario: validator double signing, miner double spend

2. Headers have at least one direct child

* Satisfied if: liveness, lightclient verifier continuability
* Possible violation scenario: synchronized halt, incompatible hard fork

3. Headers' accumulator root are valid transition from the parents'

* Satisfied if: decentralized block generation, well implemented state machine
* Possible violation scenario: invariant break, validator cartel

If a block is submitted to the chain(and optionally passed some challenge period for fraud proof), 
then it is assured that the packet is finalized so the application logic can process it.
If a header is proven to violate one of the properties, the tracking chain should detect and take
action on the event to prevent further impact. See (link for the ICS for equivocation and fraud proof) 
for details.

### Example Implementation

An example blockchain `B` is run by a single operator. If a block is signed by the operator, 
then it is valid. `B` contains `KVStore`, which is a associative list with type of `[([byte], [byte])]`.  

// TODO

## History 

March 5th 2019: Initial ICS 2 draft finished and submitted as a PR