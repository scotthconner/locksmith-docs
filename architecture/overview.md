---
description: A Smart Contract Wallet Architecture Summary
---

# 🗺 Overview

There are a few design considerations that were made early on that heavily influence the approaches taken that need to be considered.

## Design Considerations

### Infrastructure

The Locksmith Virtual Wallet system operates entirely as a set of verified on-chain Solidity smart contracts. There are no relayers, servers, or indexers associated with its operation. Once deployed, it's full continued operation is contingent only on the viability and security of the network it is deployed on.

To reduce overall complexity and increase security, most of the smart contract stack operates as  singleton contracts. There are also components like Collateral Providers, Scribes, and Dispatchers of which can be deployed per user.&#x20;

### API Design

Each smart contract plays a defined role in the overall system, and thus contains the APIs needed to fully encapsulate the role but nothing more.&#x20;

The design concept is also to be able to build a UX easily from a non-sophistcated client. This means that desired UX and workflow management needed to be accessible directly from public view functions on the smart contracts themselves.&#x20;

Each contract maintains the unique state it needs to and avoids duplicating state from other actors by understanding dependencies at deployment.

### Composability

To ensure that Locksmith could be easily extensible by anyone, each component needs to make the fewest assumptions possible as part of their role within the system.

## Architectural Features

The design decisions results in a few characteristics that define much of its operation:

1. **Logical Deployment Sequence.** Composability leads to a clear dependency graph that can be managed both architecturally and procedurally as part of the wallet owner's trust logic.
2. **Clear Security Model.** The sequence of "external actors" is clear - the wallet owner trusts everything an individual contract depends on, and each contract's operational role is to distrust all external input.&#x20;
3. **Iterative Orchestration.** Each contract leverages the power of those below it and thus more sophisticated features are often contracts with little logic and state.

## Known Trade-Offs

There are a few known trade-offs that at this point:

1. **Gas-less transactions.** Without a relayer or a functioning EIP-4337 environment, the best we would be able to do here is automatically refunding your traditional wallet at the end of any virtual wallet transaction. For sustainability reasons, paying your own gas is a trade-off for full de-centralization.
2. **Storage Costs.** Wallet storage, while minimal - does cost gas. There isn't a centralized server running a wallet indexer and serving it up easily at the expense of investors. Thus, some gas is taken to store valuable data for introspection later. This gas cost is a two way door with initial upgradeability to eliminate storage requirements. To foster an environment that may not need on-chain storage, the program is also thoroughly designed to be able to completely re-build the entire on-chain state from reading emitted events.
3. **Deployment Costs.** In combination with initial upgradeable contracts, having multiple composable and orchestrated contract deployments could be cost prohibitive depending on the network environment.
4. **RPC Load.** Balancing contract byte-code, and on-chain storage/gas costs with the need to serve a full application of the contract APIs leads to some short but plentiful roundtrips to the node to read public view contract state.

## Contract Platform Diagram

Below is a logical view of the contracts, and where they sit in the Locksmith Virtual Wallet application stack. Further portions of this document will describe each of them in  detail.

<figure><img src="../.gitbook/assets/Locksmith Architecture.png" alt=""><figcaption></figcaption></figure>

Logically, the diagram starts at (0, 0) on the bottom left. The black arrows denote a logical deployment sequence, although it is not the only viable dependency path.

### Permissions Layer

Locksmith is a wallet that is designed around using NFTs as permissions. For this, it is the foundational layer.&#x20;

These components provide a robust permission platform and wallet ledger. It consists of four contracts:

1. **KeyVault:** This is the base ERC1155 contract, which enables only a single trusted party to mint (the locksmith), burn, or soulbind tokens. It also contains the supply storage and enforces soulbound behavior by reverting token transfers. It doesn't understand anything other than to create and track supply of the ERC1155 NFTs based on whatever its "respected" Locksmith says.
2. **Locksmith:** Once respected, the Locksmith can utilize the KeyVault to service end-user requests for managing keys. The Locksmith contains the "trust" logic of the wallet, including who is the trust owner (root NFT key holders), and which keys are subservient to which trusts. The locksmith enforces a paradigm that only a root key can fully administrate their own active key collection.
3. **Notary:** The Locksmith contract itself does not hold permissions against individual keys. Instead, those permissions are created with module extensions that are configured by a root key holder. The Notary is a specific permission manager for the wallet's _collateral_. It maintains a set list of trusted contract addresses (modules) that can optionally deposit funds, withdrawal funds, re-distribute funds amongst keys, or trigger events that effect the ledger balance. These addresses are managed by root key holders and apply only to their trust model.
4. **Ledger:** All assets, whatever they are, have access rights associated with them. Access rights are determined by which particular NFT key has access. Because the root key owner can mint many of a single key, it immediately creates a virtual fund pool for anyone holding that key. The ledger keeps track of a wallets balance per asset, per provider, per key. Only actors previously notarized by the root key holder can deposit, withdrawal, or otherwise move funds.

### Collateral Layer

The next layer takes full advantage of key-holder mechanics, the notary, and ledger by providing the ability to manage the basic lifecycle of collateral: depositing funds, safely re-distributing asset rights, and eventually withdrawing funds.

This layer consists of 4 contracts:

1. **EtherVault.** A simple vault that holds only ether, and allows only a key-holder to deposit funds into contract for on-chain storage. The key-holder must belong to a trust that has previously entrusted the vault's contract with deposit rights via the Notary. This model ensures that any key-holder depositing funds into a Locksmith wallet collateral provider must be trusted by the wallet's owner first. The contract respects Ledger's key balances, along with mechanisms to detect discrepancies, to facilitate withdrawals. In this way, the EtherVault indeed is the vault of Ether, but the Ledger keeps track of who owns what.
2. **TokenVault.** Similar to EtherVault, with an explicit interface for ERC20 tokens. Instead of a payable interface for deposits, the vault takes a token contract address, and expects a proper approval before hand. It also respects the ledger balance for key rights and fulfill withdrawals.
3. **TrustEventLog.** To facilitate re-distribution logic, we want to be able to keep track of events that need to or have occurred within the wallet's lock-logic. This introduces a _Dispatcher_ model, which we will more fully explore in the application layer.
4. **Trustee**. A basic recovery module that acts as a trusted scribe on the ledger. It enables a key-holder to re-distribute funds from the root key to a strictly defined list of keys at their discretion. The ability to distribute funds is gated behind optional event requirements, which it trusts the TrustEventLog to faithfully provide.

### Application Layer

The application layer is where we begin to expose specific wallet functionality. The foundational permission and collateral storage layer, along with the notary unlocks the ability to build applications that virtualize asset movements amongst a collection's NFT holders.

This layer consists of 4 contracts:

1. **KeyOracle.**  The Key Oracle operates as a `Dispatcher`, an agent at a specific address entrusted by the root key holder to register and emit events into the wallet's event log. A "Key Oracle" provides a workflow enabling a root key holder to anoint a key within their collection to be able to "fire" a specific event by signing a transaction. It turns the key holder into an oracle for whatever that event is meant to signify. The wallet owner is fully entrusting whomever may hold the key to faithfully register the event whatever it may be. This could take the form of attestations, proof-of-access, life event's registered by trustees, or wallet integrations.
2. **AlarmClock.** Like the Key Oracle, the Alarm Clock operates as a `Dispatcher` and registers and fires events into the event log. The event is configured by the root key holder, and is designed to act like a snooze-able alarm clock. At a specific date in the future, the event can be openly "challenged" by anyone willing to sign the transaction and the event will fire. However, if an anointed key-holder  snoozes the alarm a pre-defined length of time will be added to the challenge period. This enables proof-of-life, one-time vesting schedules, wallet recovery, and other use cases. Combined with a configuration on the **Trustee** contract, easily enables a traditional trust-estate proceeding on anything still against the root key.
3. **VirtualKeyAddress.** This contract is a product of a `CREATE2` factory call on behalf of a given root key holder and is thus unique for the Locksmith key. This enables a virtual CA address to be used as a "wallet identity". The contract has an owner and an identity, both of which are specified by the root key holder. The owner owns the contract, and the contract operates as the virtualized  "smart wallet" and can send and receive like a normal wallet. Additional functionality like multi-call are implemented. Additional functionality can include deposit blocking, send-blocking, spam filtering, NFT delegation. The contract satisfies the security model of the platform by accepting and holding a soulbound key.&#x20;
4. **KeyAddressFactory**. A factory contract is included that creates per-key VirtualKeyAddresses on-chain. The way it operates is a root key holder (and only a root key holder) temporarily sends their root key into the factory with a specific owner and identity key selected. The factory uses a `CREATE2` call to deploy a new "key inbox." Once deployed, the factory uses the root key to mint a copy of the identity key and soul-binds it to the newly created contract address, enabling the inbox to act as if it was the key-holder itself. It then also registers the new contract address with the Post Office for visibility At the very end of the transaction, the factory sends the root key back to the caller.

### Orchestration Layer

The Locksmith contracts are numerous and while many are individually useful, their behavior in concert is what unlocks the power of a permission-based management system. In order to maintain the benefits of this, while still providing feasible user experiences, a meta-layer of orchestration can utilize many Locksmith dependencies to build robust functionality and complex single-transaction workflows.

1. **Post Office.** The Key Factories generate smart contract "inboxes" on demand assuming there isn't one that already exists in the Post Office. The Post Office stores, maintains and protects data about keys, their addresses, and which key owns the contract. Only root keys can register inboxes with the Post Office, and they can only register inboxes for keys that are within their trust model. This enables wallet functionality, front-ends, and even other contracts to quickly locate the inbox contract for any existing Locksmith key. It also decouples the inbox creation and management from the life-cycle of the key itself.
2. **Trust Creator.** To build out a full wallet, a lot of smart contract calls must be orchestrated. The Trust Creator can configure everything necessary for a secure, recoverable, and inheritable wallet in a single transaction. Within the transaction, the TrustCreator will create the trust and root key, entrust the EtherVault, TokenVault, Trustee, Alarm Clock, and Key Oracle contracts as trusted with the Notary, create Trustee and Beneficiary NFT keys and send them to their proper destinations, soulbind them if required, and configure a proof-of-life event on your behalf for inheritance/recovery. It then creates the VirtualKeyAddress via the factory and adds it to the PostOffice for registration. After which, the user will be given the resulting root key back from the creator. This enables a caller to create a recoverable wallet that supports Ether and ERC20 tokens (NFTs coming soon) with a controllable on-chain identity.
