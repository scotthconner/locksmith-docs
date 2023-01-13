---
description: ERC1155 Contract with some flare
---

# 🏦 Key Vault

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
 function mint(address receiver, uint256 keyId, uint256 amount, bytes calldata data) external {
   require(locksmith == msg.sender, "NOT_LOCKSMITH");
   keySupply[keyId] += amount;
   _mint(receiver, keyId, amount, data);
 }
```

### Soulbind

### Burn

