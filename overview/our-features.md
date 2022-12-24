---
description: An extensible on-chain smart wallet solution
---

# 📽 Introduction

## Solution Space

In order to make progress on the usability and security challenges of managing crypto-based assets, we need to change the relationship between private keys and access to crypto assets.

A novel solution would then have the following traits:

1. **On-chain.** Do not rely on centralized off-chain actors, companies, custodial outfits, "Web2" based services, servers, or deployments.
2. **Self custodial.** Maintain full control of and visibility into your assets at all times without enabling long standing approval permissions.
3. **Recoverable.** Losing a private key, or leaving crypto-assets as inheritance through a will needs to be secure and reliable. This includes cases where assets are staked, locked up, loaned, borrowed, etc.
4. **Extensible.** Users should not be limited in what they can do, and should enable others to securely interact with their funds. ****&#x20;

## Introducing Locksmith Smart Wallet

The Locksmith Smart Wallet is built of composable units that orchestrate into a secure on-chain, fully self custody wallet with programmable traits.

### Key Management

Instead of requiring a private key signature with assets in-wallet to do business, the Locksmith Wallet provides access to deposits, withdrawal, distributions, delegation access, and event triggering through the valid possession of a specific [ERC-1155](https://eips.ethereum.org/EIPS/eip-1155) NFT tokens.

The wallet is created when a user creates their own **root** key. A root key holder has unilateral permission to manage the full life-cycle of other keys:

1. **Create:** Root key holders can create additional unique ERC-1155 keys to hold, or distribute to others, or embed into contracts.&#x20;
2. **Copy:** Root key holders can copy any existing or extinct keys.
3. **Burn:** They can burn any currently minted key.&#x20;
4. **Soulbind:** Root key holders can bind keys to holders such that the key cannot be moved, sent, or stolen.

Keys can be held by end users in traditional wallets, ledgers, browser plug-ins, or any on-chain contract willing to accept and hold it for use. The capabilities and permissions of the key are managed by the root key holder at all times.

### Asset and Collateral Management

To overcome the challenge of using EOA private-keys to secure asset possession the Locksmith wallet provides a composable on-chain wallet protocol that tracks and maintains balances and access rights for all on-chain assets.

**Ledger**

All on-chain assets have a unique identifier called the Asset Resource Name (ARN).  It is a keccak256 hash of the contract address of the token (or 0x0 for gas), the token standard (20/721/1155/etc), and the token ID, and thus unique by asset type.

All assets are tracked via their ARN and have withdrawal rights that are associated with Locksmith keys.&#x20;

#### Notary

Any assets that that are deposited, re-distributed to key-holders, or withdrawn must be approved by the wallet's programmable Notary**,** whose wishes are controlled by the root key holder. &#x20;

