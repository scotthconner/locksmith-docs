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

## Architectural Features

The design decisions results in a few characteristics that define much of its operation:

1. **Logical Deployment Sequence.** Composability leads to a clear dependency graph that can be managed both architecturally and procedurally as part of the wallet owner's trust logic.
2. **Clear Security Model.** The sequence of "external actors" is clear - the wallet owner trusts everything an individual contract depends on, and each contract's operational role to distrust all external input.&#x20;
3. **Iterative Orchestration.** Each contract leverages the power of those below it and thus more sophisticated features are often contracts with little logic and state, but many dependencies.

## Known Trade-Offs

There are a few known trade-offs that at this point:

1. **Gas-less transactions.** Without a relayer or a functioning EIP-4337 environment, the best we would be able to do here is automatically refunding your traditional wallet at the end of any virtual wallet transaction. For sustainability reasons, paying your own gas is a trade-off for full de-centralization.
2. **Storage Costs.** Wallet storage, while minimal - does cost gas. There isn't a centralized server running a wallet indexer and serving it up easily so some gas is taken to store valuable data for introspection later. This gas cost is a two way door with initial upgradeability to eliminate storage requirements. To foster an environment that may not need on-chain storage, the program is also thoroughly designed to be able to completely re-build the entire on-chain state from reading events.
3. **Deployment Costs.** In combination with initial upgradeable contracts, having multiple composable and orchestrated contract deployments could be cost prohibitive depending on the network environment.
4. **RPC Load.** Balancing contract byte-code, and on-chain storage/gas costs with the need to serve a full application of the contract APIs leads to some short but plentiful roundtrips to the node to read public view contract state.

## Contract Platform Diagram

Below is a logical view of the contracts, and where they sit in the Locksmith Virtual Wallet application stack. Further portions of this document will describe each of them in  detail.

<figure><img src="../.gitbook/assets/Locksmith Architecture (2).png" alt=""><figcaption></figcaption></figure>

Logically, the diagram starts at (0, 0) on the bottom left. The black arrows denote a logical deployment sequence, although it is not the only viable dependency path.

### Permissions Layer

Locksmith is a wallet that is designed around using NFTs as permissions. For this, it is the foundational layer.&#x20;

These components provide a robust permission platform and wallet ledger. It consists of four contracts:

1. **KeyVault:** This is the base ERC1155 contract, which enables only a single trusted party to mint (the locksmith), burn, or soulbind tokens. It also contains the supply storage and enforces soulbound behavior by reverting token transfers. It doesn't understand anything other than to create and track supply of the ERC1155 NFTs based on whatever its "respected" Locksmith says.
2. **Locksmith:** Once respected, the Locksmith can utilize the KeyVault to service end-user requests for managing keys. The Locksmith contains the "trust" logic of the wallet, including who is the trust owner (root NFT key holders), and which keys are subservient to which trusts. The locksmith enforces that only a root key can manage keys, and they can only manage keys they've created with that root key. In essence, it provides an administrator interface for a set of NFT keys to anyone motivated to create one.
3. **Notary:** The Locksmith contract itself does not hold permissions against individual keys. Instead, those permissions are created with module extensions that are configured by a root key holder. The Notary is a specific permission manager for the wallet's _collateral_. It maintains a set list of trusted contract addresses (modules) that can optionally deposit funds, withdrawal funds, or re-distribute funds amongst keys. These addresses are managed by root key holders and apply only to their trust model.
4. **Ledger:** All assets, whatever they are, have access rights associated with them. Access rights are determined by which particular NFT key has access. Because the root key owner can mint many of a single key, it immediately creates a virtual fund pool for anyone holding that key. The ledger keeps track of a wallets balance per asset, per provider, per key. Only actors previously notarized by the root key holder can deposit, withdrawal, or otherwise move funds or change access rights.

### Collateral Layer

The next layer takes full advantage of key-holder mechanics, the notary, and ledger by providing the ability to manage the basic lifecycle of collateral: depositing funds, safely re-distributing access rights, and eventually withdrawaling funds.

This layer also consists of 4 contracts:

1. **EtherVault.** A simple vault that holds only ether, and allows only a key-holder to deposit funds into contract for on-chain storage. The key-holder must belong to a trust that has previously entrusted the vault's contract with deposit rights via the Notary. This model ensures that any key-holder depositing funds into a Locksmith wallet collateral provider must be trusted by the wallet's owner first. The contract respects Ledger's key balances, along with mechanisms to detect discrepancies, to facilitate withdrawals. In this way, the EtherVault indeed is the vault of Ether, but the Ledger keeps track of who owns what.
2. **TokenVault.** Similar to EtherVault, with an explicit interface for ERC20 tokens. Instead of a payable interface for deposits, the vault takes a token contract address, and expects a proper approval before hand. It also respects the ledger balance for key rights and fulfill withdrawals.
3. **TrustEventLog.** To facilitate re-distribution logic, we want to be able to keep track of events that need to or have occurred within the wallet's lock-logic. This introduces a _Dispatcher_ model, which we will more fully explore in the application later.
4. **Trustee**. A basic recovery module that acts as a trusted scribe on the ledger. It enables a key-holder to re-distribute funds from the root key to a strictly defined list of keys at their discretion. The ability to distribute funds is gated behind optional event requirements, which it trusts the TrustEventLog to faithfully provide.
