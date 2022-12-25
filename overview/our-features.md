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

The Locksmith Smart Wallet is a composable and extensible wallet that uses semi-fungible NFTs as wallet permissions instead of validating private key signatures against a specific wallet address. This enables a host of unique features that address pain-points of today's wallets:

1. **Private Key Agnostic:** Wallet action requires valid possession of the proper NFT "Key." Externally Owned Address (EOA) or Contract Addresses (CA) actors can access the wallet similarly as long as their public address holds the proper NFT key. This in essence enables many valuable features of [account abstraction](https://blog.pantherprotocol.io/ethereum-account-abstraction-everything-you-need-to-know/). The NFTs can be "[soul-bound](https://vitalik.ca/general/2022/01/26/soulbound.html)" to a specific address to prevent phishing, exploits, or scams.
2. **Distributed Asset Management:** Assets no longer have to reside at a singular EOA or CA address, but rather can be composed and orchestrated across any trusted collateral provider.
3. **Automation:** Funds and permissions can be safely transferred, made available for specific recipients, and utilized against the wallet without your signature or gas fees. ****&#x20;
4. **Deposit Control:** Prevent deposits, dusts, and scams from specific senders or token types.
5. **Open Development:** Commitment to composability enables developers to build and extend the wallet API to provide further automation and features on-chain and without permission.

### Key Management

Instead of requiring a private key signature with assets in-wallet to do business, the Locksmith Wallet provides access to deposits, withdrawal, distributions, delegation access, and event triggering through the valid possession of a specific [ERC-1155](https://eips.ethereum.org/EIPS/eip-1155) NFT token minted by the wallet owner.

The wallet is created when a user creates their own **root** key. A root key holder has unilateral permission to manage the full life-cycle of other keys:

1. **Create:** Root key holders can create additional unique ERC-1155 keys to hold, or distribute to others, or embed into contracts.&#x20;
2. **Copy:** Root key holders can copy any existing or extinct keys.
3. **Burn:** They can burn any currently minted key.&#x20;
4. **Soulbind:** Root key holders can bind keys to holders such that the key cannot be moved, sent, or stolen.

Keys can be held by end users in traditional wallets, on hardware ledgers, browser plug-ins, or any on-chain contract willing to accept and hold it for use. The capabilities and permissions of the key are managed by the root key holder at all times.

### Asset and Collateral Management

To overcome the challenge of using EOA private-keys to secure asset possession, the Locksmith wallet provides a composable on-chain wallet protocol that maintains balances and access rights for all on-chain assets.

**Ledger**

All on-chain assets have a unique identifier called the Asset Resource Name (ARN).  It is a keccak256 hash of the contract address of the token (or 0x0 for gas), the token standard (20/721/1155/etc), and the token ID, and thus unique by asset type.

All assets are tracked via their ARN and have withdrawal rights that are associated with Locksmith keys.

#### Notary

Any assets that that are deposited, re-distributed to key-holders, or withdrawn must be approved by the wallet's programmable Notary**,** whose wishes are controlled by the root key holder. &#x20;

#### Collateral Providers

The ledger tracks assets that are held at a specific trusted addresses. The total account is subdivided by the individual key withdrawal rights. By default, the platform supports:

* Ether Vault - An on-chain Ethereum vault that takes deposits and services withdrawals for valid keys as determined by the wallet's ledger.
* Token Vault - An on-chain ERC20 token vault that takes deposits and services withdrawals for valid keys as determined by the wallet's ledger.&#x20;
* NFT Vaults (Coming soon) - ERC721 and 1155 NFT vaults. Enable wallet key deposit and transfers, hold NFTs and delegate the airdrop.

#### Scribes

Wallet owners can allow programatic access to re-distribute funds..

