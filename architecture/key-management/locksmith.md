---
description: Providing the Trust Model
---

# 🔓 Locksmith

## Design Ethos

The Locksmith contract is  dependent on a functioning and compatible KeyVault. Once deployed, the KeyVault is then set by its deployer to "respect" the Locksmith's supply management wishes. In this way, the "Locksmith" becomes the minter of any and all keys.

With full single-threaded access to key management, the Locksmith exposes a singular "Trust" model for each key. New trusts are created when a user requests one and is minted a root key.&#x20;

It operates simiarly as a UUPSUpgradeable and can be locked after development is complete. It stores the information about the trust, the keys that belong to it, and some metadata about the keys.

Root key holders can interact with the Locksmith by calling the operations. For each operation except `createTrustAndRootKey` the caller must possess a valid Locksmith root key.

## Storage

The main contract storage is the registry of the Trust. Trusts have their own ID structure, and contain all of the metadata about the trust and associated keys.

<pre class="language-solidity"><code class="lang-solidity">
address public keyVault;

struct Trust {
        uint256 id;
<strong>        bytes32 name;
</strong>        uint256 rootKeyId;
        EnumerableSet.UintSet keys;
        mapping(uint256 => bytes32) keyNames;
}

mapping(uint256 => Trust) private trustRegistry;
uint256 private trustCount;

mapping(uint256 => uint256) private keyTrustAssociations;
uint256 private keyCount;
</code></pre>

### keyVault

The Locksmith contract is deployed with an address pointing to a `KeyVault` on initialization. This is the only KeyVault the Locksmith will use to manage his key requests.

### struct Trust

#### id

The trust's global unique identifier.&#x20;

#### name

Stores as a byte-encoded `bytes32`, a short descriptive name that does not require global uniqueness.

#### rootKeyId

Each trust has a single root key Id that can never change. This field denotes the key ID that represents the root key for that trust model instance.

#### keys

The array of key IDs that belong to the trust. Will always contain the root key ID.

#### keyNames

A mapping of key IDs to their human readable names encoded as a `bytes32`.

### trustRegistry

A mapping from a trust's ID to the Trust structure.

### trustCount

The number of trusts the locksmith has in it's registry.

### keyTrustAssociations

Each key belongs to a trust, but it isn't easy to find out which one from an array of Trust structures. An additional mapping keeps this unique piece of key metadata to access the trust it belongs to.

### keyCount

An internal state variable that determines immediately what is considered a "valid key" by keeping track of the number of unique keys in existence.

## Operations

There are introspection methods, that will not get extensive review here. They are `getKeyVault()`, `getTrustInfo(uint256 trustId)`,  `getKeys(uint256 trustId)`, `inspectKey(uint256 keyId)`. These methods supply the key vault's contract address, the Trust metadata stored in the struct,  the list of keys and their data as described in the previous section.

### createTrustAndRootKey

This is the method any message sender can call to immediately create a trust and send the root key to the designated recipient. It only takes a trust name, and a destination address. The `mint` function is an internal one that adds the given key id (in this case, the root key), optionally soulbinds the receiver before minting, and then minting the new key.

```solidity
/**
  * createTrustAndRootKey
  *
  * @param trustName A string defining the name of the trust, like 'My Family Trust'
  * @param recipient The address to receive the root key for this trust.
  * @return the trust ID that was created
  * @return the root key ID that was created
  */
  function createTrustAndRootKey(bytes32 trustName, address recipient) external returns (uint256, uint256) {
        // do all state changes up front to prevent
        // re-entrancy issues on mint
        Trust storage t = trustRegistry[trustCount];
        t.id = trustCount++;
        t.rootKeyId = keyCount++;
        t.name = trustName;

        // add the root key to the pool mapping, and associate
        // the key with the trust
        t.keys.add(t.rootKeyId);
        t.keyNames[t.rootKeyId] = 'root';
        keyTrustAssociations[t.rootKeyId] = t.id;

        // re-entrant
        // mint the root key, give it to the sender.
        mintKey(t, t.rootKeyId, recipient, false);

        // the trust was successfully created
        emit trustCreated(msg.sender, t.id, t.name, recipient);
        return (t.id, t.rootKeyId);
  }
```

### isRootKey

Definitely answers if a key is a considered root for a valid trust by making sure the key ID itself is valid, the root key matches the one determined in the trust association index, and that the key is on the trust's ring list.

```solidity
/**
 * isRootKey
 *
 * @param keyId the key id in question
 * @return true if the key Id is the root key of it's associated trust
 */
 function isRootKey(uint256 keyId) public view returns(bool) {
        // key is valid
        return (keyId < keyCount) &&
        // the root key for the trust is the key in question
        (keyId == trustRegistry[keyTrustAssociations[keyId]].rootKeyId) &&
        // the key is on the ring list
        (trustRegistry[keyTrustAssociations[keyId]].keys.contains(keyId));
 }
```

### createKey

The holder of a root key can use this method to generate brand new keys and add them to the root key's associated trust key ring. By default, the generated keys will have no attached permissions or special features. The internal method `getTrustFromRootKey` checks to ensure the message sender holds the root key used for the operation, and that the key is in fact root. If either are not true the transaction will revert. It will otherwise provide the trustId in return.

**This method can only be successfully called by a root key holder.**

