# Zcash/Cosmos Pegzone

This repo will eventually contain an implementation of a [Cosmos] [peg
zone][pegzone] that bridges the Cosmos ecosystem to the Zcash shielded pool.
For now it contains design notes and planning.

## Zcash-to-Cosmos Peg

This will produce Cosmos-ZEC on a Cosmos pegzone, connected by [IBC] to the
rest of the Cosmos network.  Each unit of Cosmos-ZEC is backed 1:1 by ZEC in
the Sapling shielded pool, and secured by capital staked with the pegzone
validators.  Zcash users can send ZEC to a pegzone z-addr with a Cosmos address
in the transaction's memo field.  The pegzone z-addr is controlled by a set of
Tendermint validators, who can detect that the transaction occurred and come to
consensus that coins have been received.  When this happens, they mint an
identical amount of Cosmos-ZEC and send it to the specified Cosmos address.

## Cosmos-to-Zcash Peg

Cosmos-ZEC holders create a transaction that specifies a z-addr and burns
Cosmos-ZEC.  As the pegzone validators reach consensus on theCosmos-ZEC
transaction, they use distributed signing on the spend authorization key to
prepare a Zcash transaction that releases the burned amount of ZEC to the
z-addr specified in the Cosmos transaction.  

# Components

## Distributed Key Generation for RedJubjub

The validators for the pegzone need to perform distributed key generation (DKG)
for the pegzone address's spend authorization key.  This will use a forthcoming
construction by Chelsea Komlo.

Because the validators collectively control the pegzone address, the validator
set cannot change without changing the pegzone address.  This means that unlike
other Cosmos chains (e.g., the Cosmos Hub), the validator set cannot change at
any block but only at specified epochs.  This will likely involve some kind of
3-phase key rotation schedule (previous/current/next), with mechanisms to
ensure that funds sent to the previous pegzone address (e.g., close to the
rollover boundary or by mistake) are not permanently lost, and that the next
pegzone address is available in advance (so that software using the pegzone can
have continuity).

## Distributed Signing for RedJubjub

The validators for the pegzone need to perform distributed signing (DKG) to
produce signatures with the pegzone address's spend authorization key.  This
will use a forthcoming construction by Chelsea Komlo.

Transaction generation should be integrated with the pegzone's consensus
mechanism, but the details are yet unspecified.  Ideally, as the pegzone
validators come to consensus on transactions burning Cosmos-ZEC, they should
also come to consensus on the signature for the withdrawal transaction.

## Transaction Monitoring

The validators for the pegzone need to be able to scan the Zcash chain for
pegzone-related transactions and determine whether or not they occurred.  In
the first iteration of the design this will use a light client approach, and in
a later iteration, reuse the components of [Zebra] as libraries.

To determine whether funds have been sent to the pegzone address, the
validators share an [incoming viewing key (IVK)][ivk].  To monitor outgoing
transactions, the validators also share an [outgoing viewing key (OVK)][ovk].
The Zcash protocol does not enforce that transactions have a valid OVK, so
validators must also monitor for improperly published note nullifiers.

## Security / Fraud

The pegzone is secured by capital staked on the behaviour of the pegzone
validators, whose stake can be slashed in the event of misbehavior, such as
unauthorized sends or failure to mint Cosmos-ZEC.  Because some behaviour may
not be traceable to a particular validator, overslashing may be required to
ensure correct incentives for each validator.  Validators will collect fees on
Cosmos-ZEC transactions in the pegzone to cover the cost of the capital
required to secure it.   Multiple pegzones can compete on fees.

Because each pegzone pegs *shielded* ZEC, not transparent ZEC, it acts as an
additional gateway between a transparent world (in this case, Cosmos) and the
shielded pool.  This increases the operational complexity of monitoring all
transactions in and out of the shielded pool, as those now happen in multiple
places.  It also provides increased privacy for all Zcash users, not just the
users of the pegzone, by increasing the size of the shielded pool.

However, `Cosmos->Zcash->Cosmos` transfers are likely to have similar problems
as `t->z->t` transfers.  For this reason, the pegzone will provide a
transaction planner tool that can prepare a transfer strategy (sharding over
transparent addresses and time) and produce audit proofs that it executed
correctly.  This allows users to maintain some amount of privacy outside of the
shielded pool while still retaining the ability to privately disclose proofs of
their behaviour to relevant authorities.

## Cosmos State Machine Components

1. Asset issuance module.
2. User Account balance module.
3. Asset burning module.
4. IBC implementation.
5. IBC fungible token transfer application protocol implementation.
6. On-chain processing of fraud proofs.

[Cosmos]: https://cosmos.network
[pegzone]: https://blog.cosmos.network/the-internet-of-blockchains-how-cosmos-does-interoperability-starting-with-the-ethereum-peg-zone-8744d4d2bc3f
[Zebra]: https://github.com/ZcashFoundation/zebra
[ivk]: https://zips.z.cash/protocol/protocol.pdf#addressesandkeys
[ovk]: https://zips.z.cash/protocol/protocol.pdf#addressesandkeys
