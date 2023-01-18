---
description: Asset Rights Decoupling for Virtualized Wallets
---

# 📖 Ledger

## Design Ethos

Having an extensible asset storage interface drives some important goals:

* **Decouple storage from Rights.** The on-chain storage location shouldn't have to care about the internal logic of sub-account distribution. Decoupling these concerns enables other programs to move assets between keys for various reasons without requiring the collateral provider's involvement.
* **Asset and Use Case Diversity.** Support different types of assets, storage, and yield schemes. Specialty NFT vaults, staking providers, exchanges, and bridges can all provide collateral management equally.
* **Unified Management.** View and manage sub-account access, recover locked up assets, and transfer rights for all of your funds, wherever they are stored to anyone within your trust with a key. If there was a RocketPool provider, a loved one recovering the wallet after a death could locate the staked ETH easily within the wallet.

### Ledger and Collateral Relationship

To do this we must separate the Collateral Provider asset storage from the ledgered account. A trusted relationship between the `ICollateralProvider` and the`Ledger` is managed by the `Notary`. The notary respects the root key holder and only enables the Collateral Provider to deposit funds for a proper key, or withdrawal from a key if the key-holder has allowed it.

A couple of trust assumptions are being made in this established relationship:

* The collateral provider trusts, verifies, and obeys the key allocation on the associated ledger.
* The root key holder trusts the collateral provider to faithfully service deposits and withdrawals.

By default, a Locksmith wallet comes with trusted relationships established with both the `EtherVault` and `TokenVault`. These contracts are simple asset storage vaults that service key deposits and withdrawals against the ledger.

### Asset Identity Protocol

Since the `Ledger` is not actually holding the assets, it identifies the assets with what is called an Asset Resource Name (ARN). It is defined as such:

```solidity
struct AssetType {
        // all EVM tokens originate from a contract, gas is zero.
        address contractAddress;

        // identifies the token standard for this asset
        // '0' is considered the native gas token,
        // or is otherwise considered 20, 721, 777, 1155, etc.
        uint256 tokenStandard;

        // for token types that have a non-fungible ID, this
        // field is used to denote that type of asset.
        uint256 id;
}
```

A short form ARN can be derived out of a simple hash scheme to guarantee uniqueness and identifiability. It is represented as a `bytes32`.

```solidity
/**
 * arn 
 *
 * Returns a UUID, or "asset resource name" for the asset type.
 * It is essentially the keccak256 of the asset type struct.
 * This is super convienent for mappings.
 *
 * @param asset the asset you want the arn/UUID for.
 * @return an opaque but unique identifer for this asset type.
 */
 function arn(AssetType memory asset) internal pure returns (bytes32) {
    return keccak256(abi.encode(asset.contractAddress, asset.tokenStandard, asset.id));
 }
```

The assets are tracked in a three-tiered Ledger object that keeps an account of key access rights for each asset actively stored for each collateral provider. The ledger balances are tracked, reconciled, and aggregated for each asset across each key, wallet, and Ledger during each entry. In this way, proof of solvency can be proven when the transaction is successfully added to a block.

<figure><img src="../../.gitbook/assets/Locksmith Architecture - Page 4.png" alt=""><figcaption></figcaption></figure>

The three-tiered structure provides the following immediate transaction guarantees:

* The Ledger verifies the collateral on the ledger is solvent for the provider, wallet owner, and key holder.
* The collateral provider has verified the ledger's account balance to satisfy it's ledger obligations.
* Moving funds within a wallet does not change collateral obligations for any provider.

## Storage



## Operations