```solidity
/**
 * createKey
 *
 * @param rootKeyId key the sender is attempting to use to create new keys.
 * @param keyName   an alias that you want to give the key
 * @param receiver  address you want to receive an NFT key for the trust.
 * @param bind      true if you want to bind the key to the receiver
 * @return the ID of the key that was created
 */
 function createKey(uint256 rootKeyId, bytes32 keyName, address receiver, bool bind) 
        external returns (uint256) {
        
        // Make sure the message caller is legit
        Trust storage t = trustRegistry[getTrustFromRootKey(rootKeyId)];
        
        // increment the number of unique keys in the system
        uint256 newKeyId = keyCount++;

        // push the latest key ID into the trust, and
        // keep track of the association at O(1), along
        t.keys.add(newKeyId);
        t.keyNames[newKeyId] = keyName;
        keyTrustAssociations[newKeyId] = t.id;

        // mint the key into the target wallet.
        // THIS IS RE-ENTRANT!!!!
        mintKey(t, newKeyId, receiver, bind);
        return newKeyId;
 }
```

### copyKey

The root key holder can call this method if they have an existing key type within their trust they want to copy. This method will work even if the key is "extinct" in that all previous copies were burnt or lost. Having multiple instances of the same key can be very useful and flexible.

**This method can only be successfully called by a root key holder.**

```solidity
 /**
  * copyKey
  *
  * @param rootKeyId root key to be used for this operation
  * @param keyId     key ID the message sender wishes to copy
  * @param receiver  addresses of the receivers for the copied key.
  * @param bind      true if you want to bind the key to the receiver
  */
  function copyKey(uint256 rootKeyId, uint256 keyId, address receiver, bool bind) 
    external {
        Trust storage t = trustRegistry[getTrustFromRootKey(rootKeyId)];

        // we can only copy a key that already exists within the
        // trust associated with the valid root key
        require(t.keys.contains(keyId), 'TRUST_KEY_NOT_FOUND');

        // the root key is valid, the message sender holds it,
        // and the key requested to be copied has already been
        // minted into that trust at least once.
        mintKey(t, keyId, receiver, bind);
    }
```

### soulbindKey

This method can be called by a root key holder to make a key soulbound to a specific walet. When soulbinding a key, it is not required that the target address yet holds that specific key. The amount set ensures that when sending a key of a specific type, that they hold at least the amount that is bound to them after the transaction.

**This method can only be successfully called by a root key holder.**

<pre class="language-solidity"><code class="lang-solidity">/**
<strong>  * soulbindKey
</strong>  *
  *
  * @param rootKeyId the operator's root key
  * @param keyHolder the address to bind the key to
  * @param keyId     the keyId they want to bind
  * @param amount    the amount of keys to bind to the holder
  */
  function soulbindKey(
    uint256 rootKeyId, 
    address keyHolder, 
    uint256 keyId, 
    uint256 amount) 
  external {
        Trust storage t = trustRegistry[getTrustFromRootKey(rootKeyId)];

        // is keyId associated with the root key's trust?
        require(t.keys.contains(keyId), 'TRUST_KEY_NOT_FOUND');

        // the root key holder has permission, so bind it
        IKeyVault(keyVault).soulbind(keyHolder, keyId, amount);
    }
</code></pre>

### burnKey

The root key holder can call this method if they want to revoke a key from a current holder. This function alone makes the Locksmith NFTs utilitarian on behalf of the root key holder and not speculative in nature.

**This method can only be successfully called by a root key holder.**

```solidity
/**
 * burnKey
 * 
 * @param rootKeyId root key for the associated trust
 * @param keyId     id of the key you want to burn
 * @param holder    address of the holder you want to burn from
 * @param amount    the number of keys you want to burn
 */
 function burnKey(
   uint256 rootKeyId, 
   uint256 keyId, 
   address holder, 
   uint256 amount) 
 external {
        Trust storage t = trustRegistry[getTrustFromRootKey(rootKeyId)];

        // is keyId associated with the root key's trust?
        require(t.keys.contains(keyId), 'TRUST_KEY_NOT_FOUND');

        // burn them, and count the burn for logging.
        // this call is re-entrant, but we do all of
        // the state mutation afterwards.
        IKeyVault(keyVault).burn(holder, keyId, amount);

        emit keyBurned(msg.sender, t.id, keyId, holder, amount);
 }
```

### validateKeyRing

A method that determines if a set of keys all belong to the same trust. Optionally enables you to revert if a root key is in the list.

```solidity
/**
  * validateKeyRing
  *
  * @param trustId   the trust ID you want to validate against
  * @param keys      the supposed keys that belong to the trust's key ring
  * @param allowRoot true if having the trust's root key on the ring is acceptable
  * @return true if valid, or will otherwise revert with a reason.
  */
  function validateKeyRing(
      uint256 trustId, 
      uint256[] calldata keys, 
      bool allowRoot) 
  external view returns (bool) {
        // make sure the trust is valid
        require(trustId < trustCount, 'INVALID_TRUST');

        // this is safe since the trust is valid
        Trust storage t = trustRegistry[trustId];

        // invariant: make sure the root key was minted once
        assert(t.keys.contains(t.rootKeyId));

        for(uint256 x = 0; x < keys.length; x++) {
            // make sure the key is a valid locksmith key. This
            // prevents funds on the ledger being allocated to future-minted
            // keys within different trusts.
            require(keys[x] < keyCount, 'INVALID_KEY_ON_RING');

            // in some cases a root key can't be allowed on a key ring
            require(allowRoot || (keys[x] != t.rootKeyId), 'ROOT_ON_RING');

            // make sure this valid key belongs to the same trust. this
            // call is only safe after checking that the key is valid.
            require(t.keys.contains(keys[x]), "NON_TRUST_KEY");
        }

        // at this point, the trust is valid, the root has been minted
        // at least once, every key in the array is valid, meets the
        // allowed root criteria, and has been validated to belong
        // to the trustId
        return true;
    }
```
