# Stateless Tech Tree

A 'roadmap' delineates centrally determined deliverables and milestones. Because the Stateless Ethereum effort is not centrally planned, it's helpful to instead use a [tech tree](https://en.wikipedia.org/wiki/Technology_tree) to organize and record ongoing work in the Stateless Ethereum initiative. This helps to map out a shared understanding of 'the critical path' and dependencies in research / implementation.

![tech_tree](./assets/Stateless_Tech_Tree_08-2020.png)

> ### Key
> * **Red (milestone)**: Major milestone of Stateless Ethereum.
> * **Purple (rounded and square corners)**: Specific pre-requisites for a major milestone. Any topic that could or will eventually be refined into an EIP, as well as anything considered on 'the critical path'.
> * **Yellow (rounded)**: Support work needed for a major milestone, such as software tools or data analysis.


## Witness Prototype

It's possible to build a stateless client today with the current network primitives; beam sync is an example of a semi-stateless client implementation. The objective of this milestone is to have implemented clients on mainnet passing around prototype witnesses. It's likely that the first witnesses will be too large to propagate efficiently, but a first implementation will be a good starting point to improve upon through other objectives such as code merkleization and a binary state trie.

The starting point for a witness format prototype is the [Witness Specification](./witness.md), which aims to define witnesses without ambiguity for the sake of soundness in implementation.

The specification is still in draft form and contributions, especially in writing tests, are welcome.

In addition to the specification, there are two noteworthy issues that a witness implementation must 'solve' in some way.

#### Witness Indexing

We need a reliable way of determining which witness corresponds to which block and associated state. This could be as simple as putting a `witnessHash` field into the block header, or something else that serves the same purpose but in a different way.

## Binary State Trie Transition

In theory converting the existing [hexary patricia trie] to a binary format could reduce the size of a witness substantially, and this has been [supported by early research](https://medium.com/@mandrigin/stateless-ethereum-binary-tries-experiment-b2c035497768). Re-computing the Ethereum state as a binary trie is a complex transition to manage, and there are at the moment a few proposals for effecting that change.

See this ethresearch post for the current state of [the binary trie format](https://ethresear.ch/t/binary-trie-format/7621)

#### Merkelization rules
The central decision point of binary trie formatting is the merkleization rule, i.e. how the root hash is calculated based on the types of nodes in the trie. See [this ethresearch post](https://ethresear.ch/t/binary-trie-format/7621/18?u=gichiba) for data and updates.

#### Overlay method

Although there are a few abstract strategies for transitioning to a binary trie, the [one that is currently being worked on](https://medium.com/@gballet/ethereum-state-tree-format-change-using-an-overlay-e0862d1bf201) entails a transition period in which the new (partial) binary state captures all *new* changes to the state, while the (read only) hexary trie remains. Only after the entire state has been converted and verified in the background is the legacy hexary structure abandoned entirely.


#### Code Merkleization

One major component of a witness is accompanying code. Merkelization means splitting up contract bytecode so that only the portion of the code called is required to generate and verify a witness for the transaction. This could have substantial effects on  [the size of witnesses](https://ethresear.ch/t/some-quick-numbers-on-code-merkelization/7260/3)

Some work has been done on possible strategies for code chunking rules, such as relying on `JUMPDEST` instructions to prevent unverified jumps during execution, and it's thought that this feature is one of the first able to be formulated into an EIP for inclusion.

## reGenesis

The best place to get a handle on the reGenesis proposal is [this explanation by @mandrigin](https://medium.com/@mandrigin/regenesis-explained-97540f457807) or Alexey's [reGenesis plan](https://ledgerwatch.github.io/regenesis_plan.html), but the TL;DR is that reGenesis is essentially "spring cleaning for the blockchain". The full state would be conceptually divided into an 'active' and an 'inactive' state. Periodically, the entire 'active' state would be de-activated and new transactions would start to build an active state again from *almost* nothing (hence the name "reGenesis"). If a transaction needed an old part of state, it would provide a witness very similar to what would be required for Stateless Ethereum: a Merkle proof proving that the state change is consistent with some piece of inactive state. If a transaction touches an ‘inactive’ portion of the state, it automatically elevates it to ‘active’ (whether or not the transaction is successful) where it remains until the next reGenesis event. This has the nice property of creating some of the economic bounds on state usage that state rent had without actually deleting any state, and allowing transaction sender unable to generate a witness to just blindly keep trying a transaction until everything it touches is ‘active’ again.

## EVM

Stateless Ethereum will require a change to EVM semantics, particularly in gas execution and opcode repricing.  Because the EVM is currently an implicit (and not explicit) Ethereum State Transition Function, changes to semantics can be quite complex, and will likely affect future work on Eth1x as well as Eth2.

#### Witness Gas Accounting

Generating a block’s witness involves some computation that will be performed by the block’s miner, and therefore will need to have an associated gas cost, paid for by the transaction’s sender. Currently there is little more than a "finger in the air" estimate of what the gas costs for witness production should be. The problem requires substantive analysis of EVM execution and historical blocks.

An unfortunate side-effect of opcode repricing is potential breaking changes in smart contracts that have built-in assumptions about gas costs.  Various strategies have been proposed to mitigate this negative side effect, with discussion centered around making gas used *unobservable* in whole or in part.

## State Network

The Ethereum state is 40-100GB, depending on how you store it. The de facto network primitive for state sync is `getNodeData`, which requests the state trie node by node from a pool of (full node) peers. This is a slow and inefficient method of sync that puts a heavy burden on fully synced nodes serving greedy leechers.

An alternative network primitive is sought that can eliminate or at least substantially reduce reliance on `getNodeData` for state sync.

In the mean time, a more practical network based on [discovery v5.1](https://github.com/ethereum/devp2p/pull/157) is being investigated to serve historical chain data, as well as reGenesis 'inactive' state.

#### SNAP

A promising auxilary protocol meant to be run side-by-side with the eth protocol, not standalone (e.g. chain progression is announced via eth). The `snap` protocol permits downloading the entire Ethereum state without having to download all the intermediate Merkle proofs, which can be regenerated locally.

[The snap protocol](https://github.com/ethereum/devp2p/blob/3fe9713658f3b3b56e4e99493c54f313e11b43a0/caps/snap.md) is currently in an early version, but may be the most promising alternative to current sync strategies, and a viable path to improving state sync

#### Merry-go-round

At a high level, [MGR](https://ethresear.ch/t/merry-go-round-sync/7158) would operate by enumerating the full state in a predetermined order and gossiping this data among the clients which are actively syncing. For a client to fully sync it needs to “ride” one full rotation of the merry-go-round. There has been some discussion around the best way to organize the state such that it can be 'rotated' through, as well as how storage tries and code blocks will be handled when breaking the full state into several similarly sized pieces for sync.

## Misc.

#### Stateless tx Validation

This is an interesting early problem [thoroughly summarized on the ethresearch forums](https://ethresear.ch/t/the-current-transaction-validation-rules-require-access-to-the-account-trie/7046). In summary, clients need to quickly check if incoming transactions (waiting to be mined into a future block) are at least *eligible* to be included in a future block.

#### Flat Database layout

Most Ethereum clients (including geth) use the "naive" database layout in which trie nodes are stored (hash, node) as the `[key, value]` pairs in the DB. [Turbo-geth](https://github.com/ledgerwatch/turbo-geth) is a notable exception, which uses a different scheme to encode some structural information into the key value pairs, making some operations much more efficient for the client.

It is thought that a flat database layout will be more capable of computing and seeding the new trie structure in real time for clients working under the "naive" layout (whom are unable to compute both states at the same time).
