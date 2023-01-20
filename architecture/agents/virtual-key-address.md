---
description: Building a Virtualized Wallet Identity
---

# 📬 Virtual Key Address

## Design Ethos

The `VirtualKeyAddress` is a per-key-instance contract that gives each individual key an on-line contract address to send and receive funds from. Operators of the virtual account must hold a specific key.

Operators have virtualized access to use all of the funds in the trust's collateral providers, and to external parties will see the `msg.sender` be the virtual address of the key's "inbox" and not the EOA account holding the key and signing the transaction.

This contract can use any funds from any asset the key as rights to in single or multi-call transactions.

The contract achieves and maintains the authorization and security by requiring a **soulbound** **key** to be attached to the contract so the contract can act as a definitive key holder with collateral providers and the `Notary`. For non-receive functions, both the caller and the contract must hold the same identity key.

The VirtualKeyAddress has further extendability to be able to:

* Block senders or receivers based on address.
* Block dustings or unwanted NFTs, tokens.
* Provide multi-key multi-sig

This functionality is a trio of three contracts.

* VirtualKeyAdress: The per-instance contract that enables each key to have a unique address and provides key security.&#x20;
* PostOffice: This is a registry that maintains a mapping of key IDs to contract addresses. This enables discoverability and management for UX. It also helps the root key holder from creating duplicate key addresses by accident.
* KeyAddressFactory: This agent takes possession of a root key long enough to create a copy o the required key, create the inbox, soulbind the new key to the inbox address, and register it with the post office. The root key is given back to the message sender at the end of the transaction.

## VirtualKeyAddress

### Storage

The virtualized wallet keeps an internal model of the transaction history on-chain. It is modeled as such.

<pre class="language-solidity"><code class="lang-solidity">enum TxType { INVALID, SEND, RECEIVE, ABI }
<strong>
</strong><strong>struct Transaction {
</strong>    TxType transactionType; // what type of transaction is it?
    uint256 blockTime;      // when did this transaction happen?
    address operator;       // who is exercising the address?
    address target;         // who is the target of the action?
    address provider;       // what provider is involved?
    bytes32 arn;            // what asset is involved?
    uint256 amount;         // how much of that asset was involved?
}
</code></pre>

The contract storage is as follows:

```solidity
bytes32 public ethArn;

// owner and identity management
uint256 public ownerKeyId;     // the owner of this contract
uint256 public keyId;          // the virtual address "identity"
bool    public keyInitialized; // separates operation from key ID 0 by default

// Collateral provider configuration
IEtherCollateralProvider public defaultEthDepositProvider;
bool    public depositHatch; // this is used to prevent withdrawals from trigger deposits

// Platform references required
address public locksmith;

// chain storage for transaction history
Transaction[] public transactions;
uint256 public transactionCount;
```

#### ethArn

Computed at initialization, stores the ethArn for easy use.

#### ownerKeyId

This is the key ID from the locksmith that legitimately owns the contract and is the only one who can upgrade it.

#### keyId

This is the key ID that the contract assumes the identity of. It will require that for operations that use vaulted collateral to hold this key for successful operation. The contract aslo assumes that a key with this ID is soulbound to the contract given it the authority to act as a key holder on behalf of the caller.

#### keyInitialized

Determines if the key is properly initialized. Protects the keyId from looking like `0` when not initialized.

#### defaultEthDepositProvider

Wallet requires a default ether collateral provider to automatically service deposits. ERC20s do not provide callbacks and thus do not have a default provider.

#### depositHatch

This is a state variable that is used to model re-entrancy that comes from a collateral provider delivering the contract funds during a funding preparation step. Without this hatch, retrieving ether or 721s from vaults would look like incoming deposits from other senders and not preparation for an internal contract operation.

#### locksmith

The reference to the contract used to determine key possession.

#### transactions

The on-chain storage for this wallet's transaction history.

#### transactionCount

The number of transactions recorded in the history.

### Operations

## PostOffice

### Storage



### Operations

## KeyAddressFactory



### Storage



### Operations
