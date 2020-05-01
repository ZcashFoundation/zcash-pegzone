# Bringing Privacy to Cosmos

The Zcash Foundation wants to bring privacy to the Cosmos ecosystem.  Zcash is
unique among privacy solutions in that it has strong network effects: new users
gain anonymity from all prior transactions of existing users, while in turn
contributing to a greater anonymity set for the entire system.  Our plan is to
take advantage of these network effects by giving Cosmos users access to this
anonymity set through an IBC-enabled pegzone.  This work will proceed in two
phases, with the design of the first phase enabling the features of the second
phase.  In the first phase, the pegzone will provide tokens backed by ZEC in
the existing Zcash shielded pool. These tokens can be sent throughout the
Cosmos ecosystem, allowing Cosmos users to trade and use ZEC.  In the second
phase, we plan to add a shielded pool to the pegzone itself, providing shielded
staking, shielded IBC assets, and shielded cross-chain transfers.  This plan
provides an increasingly useful privacy layer for the Cosmos ecosystem, while
growing the anonymity set of Zcash.

## What is a pegzone?

[Cosmos] is designed to enable cross-blockchain asset transfers.  These transfers
are accomplished by the Inter-Blockchain Communication (IBC) protocol, which
provides a standardized way to lock up assets on one chain and provide bearer
assets on another chain.

This provides horizontal scalability by allowing different “zones” –
blockchains with sovereign consensus mechanisms – to easily interoperate, or,
as the Cosmos slogan puts it, to provide an “internet of blockchains”.

IBC requires transaction finality on each of the chains. However,
proof-of-work systems only have probabilistic finality: if miners produce a
longer block chain, transactions could be removed. However, there's still a
conceptual gap between absolute finality required by IBC and the probabilistic
finality provided by a proof-of-work chain.

The gap is addressed by a [pegzone], a blockchain that works as an adapter for
probabilistic finality by declaring transactions to be final after some number
of confirmations.

## Project phasing

Our pegzone design proceeds in two phases, providing a minimum viable pegzone
in the first phase with a path to a full privacy layer for Cosmos in the second
phase.

- **Phase 1**.  The Zcash pegzone will provide an IBC-compatible asset, called
  PZEC, backed 1:1 with ZEC held in the Zcash shielded pool.  PZEC can be sent
  throughout the Cosmos ecosystem, traded and used in other zones, and redeemed
  for ZEC on the Zcash chain.  This allows Cosmos users access to the anonymity
  set of the Zcash shielded pool, with PZEC in Cosmos functioning similarly to
  Zcash t-addresses, while laying the groundwork for full shielding in the
  second phase using a novel shielded-compatible staking mechanism described
  below.

- **Phase 2**.  In the second phase, we'll add a Sapling-style shielded pool to
  the pegzone itself and implement shielded staking.  This allows shielded
  transfers from the pegzone to Zcash and vice versa.  We also intend to allow
  any IBC assets to move into the pegzone's shielded pool, coordinating to
  ensure that the ongoing Zcash user-defined-asset (UDA) support is
  IBC-compatible.

We plan to build the Zcash portions of this project using the [Zebra]
libraries, which provide modular, reusable components for working with the
Zcash chain.

## Phase 1 mechanism design

The pegzone will be a proof-of-stake chain.  The mechanism design for the
pegzone has three parts: the staking mechanics, the peg mechanics, and the fee
mechanics.

### Staking mechanics

As a proof-of-stake chain, the pegzone requires a staking token, and the
pegzone must be able to control the supply of the staking token.

However, rather than employ staking rewards as in the Cosmos Hub, we propose a
new design based on a pair of tokens, “SZEC” and “DZEC”, with a predetermined,
time-varying exchange rate.  The key advantage of this mechanism is that it is
future-compatible with shielded staking, by eliminating the requirement for
delegators to claim rewards.

