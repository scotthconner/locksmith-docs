---
description: The Locksmith Trust Model is the backbone of security.
---

# 🔑 Permissions

## Locksmith Trust Model

Locksmith uses a singleton ERC-1155 token contract to model permission possession called the "Key Vault". The tokens, aka "Keys", behave like a normal ERC-1155 and can be stored at an address, displayed in traditional wallet, sent, received, and handled.&#x20;

On its own, the KeyVault only manages existing supply by keeping tracks of holders and enforcing soulbound token parameters by preventing transfers. Minting, soul-binding, and burning are at the full discretion of an anointed commanding "Locksmith"  set by the contract's owner. This keeps the logic and contract size of the KeyVault down and delegates all other business logic explicitly upwards in the stack.

The Locksmith contract acts as a Key Minter. Any message sender can create a key for their own independent security model, called a "trust." The message sender will be given an ERC-1155 custom minted as the root key for their permission model. From there, the Locksmith will respect any NFT management commands for your collection as long as you are holding a valid copy of your root key.

<figure><img src="../../.gitbook/assets/Locksmith Architecture - Page 2.png" alt=""><figcaption><p>A wallet has a single root key.</p></figcaption></figure>

## Root Key

A **root key** is generated when a caller invokes the Locksmith to create a new "Trust." By default, every new trust mints a root key. The root key provides permissions to the system that no other key can do on behalf of their trust:

* Mint, copy, bind, or burn keys within the trust.
* Register or de-register trusted Collateral Providers, Scribes, or Dispatchers with the Notary.
* Create Key Inboxes and register them with the post office.
* Set up rules within the Trustee, Alarm Clock, or Key Oracle feature contracts.
* Extend functionality with their own permission modules or use case implementations.

In effect, the root key controls not only the set of active permissions themselves, but who has them, as well as what actors are capable of doing what within the system.&#x20;

## Ring Keys

A Ring key is a key that belongs to and is controlled by a root key. A wallet owner could have many motivations to create multiple ring keys:

* **"Air-locking" Ledgers.** Having a root key that owns a full treasury, use a second ring key that are only able to transfer funds from root to a third key. The third key is a hot wallet that uses the funds, operates with dApps, or safely delegates drops via a virtual address.
* **Inheritance.** Same scheme as air-locking a ledger with a warm and hot wallet access, with a dead-man-switch configuration on an Alarm Clock event to enable the secondary key distribution rights.
* **Recurring Payments.** As part of a pledge with a service provider subscription, provide them a key that gives access to periodic funding rights from your wallet that you control.

There are also advanced considerations around the power of secondary keys. One can secure privilege escalation by soul-binding a root key into a contract that enables it to act as a root holder in one specific manner, including requiring a secondary key.

In the next sections we will go into each contract in depth.
