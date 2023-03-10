---
description: ERC1155 Contract with some flare
---

# 🏦 Key Vault

## Design Ethos

The KeyVault contract is a simple ERC1155 contract that keeps a robust internal ledger of NFT supply and enforces soulbound requirements for token transfers. The goal was to separate the token way from the business logic of its use cleanly to keep concerns encapsulated.

Another name for the contract could easily be `ERC1155SupplyTrackedSoulboundUpgradeable`, however `KeyVault` was chosen.

The open alpha will leverage [ERC1155Upgradeable](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/docs/modules/ROOT/pages/erc1155.adoc) as well as [UUPSUpgradeable](https://docs.openzeppelin.com/contracts/4.x/api/proxy#UUPSUpgradeable) implementations from OpenZeppelin. The overall architecture will enable the developer to progressively lock each contract once we are confident they are stable. The typical lifecycle on deploying, upgrading, and managing that applies.

Because this is an ERC1155 contract, each NFT or "Key" will have a unique ID modeled as a `uint256`.

## Storage

```
address private owner;
address public  locksmith;

mapping(address => mapping(uint256 => uint256)) private soulboundKeyAmounts;
mapping(address => EnumerableSet.UintSet) private addressKeys;
mapping(uint256 => EnumerableSet.AddressSet) private keyHolders;
mapping(uint256 => uint256) public keySupply;
```

### owner

This is the owner of the contract. The decision was made to do it directly and save the Ownable implementation byte-code. That decision was made before turning the optmizer on.

### locksmith

After deployment, the owner of the contract will "come back" with the address of the locksmith contract that it will respect contract calls from. In every other deployment scenario, dependent contract deployments come in at the initializer stage and never change.

### soulboundKeyAmounts

A mapping that keeps track of the number of each key type that an individual address has "soulbound." This is the number of ERC1155 tokens they must have in their wallet to make a valid transfer outside their wallet. An end user could have 3 of the same key type in their wallet, and be soulbound for 2 of them - in effect giving them the ability to only active move around one. This information is used during `_beforeTokenTransfer` __ to revert any transaction that would breach this consideration with a `SOUL_BREACH` reversion error. This value can be retrieved with `keyBalanceOf(address acount, uint256 keyId, bool soulbound)`.

### addressKeys

Mainly for introspection to drive management, keeps a quick set of all keys a given address holds. Enables the wallet to quickly determine what they are holding without processing events on the chain. Is the essense of `getKeys(address holder)`.

### keyHolders

Inversely, its also very valuable to see quickly who holds a given key type. This gives the wallet the ability to tell a root key holder any unique address that holds a key that they own. Is the essense of `getHolders(uint256 keyId)`.

### keySupply

For similar motivations we want to be able to keep track of the total supply for each key.

## Operations

The following operations are only available to the anointed locksmith, which is explicitly set by the contract owner by calling `setRespectedLocksmith(address _Locksmith)` sometime after deployment.

### Mint

This method simply checks to make sure the caller is the locksmith, increments the `keySupply` count, and `calls _mint(receiver, keyId, amount, data)` with no assumptions.

```solidity
/**
 * mint
 *
 * Only the locksmith can mint keys.
 *
 * @param receiver   the address to send the new key to
 * @param keyId      the ERC1155 NFT ID you want to mint
 * @param amount     the number of keys you want to mint to the receiver
 * @param data       the data field for the key
 */
 function mint(address receiver, uint256 keyId, uint256 amount, bytes calldata data) 
   external {
   require(locksmith == msg.sender, "NOT_LOCKSMITH");
   keySupply[keyId] += amount;
   _mint(receiver, keyId, amount, data);
 }
```

### Soulbind

This operation sets the soul-bound amount directly for a given key and holder. It checks to ensure the caller is the locksmith, sets the soulboundKeyAmounts, and emits an event.

<pre class="language-solidity"><code class="lang-solidity"><strong>/**
</strong> * soulbind
 *
 * 
 * It is safest to soulbind in the same transaction as the minting.
 * This function does not check if the keyholder holds the amount of
 * tokens. And this function is SETTING the soulbound amount. It is
 * not additive.
 *
 * @param keyHolder the current key-holder
 * @param keyId     the key id to bind to the keyHolder
 * @param amount    it could be multiple depending on the use case
 */
function soulbind(address keyHolder, uint256 keyId, uint256 amount) external {
    // respect only the locksmith in this call
    require(locksmith == msg.sender, "NOT_LOCKSMITH");

    // here ya go boss
    soulboundKeyAmounts[keyHolder][keyId] = amount;
    emit setSoulboundKeyAmount(msg.sender, keyHolder, keyId, amount);
}
</code></pre>

### Burn

This operation enables the locksmith to burn any key. There is one exception to this in that we've empowered each key holder to be able to burn an unwanted key as long as it wasn't soulbound. This prevents accidental soul breaches through scammers, but also enables ring key holders to safely relinquish their own possession.

Similarly to `mint`, it makes sure you are either the locksmith or the holder in question. Then, it manages the `keySupply` and  calls `_burn`.

```solidity
/**
 * burn 
 *
 * @param holder     the address of the key holder you want to burn from
 * @param keyId      the ERC1155 NFT ID you want to burn
 * @param burnAmount the number of said keys you want to burn from the holder's possession.
 */
 function burn(address holder, uint256 keyId, uint256 burnAmount) external {
     require(locksmith == msg.sender || holder == msg.sender, "NOT_LOCKSMITH_OR_HOLDER");
     keySupply[keyId] -= burnAmount;
     _burn(holder, keyId, burnAmount);
  }
```

### BeforeTokenTransfer

The KeyVault extends the typical `_beforeTokenTransfer` protocol on ERC1155. The first thing it ensures is the NFT transfer will not result in a state that violates the soulbound amount for the sending address and key type. It also updates the proper tallies in `addressKeys` and `keyHolders`.
