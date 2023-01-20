---
description: Simple Key Attestation
---

# 👁🗨 Key Oracle

## Design Ethos

The KeyOracle contract acts as Dispatcher and is designed to model on-chain attestations by a pre-designated key-holder.

This enables a programmatic outsourcing of determining whether or not an event registered by the root key holder has happened. Any use case for firing events securely could be built on top of a Key Oracle.

The contract works by taking an event description from a root key holder, along with an associated key id that is able to fire the event at their will. The key id must belong to the root key holder. Thus, the event registration comes with the assumption that the root key holder trusts the assigned key to faithfully fire the event.

When the key holder attests to the event, the `KeyOracle` contract fires the event on the `TrustEventLog`, enabling its state to be reflected to `Scribes` and other programs.

<figure><img src="../../.gitbook/assets/Locksmith Architecture - Page 4 (3) (1).png" alt=""><figcaption></figcaption></figure>

## Storage

The KeyOracle has very little state or logic other than to keep track of key level dispatcher permissions per event.

```solidity
ILocksmith public locksmith;
ITrustEventLog public trustEventLog;

// keyId => [eventHashes]
mapping(uint256 => EnumerableSet.Bytes32Set) private oracleKeyEvents;

// eventHash => keyId
mapping(bytes32 => uint256) public eventKeys;
```

### locksmith

The locksmith reference the contract relies on to check for key possession.

### trustEventLog

The event log reference where this contract will register and fire events to. For the `KeyOracle` to work properly, each trust owner must approve it as a dispatcher at the `Notary`.

### oracleKeyEvents

This is a mapping for each key, to a set of events that the key can be responsible for.

### eventKeys

For each event, which key is responsible for firing it.

## Operations

The operations on this contract are simple. Register an event that can only be fired by a given key, and fire that even when the key holder attests.

### createKeyOracle

Creates the key oracle. This method can only be called by a root key holder of a trust that has attested to the `Notary` that the `KeyOracle` contract is a valid dispatcher. The key assigned to the event must also be within the root key holder's trust model. The root key holder can specify a short description encoded as a `bytes32` to describe the event significance.

<pre class="language-solidity"><code class="lang-solidity"><strong>/**
</strong><strong> * createKeyOracle
</strong><strong> *
</strong><strong> * @param rootKeyId   the root key to use to create the event.
</strong> * @param keyId       the trust to associate the event with
 * @param description a small description of the event
 */ 
<strong>function createKeyOracle(uint256 rootKeyId, uint256 keyId, bytes32 description) external {
</strong>    // ensure the caller is holding the rootKey
    require(IKeyVault(locksmith.getKeyVault()).keyBalanceOf(msg.sender, rootKeyId, 
        false) > 0, 'KEY_NOT_HELD');
    require(locksmith.isRootKey(rootKeyId), "KEY_NOT_ROOT");

    // inspect the keys used, make sure each of them are valid
    // and that the oracle key belongs to the root key's trust
    (bool rootValid,, uint256 rootTrustId,,) = locksmith.inspectKey(rootKeyId);
    (bool keyValid,, uint256 keyTrustId,,) = locksmith.inspectKey(keyId);
    require(rootValid &#x26;&#x26; keyValid &#x26;&#x26; rootTrustId == keyTrustId, 'INVALID_ORACLE_KEY');

    // register it in the event log first. If the event hash is a duplicate,
    // it will fail here and the entire transaction will revert.
    bytes32 finalHash = trustEventLog.registerTrustEvent(rootTrustId,
        keccak256(abi.encode(rootKeyId, keyId, description)), description);

    // if we get this far, we know its not a duplicate. Store it
    // here for introspection.
    oracleKeyEvents[keyId].add(finalHash);
    eventKeys[finalHash] = keyId;

    // emit the oracle creation event
    emit keyOracleRegistered(msg.sender, rootTrustId, rootKeyId, keyId, finalHash);
}
</code></pre>

### fireKeyOracleEvent

Once a KeyOracle is susccessfully created, the specified key holder can come back and formally fire the event.

```solidity
function fireKeyOracleEvent(uint256 keyId, bytes32 eventHash) external {
     // ensure the caller holders the key Id
     require(IKeyVault(locksmith.getKeyVault()).keyBalanceOf(msg.sender, keyId, false) > 0, 'KEY_NOT_HELD');

     // make sure the event hash is registered to the given key
     require(oracleKeyEvents[keyId].contains(eventHash), 'MISSING_KEY_EVENT');

     // fire the event to the trust event log
     trustEventLog.logTrustEvent(eventHash);
}
```