The staking token is a new token called SZEC.  SZEC is obtained at a 1:1 ratio
by burning ZEC on the Zcash chain.  This avoids distributional issues about the
initial holders of the staking token: all ZEC holders have the option to obtain
SZEC if they choose to do so.  SZEC is always freely transferable, as it
represents an unstaked state of the staking token.

SZEC can be converted to DZEC by delegating it with a validator, and DZEC can
be converted to SZEC by removing it from delegation.  SZEC and DZEC are not
exchanged at a 1:1 rate, but at a blockheight-dependent rate `D(h) <= 1` which
measures the measures the cumulative depreciation of SZEC relative to DZEC from
genesis to blockheight `h` and decreases monotonically in `h`.

Delegating 1 SZEC at height `h_1` results in `D(h_1)` DZEC bonded to a
particular validator.  Undelegating 1 DZEC at height `h_2` results in
`1/D(h_2)` SZEC.  This transaction is only settled after some unbonding period,
during which the DZEC may still be slashed in the event of validator
misbehavior.

This can be thought of as treating all DZEC as if it had been delegated since
(pegzone) genesis, and pre-debiting the staking rewards over the period before
they began delegation, so that when they undelegate, they receive rewards only
over the delegation period.  Crucially, this means that all DZEC is fungible up
to the choice of validator, because there is no need to track how long
particular DZEC has been delegated.

This is economically equivalent to staking rewards as used on the Cosmos Hub,
but because the staking reward is instead priced in to the SZEC/DZEC exchange
rate, there is no requirement for delegators to claim rewards, and all
delegators are rewarded at the same rate (e.g., there is no question about the
compounding interval).  Removing staking rewards makes it relatively easy to
add shielded staking in phase 2 of the project, described in more detail below.

### Peg mechanics

The Zcash pegzone will provide an IBC-compatible asset, called PZEC, backed 1:1
with ZEC held in the Zcash shielded pool.  PZEC can be sent throughout the
Cosmos ecosystem, traded and used in other zones, and redeemed for ZEC on the
Zcash chain.

Zcash Sapling addresses have a [capability-based key hierarchy][saplingkeys],
splitting each logical capability related to that address's funds into a
different key.  The incoming and outgoing viewing keys will be replicated
across all validators, allowing any validator to individually inspect the
pegzone funds.  To authorize spending, the validators will share control of the
address' spend authorization key using [FROST], a round-optimized threshold
multi-signature scheme designed in collaboration between the Zcash Foundation
and the University of Waterloo.

Upon receipt and confirmation of a z2z transaction on the Zcash chain, the
validators issue PZEC to a pegzone address specified in the transaction's memo
field.  Pegzone users can redeem PZEC in the pegzone to obtain ZEC on the Zcash
chain, less some fees described below. To do this, they create a transaction on
the pegzone that burns PZEC and specifies a destination z-addr on the Zcash
chain.  As the pegzone validators reach consensus on the PZEC transaction, they
perform distributed signing on the spend authorization key to prepare a
shielded Zcash transaction that sends ZEC from the pegzone address to the
user-specified address.

We handle key rotation and validator set changes using a single epoch
mechanism.  The system fixes an epoch length parameter, measured in pegzone
blocks, and chosen to be a relatively short interval (e.g., approximately one
day).  A relatively short key rotation interval is preferable to a long one,
because it makes the key rotation mechanism impossible to ignore in client
software, reducing the risk of unexpected surprises.

Validator set changes can only occur at epoch boundaries, not at every block
(as in the Cosmos Hub).  Each epoch has a primary z-addr controlled by that
epoch's validator set.  The previous epoch's z-addr stays active until the end
of the current epoch, and its validators are responsible for rolling any funds
sent to it by mistake to the current epoch's address, while the next epoch's
z-addr is generated at the beginning of the current epoch so that it is
available in advance.  This provides a constant, pre-coördinated key rotation
mechanism, without requiring precise alignment between the pegzone blockheight
and users' clocks.

### Fee mechanics

