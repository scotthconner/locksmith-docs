---
description: Key to Key Distribution Permissions
---

# 🛡 Trustee

## Design Ethos

The permissions and collateral providers enable safe key-level storage within a trust model. To collect their functionality, the `Trustee` contract acts as a `Scribe` to move funds between keys on the `Ledger` without moving the collateral itself at the provider.

Because it is a scribe, to enable the functionality a root key holder must trust the contract address at their `Notary`.

The Trustee contract has the following features:

* **Root Key Security.** Only root key holders are authorized to create policies.
* **Permissioned Distribution.** Configure which keys are allowed to move funds, and what keys they can move funds to.
* **Conditional Access.** Use configured `Events` to only enable the distribution permission under certain circumstances.

This enables a few high level use cases:

* **Asset Recovery.** Hold a second key that is otherwise inert. After a deadman's switch is triggered, the second key has access to all of the funds.
* **Air-locking Cold Storage.** Build a warm key that can only move funds from the treasure key to a hot wallet. Separate access to funds from the ability to move it across private keys.
* **Corporate Distribution.** The root key is a corporate treasury. Setting up tiered structures enables key holders to be decision makers without actually having access to the funds.

Additional `Scribe` and `Event` combinations can un-cover other use cases.

## Storage

```solidity
ILocksmith public locksmith;         // key validation
ITrustEventLog public trustEventLog; // event detection
ILedger public ledger;               // ledger manipulation

// A trust Policy enables a keyholder
// to distribute key trust funds.
struct Policy {
    bool enabled;               // is this policy enabled
    bytes32[] requiredEvents;   // the events needed to enable

    uint256 rootKeyId;          // the root key used to establish the policy
    uint256 sourceKeyId;        // where the funds can be moved from
    EnumerableSet.UintSet beneficiaries;    // where funds can be moved to
}

// maps trustee Keys, to the metadata.
// each key is limited to one policy in this implementation.
mapping(uint256 => Policy) private trustees;

// maps trusts to keys with policies on them
// this prevents UIs from having to spam the RPC
// with every key in the trust to get the set of policies.
 mapping(uint256 => EnumerableSet.UintSet) private trustPolicyKeys;
```

### locksmith

The reference to the locksmith the contract uses to check key permissions.

### trustEventLog

The reference to the trustEventLog the contract uses to. check for events.

### ledger

The reference to the ledger that this scribe will work on.

### struct Policy

#### enabled

Policies can be created before they are enabled. To create a policy that isn't enabled, add events to the required conditions.

#### requiredEvents

When a policy is created, it takes a list of `bytes32` that signify the unique event hash in the log that is required for policy execution.

#### rootKeyId

This is the id of the root key which owns the policy. Root keys can only create policies for keys within their trust model.

#### sourceKeyId

This is the id of the key where the policy is allowed to move funds out of. The key must valid and within the trust's model.

#### beneficiaries

This is a list of key ids that the policy is authorized to distribute funds to. Each of these keys must be within the root key's trust model and cannot contain the sourceKeyId.

### trustees

This is a global mapping of individual key IDs to their associated distribution policy. For simplicity and security, this implementation only allows one distribution policy per key. This encourages least privilege key design.

### trustPolicyKeys

Another storage that is part of the trade-off of ensuring that UIs can be feasibly built on top of the smart contract. This contains a mapping of trust IDs, to all of the available policies on keys within the specified trust. This prevents front-ends from having to collect the keys from the trust, and then poll each key to see if there is a policy, then load each policy.

## Operations

There are two introspection methods which will not be covered in detail, `getPolicy()`, and `getTrustPolicyKeys()`, which are essentially accessors to `trustees` and `trustPolicyKeys`.

### setPolicy

This method is called by root key holders to configure a distribution policy. The caller must hold `rootKeyId`. All other specified keys need to exist within the trust model specified by the `rootKeyId`. Adding required events is optional, and will default to an enabled policy.

The method validates the trust model structure of the root key, trustee key, source key, and all of the beneficiaries.&#x20;

```solidity
/** 
 * setPolicy 
 *
 * @param rootKeyId     the root key to use to set up the trustee role
 * @param trusteeKeyId  the key Id to anoint as trustee
 * @param sourceKeyId   the key id to use as the source of all fund movements
 * @param beneficiaries the keys the trustee can move funds to
 * @param events        the list of events that must occur before activating the role
 */
function setPolicy(
        uint256 rootKeyId,
        uint256 trusteeKeyId,
        uint256 sourceKeyId,
        uint256[] calldata beneficiaries,
        bytes32[] calldata events
) external {
        // ensure that the caller holds the key, and get the trust ID
        uint256 trustId = requireRootHolder(rootKeyId);

        // ensure that the beneficiary key ring isn't empty
        require(beneficiaries.length > 0, 'ZERO_BENEFICIARIES');

        // inspect the trustee key and ensure its on the trust's ring,
        (bool valid,,uint256 tid,,) = locksmith.inspectKey(trusteeKeyId);
        require(valid, "INVALID_TRUSTEE_KEY");
        require(tid == trustId, "TRUSTEE_OUTSIDE_TRUST");

        // inspect the source key and ensure its on the trust's ring
        (bool sValid,,uint256 sTid,,) = locksmith.inspectKey(sourceKeyId);
        require(sValid, "INVALID_SOURCE_KEY");
        require(sTid == trustId, "SOURCE_OUTSIDE_TRUST");

        // make sure a duplicate entry doesn't exist for this trustee
        require(trustees[trusteeKeyId].beneficiaries.length() == 0, 
                'KEY_POLICY_EXISTS');

        // we also want to validate the destination key ring.
        locksmith.validateKeyRing(trustId, beneficiaries, true);

        // at this point, the caller holds the root key, the trustee and source
        // are valid keys on the ring, and so are all of the beneficiaries.
        // save the configuration of beneficiaries and events
        Policy storage t = trustees[trusteeKeyId];
        t.rootKeyId = rootKeyId;
        t.sourceKeyId = sourceKeyId;

        // make sure that the sourceKeyId is not any of the beneficiaries either.
        for(uint256 x = 0; x < beneficiaries.length; x++) {
            require(sourceKeyId != beneficiaries[x], 'SOURCE_IS_DESTINATION');
            t.beneficiaries.add(beneficiaries[x]);
        }

        // keep track of the policy key at the trust level
        trustPolicyKeys[trustId].add(trusteeKeyId);

        // if the events requirement is empty, immediately activate
        t.requiredEvents = events;
        t.enabled = (0 == events.length);

        emit trusteePolicySet(msg.sender, rootKeyId, trusteeKeyId, sourceKeyId,
            beneficiaries, events);
}
```

