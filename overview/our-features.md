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

The Locksmith Smart Wallet uses semi-fungible NFTs as wallet permissions instead of validating private key signatures from a specific wallet address to access or move funds. This enables a host of unique features that address pain-points of today's wallets:

1. **Private Key Agnostic:** Locksmith wallet actions require valid possession of the proper NFT "Key." Externally Owned Address (EOA) or Contract Address (CA) actors can access the wallet similarly as long as their public address holds the proper NFT key. This in essence enables many valuable features of [account abstraction](https://blog.pantherprotocol.io/ethereum-account-abstraction-everything-you-need-to-know/). The NFTs can be optionally and mutably "[soul-bound](https://vitalik.ca/general/2022/01/26/soulbound.html)" to a specific address to prevent phishing, exploits, scams, pawning, or loaning against.
2. **Distributed Asset Management:** Assets no longer have to reside at a singular EOA or CA address, but rather can be composed and orchestrated across any trusted collateral provider.
3. **Automation:** Funds and permissions can be safely transferred, made available for specific recipients, and spent from the wallet without your signature or gas fees. ****&#x20;
4. **Deposit Control:** Prevent deposits, dusts, and scams from specific senders or token types.
5. **Open Development:** Commitment to composability enables developers to build and extend the wallet API to provide further automation and features on-chain.

The Locksmith Wallet's application of semi-fungible NFTs, on-chain collateral storage, and account abstraction produce a secure, configurable, and extensible wallet experience and platform.

### Key Management

Instead of requiring a private key signature with assets in-wallet to do business, the Locksmith Wallet provides access to deposits, withdrawal, distributions, delegation access, and event triggering through the valid possession of a specific [ERC-1155](https://eips.ethereum.org/EIPS/eip-1155) NFT token minted by the wallet owner.

The wallet is created when a user creates their own **root** key. A root key holder has unilateral permission to manage the full life-cycle of other keys for their wallet:

1. **Create:** Root key holders can create additional unique ERC-1155 keys to hold, or distribute to others, or embed into contracts.&#x20;
2. **Copy:** Root key holders can copy any existing or extinct keys.
3. **Burn:** They can burn any currently minted key.&#x20;
4. **Soulbind:** Root key holders can bind and unbind keys. Soulbound keys cannot be moved, sent, or stolen.

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
* NFT Vaults (Coming soon) - ERC721 and 1155 NFT vaults. Enable wallet key deposit and transfers, hold NFTs and delegate airdrop registration claims, etc.

Wallet users will trust individual collateral providers to their notary with a root key transaction, followed by the collateral provider claiming to have deposited funds on behalf of their wallet key. The wallet owner chooses which collateral provider to trust.

A marketplace of collateral providers can now offer to hold, deposit, and facilitate asset transfer with their own contracts. This unlocks a singular wallet experience encompassing portfolio of financial activity that could be simply vaulted, collateralized, staked, or loaned out.

#### Scribes

Wallet owners can allow programatic access to re-distribute funds between keys. By default, the platform includes a program that enables a specific key-holder to move funds from the root key to a restricted list of keys, called "Trustee".

The programatic interface between the notary and the ledger is robust enough to easily handle additional use cases deployed as a single permission-less smart contract:

* **Key Recovery:** Enable users to claim a recipient address for a specific key with a given code word and with required key approvals.&#x20;
* **Recurring Payments:** On a repeating time basis, move funds from one key to another given a sufficient key balance. Provide others mutable access to claim withdrawal rights from the appropriate collateral provider. Enable others to withdrawal securely from the origin of the funds. Unlocks allowance, timed inheritance, vesting schedules, monthly bill payments, etc.
* **Cross-Trust Transfers:** Ability to transfer between keys across locksmith wallets. In this scenario, the sender and recipient are on the same ledger and can move funds between their own key rights without actually moving the funds on-chain at the collateral provider. This requires that both wallets have the same collateral provider trusted at the time of transaction.

### Automation

Standard wallets require your private key signature as well as funds in-wallet to send to someone else. While it's possible to withdrawal from on-chain exchanges directly to a recipient's address, today's world still ties the access to funds across on-chain repositories to public address and private key pairs.

Locksmith wallet has created a few interfaces and patterns to enable your wallet to "work for you" while you are off-line and not signing transactions or spending gas. Notaries, Collateral Providers, and Scribes are all composable interfaces that can be swapped out to provide the wallet with a different permission model,  storage destination, or distribution scheme.

To facilitate offline automation, the wallet API has provided a few patterns by default.

#### Events

Events are one-time triggers that acts a logic gate for wallet-related decisions. Event's are registered by trusted dispatchers, whom act under their own designs and circumstances to declare the event as happened.&#x20;

In essence, events are a boolean oracle interface. By default the platform comes with the following dispatchers enabled, each designed to not require the wallet owner's interaction past set-up.

* **Key Oracle:** The wallet's root key holder can register an event that is fired at the direct will of another key-holder's attestation. This places the full trust of that event's completion to every holder of the designated key. Used either in high-trust personal scenarios (EOA), or outsourced to other logic as a soulbound contract NFT (CA).
* **Alarm Clock:** Set a date in the future when the alarm will be expired and can be challenged by _anyone_ willing to pay the gas for the transaction. A configurable "snooze" option can be enabled and guarded by requiring a specific key to snooze. Enables time-delay events, deadman switches, vesting schedules.

To register for a wallet, a dispatcher only has to be explicitly trusted to the Notary by the root key holder. Any contract or EOA actor willing to register and fire events to the wallet's event log can participate.

#### Agents