The security of the pegzone is provided by the strength of the validator's
incentives for correct behaviour: their stake.  This means that the cost of
providing PZEC is the cost of capital staked to insure its security, integrated
over the length of time the PZEC is held in the pegzone.  It's important for
the fee structure to respect that cost structure, to prevent perverse
incentives for behavior on the part of validators or pegzone users.

As an example, someone who sends 100 ZEC to the pegzone, holds it in the
pegzone for a year, then redeems for ZEC should pay essentially the same fees
as someone who sends 100 ZEC to the pegzone and moves the corresponding PZEC
back and forth once per month.

In particular, the fee structure should not penalize movement across the peg
and into the shielded pool, because the first phase of the pegzone is
unshielded, so PZEC will function similarly to t-addrs in Zcash, where privacy
requires careful movements into and out of the Zcash shielded pool.

The proposed fee mechanism for PZEC is therefore similar to the staking
mechanism. Rather than a 1:1 rate, PZEC is converted to ZEC at rate `F(h) < 1`,
which measures cumulative fees from genesis to blockheight `h` and decreases
monotonically in `h`.

Sending 1 ZEC to the pegzone at (pegzone) height `h_1` results in issuance of
`1/F(h_1)` PZEC.  Redeeming 1 PZEC at (pegzone) height `h_2` results in
distribution of `F(h_2)` ZEC.

This can be thought of as treating all PZEC as if it had been pegged since
(pegzone) genesis, and pre-crediting the user for fees up to the time of
creation.  This design removes the requirement to track how long PZEC has been
held in the pegzone, while ensuring that the fees charged are related to the
cost of capital required to secure the peg.

The excess ZEC withheld in distribution is kept by the validators and their
delegators.

One disadvantage is that fees are not collected on an ongoing basis, but only
when assets move through the peg. However, because the fee amount is not
affected by when and how assets move through the peg, avoiding moving funds
does not help users avoid paying fees.

The fee rate should be as low as possible (to incentivize pegzone usage), but
high enough to cover the cost of capital required for security. One mechanism
to accomplish this would be an automatic fee adjustment analogous to the one
used on the Cosmos Hub to control staking rewards. This would fix a minimum
collateralization ratio for the pegzone, and increase the fee rate to
incentivize staking as the collateralization ratio declines towards the
minimum.

## Phase 2 mechanism design

In the second phase, we plan to add a Sapling-style multi-asset shielded pool
to the pegzone itself and implement shielded staking.  Shielded staking will
provide delegator privacy, not validator privacy.  Each validator will have a
publicly-visible stake weight, but unlike on the Cosmos Hub, the identities of
their delegators and the distribution of delegators to each validator will be
protected.  Validators can be pseudonymous, if there is market demand for
pseudonymous validators – no strong identity is required.  The shielding design
follows straightforwardly from the SZEC/DZEC design, which ensures that all
DZEC staked with the same validator is fungible.  

The pegzone has a main multi-asset shielded pool for SZEC and any other IBC
assets moved into the shielded zone, as well as a single-asset shielded
delegation pool for each validator's DZEC.  Delegation transactions move SZEC
from the main shielded pool into the validator's delegation pool, escrowing the
portion of the delegated funds that will be slashed in case of validator
misbehavior.  A user can undelegate their funds by moving funds back to the
main shielded pool.  The unbonding period of the Cosmos Hub can be replicated
by requiring a long settlement period for this transaction.  Slashing is
implemented by burning all of the escrowed portion of the delegated funds,
allowing users to withdraw the rest.

[Cosmos]: https://cosmos.network
[pegzone]: https://blog.cosmos.network/the-internet-of-blockchains-how-cosmos-does-interoperability-starting-with-the-ethereum-peg-zone-8744d4d2bc3f
[Zebra]: https://github.com/ZcashFoundation/zebra
[saplingkeys]: https://zips.z.cash/protocol/protocol.pdf#addressesandkeys
[FROST]: https://crysp.uwaterloo.ca/software/frost/