### removePolicy

A root key holder can call this method if they would like to remove completely a disitribution policy from a key.

```solidity
/**
 * removePolicy
 *
 * @param rootKeyId    the key the caller is using, must be root
 * @param trusteeKeyId the key id of the trustee we want to remove 
 */
function removePolicy(uint256 rootKeyId, uint256 trusteeKeyId) external {
        // ensure that the caller holds the key, and get the trust ID
        uint256 trustId = requireRootHolder(rootKeyId);

        // make sure that the trustee entry exists in the first place.
        // we can know this, even on the zero trust, by ensuring
        // the beneficiary count is non-zero
        Policy storage t = trustees[trusteeKeyId];
        require(t.beneficiaries.length() > 0, 'MISSING_POLICY');

        // now that we know the trustee entry is valid, check
        // to ensure the root key being used is the one associated
        // with the entry
        require(t.rootKeyId == rootKeyId, 'INVALID_ROOT_KEY');

        // clean up the mapping
        uint256[] memory v = t.beneficiaries.values();
        for(uint256 x = 0; x < v.length; x++) {
            t.beneficiaries.remove(v[x]);
        }

        // at this point, we can delete the entry
        delete trustees[trusteeKeyId];

        // remove the policy key at the trust level
        trustPolicyKeys[trustId].remove(trusteeKeyId);

        emit trusteePolicyRemoved(msg.sender, rootKeyId, trusteeKeyId);
 }
```

### distribute

This method is called by the `trusteeKeyId` holder, and if successful will move funds from a source key to a sub-set of the designated beneficiaries of their amount. In this implementation there is no restriction on assets, amounts, or frequency.

```solidity
/**
 * distribute
 *
 * @param trusteeKeyId  the trustee key used to distribute funds
 * @param provider      the collateral provider you are moving funds for
 * @param arn           asset you are moving, one at a time only
 * @param beneficiaries the destination keys within the trust
 * @param amounts       the destination key amounts for the asset
 * @return a receipt of the remaining root key balance for that provider/arn.
 */
function distribute(
        uint256 trusteeKeyId,
        address provider,
        bytes32 arn,
        uint256[] calldata beneficiaries,
        uint256[] calldata amounts
) external returns (uint256) {
        // make sure the caller is holding the key they are operating
        require(IKeyVault(locksmith.getKeyVault()).keyBalanceOf(msg.sender, 
                trusteeKeyId, false) > 0, "KEY_NOT_HELD");

        // make sure the entry is valid
        Policy storage t = trustees[trusteeKeyId];
        require(t.beneficiaries.length() > 0, 'MISSING_POLICY');

        // make sure the trustee entry is activated, or can activate.
        // this code will panic if the events haven't haven't fired.
        ensureEventActivation(t);

        // validate the sub-set of keys provided
        for(uint256 x = 0; x < beneficiaries.length; x++) {
            require(t.beneficiaries.contains(beneficiaries[x]), 'INVALID_BENEFICIARY');
        }

        // do the distribution on the ledger, letting the notary take
        // care of validating the provider, and letting the ledger
        // assert proper balances. this call will also blow up if
        // this scribe has not been registered as a trusted one by the
        // root key holder with the notary.
        return ledger.distribute(provider, arn, t.sourceKeyId, beneficiaries, amounts);
    }
```

### ensureEventActivation

This method is called when a trustee tries to distribute. It will check and register event firing. Because these events fire asynchronously, event activation is checked view-only in `getPolicy()`, and written to record upon `distribute()`.

```solidity
/*
 * ensureEventActivation
 *
 * @param t the trustee entry in question. assumed from storage.
 */
 function ensureEventActivation(Policy storage t) internal {
        // easy exit if we've been here before
        if (t.enabled) { return; }

        // go through each required event and check the event log
        // to ensure each one of them have fired.
        for(uint256 x = 0; x < t.requiredEvents.length; x++) {
            require(trustEventLog.firedEvents(t.requiredEvents[x]), 'MISSING_EVENT');
        }

        // if we passed all the panics, let's ensure we wont
        // have to do this again.
        t.enabled = true;
}
```
