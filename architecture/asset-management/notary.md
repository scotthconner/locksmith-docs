---
description: Permissioned Interface System
---

# 🪶 Notary

## Design Ethos

The Notary is a contract that holds permissions for each critical wallet action. Where-as with a Gnosis Safe that has a group of owners and optional modules, Locksmith uses a progressively granular permission scheme that allows for maximum flexibility while maintaining a single integration pattern.

The Notary is designed to approve actions that operate on a wallet's state, like moving collateral or firing events. To approve any action the notary must determine that the action meets acceptable parameters as outlined by a wallet root key holder.

As of the first version, the Notary includes the following permissions:

* **Collateral Providers:** Contract addresses that adhere to the `ICollateralProvider` interface and are trusted by the wallet's root key-holder to custody funds safely on-chain.
* **Scribes:** Contract Addresses that are trusted to move funds between keys in a wallet.
* **Dispatchers:** Contract addresses that are trusted to register and fire events within the wallet.

&#x20;

## Storage



## Operations

