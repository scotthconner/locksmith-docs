---
description: A Smart Contract Wallet Architecture Summary
---

# 🗺 Overview

There are a few design considerations that were made early on that heavily influence the approaches taken that need to be considered.

## Design Considerations

### Infrastructure

The Locksmith Virtual Wallet system operates entirely as a set of verified on-chain Solidity smart contracts. There are no relayers, servers, or indexers associated with its operation. Once deployed, it's full continued operation is contingent only on the viability and security of the network it is deployed on.

This also means that state introspection for viable end-user workflows require additional byte-code in the contracts and additional gas to store the state for later view calls. As an example, end user strings are thus stored as limited `bytes32.`While the application will emit plenty of events there is still gas spent for some on-chain storage.

### API Design

Each smart contract plays a defined role in the overall system, and thus contains the APIs needed to fully encapsulate the role but nothing more.&#x20;

The design concept is also to be able to build a UX easily from a non-sophistcated client. This means that desired UX and workflow management needed to be accessible directly from public view functions on the smart contracts themselves. Additional contract byte-code, deployment costs, and gas are trade-offs balanced along the way. This comes in the forms of APIs like `Ledger::getContextArnBalanceSheet`, that vastly simplify frontend design patterns without having to listen to events and run an indexer at the trade-off on byte-code, and in some cases scalability (in the presented case, that would some O(n) operations on views).

### Composability

To ensure that Locksmith could be easily extensible by anyone, each component needs to make the fewest assumptions possible as part of their role within the system.\
\
This is why the `Locksmith` and the `KeyVault` are not a single ERC1155 contract. Contract size limits aside, separating out the concerns also enables the development team to incrementally lock the upgradeability of a layer once it has ossified.

&#x20;It's also why the `Notary` takes trusted public addresses for `Scribes` and `Dispatchers` instead of Locksmith key requirements. In this case, the Root key holder entrusting the address is the initial pledge that matters in that transaction. Requiring that the operator holds a key is a decision deferred to the logic of the trusted address.

### Architectural Features

The design decisions results in a few characteristics that define much of its operation:

1. **Logical Deployment Sequence.** Composability leads to a clear dependency graph that can be managed both architecturally and procedurally as part of the wallet owner's trust logic.
2. **Clear Security Model.** The sequence of "external actors" is clear - the wallet owner trusts everything an individual contract depends on, and each contract's operational role to distrust all external input.&#x20;
3. **Iterative Orchestration.** Each contract leverages the power of those below it and thus more sophisticated features are often contracts with little logic and state, but many dependencies.

### Known Trade-Offs

There are a few known trade-offs that at this point:

1. **Gas-less transactions.** Without a relayer or a functioning EIP-4337 environment, the best we would be able to do here is automatically refunding your traditional wallet at the end of any virtual wallet transaction. For sustainability reasons, paying your own gas is a trade-off for full de-centralization.
2. **Storage Costs.** Wallet storage, while minimal - does cost gas. There isn't a centralized server running a wallet indexer and serving it up easily so some gas is taken to store valuable data for introspection later. This gas cost is a two way door with initial upgradeability to eliminate storage requirements. To foster an environment that may not need on-chain storage, the program is also thoroughly designed to be able to completely re-build the entire on-chain state from reading events.
3. **Deployment Costs.** In combination with initial upgradeable contracts, having multiple composable and orchestrated contract deployments could be cost prohibitive depending on the network environment.
4. **RPC Load.** Balancing contract byte-code, and on-chain storage/gas costs with the need to serve a full application of the contract APIs leads to some short but plentiful roundtrips to the node to read public view contract state.

### Contract Platform Diagram

Below is a logical view of the contracts, and where they sit in the Locksmith Virtual Wallet application stack. Further portions of this document will describe each of them in  detail.

