---
description: Permissioned Interface System
---

# 🪶 Notary

## Design Ethos

The Notary is a contract that holds permissions for each critical wallet action. Where-as with a Gnosis Safe that has a group of owners and optional modules, Locksmith uses a progressively granular permission scheme that allows for maximum flexibility while maintaining a single integration pattern.

The Notary is designed to approve actions that operate on a wallet's state, like moving collateral or firing events. To approve any action the notary must determine that the action meets acceptable parameters as outlined by a wallet root key holder.

As of the first version, the Notary includes the following permissions:

* **Collateral Providers:** Contract addresses that adhere to the `ICollateralProvider` interface and are trusted by the wallet's root key-holder to custody funds safely on-chain.
* **Scribes:** Contract Addresses that are trusted to move funds on the `Ledger` between keys in a wallet.
* **Dispatchers:** Contract addresses that are trusted to register and fire events within the wallet to the `TrustEventLog`.

These modules are entrusted by the root key holder to provide and extend the wallet's capacity to store, move, and automate funds. The same contract can hold multiple permissions but are acquired and set separately for composability.

<figure><img src="../../.gitbook/assets/Locksmith Architecture - Page 2 (1).png" alt=""><figcaption><p>Smart Contract interaction and trust model</p></figcaption></figure>

## Storage

The 'ledger' component in these mappings is the message sender that is querying for notarization. In the current implementation this is the `Ledger` and the `TrustEventLog`.

```solidity
// the notary only respects one locksmith
ILocksmith public locksmith;

// Key-holders enable collateral to be withdrawn from
// the ledger.
// ledgerAddress / keyId / providerAddress / arn => approvedAmount
mapping(address =>
        mapping(uint256 =>
        mapping(address =>
        mapping(bytes32 => uint256)))) public withdrawalAllowances;

// trusted ledger actors
// ledger / trust / role => [actors]
mapping(address => mapping(uint256 => mapping(uint8 => EnumerableSet.AddressSet))) private actorRegistry;

// actor aliases
// ledger / trust / role / actor => alias
mapping(address => mapping(uint256 => mapping(uint8 => mapping(address => bytes32)))) public actorAliases;

// The notary cares about a few different role types
// that are attached to the ledger/trust pair. This
// enum differentiates the storage while still making
// the entire relationship state directly queryable outside
// the contract.
uint8 constant public COLLATERAL_PROVIDER = 0;
uint8 constant public SCRIBE = 1;
uint8 constant public EVENT_DISPATCHER = 2;
```

### locksmith

The reference to the Locksmith interface. Instead of using a contract registry, each Locksmith contract takes a direct deployment dependency on a contract it requires to operate.

### withdrawalAllowances

A key holder will go to the collateral provider to request a withdrawal. For collateral providers to offer a valid withdrawal, it must be fully registered and balanced on the ledger and pass notarization. Since the collateral provider does not explicitly hold the key, a critical part of withdrawal notarization is to ensure the key-holder who is withdrawing has attested to approve that amount, of that asset, from that ledger, for that collateral provider. At later layers this is composed into a single transaction.

### actorRegistry

A mapping of contract address of modules that are approved for each trust model, for each role. The roles are defined as simple unsigned integers and are documented below. These are checked during operations to ensure the actors are considered trusted.

### actorAliases

Human readable alises for each contract address and role. Because no off-chain infrastructure should be needed to create a feasible UX, we are paying the gas and the storage to provide some sane expectations for clients.

### COLLATERAL\_PROVIDER

This role describes an actor that is trusted by the root key holder to honor asset deposit and withdrawal requests from key holders, respecting the key-holder balances on the wallet's central `Ledger`. This enables wallet owners to compose their asset storage and investments with different features and providers directly into their wallet. On-chain Vaults, Staking providers, or   exchanges can expose on-chain ICollateralProvider interfaces and honor key-rights. This in effect brings all deployed assets back into a single virtual wallet.

### SCRIBE

This role describes an actor that is trusted by the root key holder to move funds between keys. A scribe has the ability to move any funds on the ledger between any set of valid keys within the wallet's trust model. It is up to the root key holder to sufficiently trust the means by which the scribe will move funds. Contracts that restrict access to move funds under different scenarios can be composed.

### DISPATCHER

This role describes an actor that is trusted by the root key holder to register and fire events. Events are immutable boolean triggers that can be consumed by collateral providers, scribes, or other applications to compose additional logic and gates into features.

## Operations

There are two typese of operations. There are operations that can only be successfully executed by a root key holder, and there are operations that are assumed to be executed by the associated role's ledger.&#x20;

### setTrustedLedgerRole

This method is called by a root key holder to specify to trust or untrust a specific contract address for a given role. It ensures that the root key holder is valid, and then will double check the users action before storing the root key's configuration.

**This method can only be successfully called by a root key holder.**

```solidity
/**
 * setTrustedLedgerRole
 *
 * @param rootKeyId  the root key the caller is trying to use to enable an actor
 * @param role       the role the actor will play (provider or scribe)
 * @param ledger     the contract of the ledger used by the actor
 * @param actor      the contract of the ledger actor
 * @param trustLevel the flag to set the trusted status of this actor
 * @param actorAlias the alias of the actor, set if the trustLevel is true
 */
 function setTrustedLedgerRole(uint256 rootKeyId, uint8 role, address ledger, address actor,
  bool trustLevel, bytes32 actorAlias) external {
  
  // make sure that the caller is holding the key they are trying to use
  require(IKeyVault(locksmith.getKeyVault()).keyBalanceOf(msg.sender, rootKeyId, false) > 0, "KEY_NOT_HELD");

  // make sure the key is a valid root key
  require(locksmith.isRootKey(rootKeyId), "KEY_NOT_ROOT");

  // the caller is holding it a valid root key, this lookup is safe
  (,,uint256 trustId,,) = locksmith.inspectKey(rootKeyId);
  
  if (trustLevel) {
      // make sure they are not already a provider on the trust
      require(!actorRegistry[ledger][trustId][role].contains(actor), 'REDUNDANT_PROVISION');

      // register them with the trust if not already done so
      actorRegistry[ledger][trustId][role].add(actor);

      // set the alias
      actorAliases[ledger][trustId][role][actor] = actorAlias;
   } else {
      // we are trying to revoke status, so make sure they are one
      require(actorRegistry[ledger][trustId][role].contains(actor), 'NOT_CURRENT_ACTOR');

      // remove them from the notary. At this point in time
      // there could still be collateral in the trust from this provider.
      // the provider isn't trusted at this moment to facilitate deposits
      // or withdrawals. Adding them back would re-enable their trusted
      // status. This is useful if a collateral provider is somehow compromised.
      actorRegistry[ledger][trustId][role].remove(actor);
   }
}
```

### setWithdrawalAllowance

All key holders must attest to the notary that collaterael providers are cleared to register withdrawals to the ledger on their behalf. This is because the collateral provider is not required to hold the key when facilitating a withdrawal. But in its place resides the key holder attestation to the notary.

A key holder calls this method, usually right before a withdrawal, to enable the collateral provider to successfully register on the ledger.

**This method can only be successfully called by a  key holder.**

```solidity
/**
 * setWithdrawalAllowance
 * 
 * @param ledger   address of the ledger to enable withdrawals from
 * @param provider collateral provider address to approve
 * @param keyId    key ID to approve withdraws for
 * @param arn      asset you want to approve withdrawal for
 * @param amount   amount of asset to approve
 */
 function setWithdrawalAllowance(
    address ledger, 
    address provider, 
    uint256 keyId, 
    bytes32 arn, 
    uint256 amount) 
 external {
     require(IKeyVault(locksmith.getKeyVault()).keyBalanceOf(msg.sender, keyId, false) > 0, 'KEY_NOT_HELD');
     withdrawalAllowances[ledger][keyId][provider][arn] = amount;
 }
```

### notarizeDeposit

The wallet's central ledger will call this for notarization when a collateral provider attempt to deposit to a wallet's ledger. This will fail if the key is invalid, or if the collateral provider is not trusted by the wallet's root key holder.&#x20;

Below is also an internal method, `requireTrustedActor()`, that is used for multiple notarization operations.

```solidity
/**
 * notarizeDeposit
 *
 * @param provider the provider that is trying to deposit
 * @param keyId    key to deposit the funds to
 * @param arn      asset resource hash of the withdrawn asset
 * @param amount   the amount of that asset withdrawn.
 * @return the valid trust Id for the key
 */
 function notarizeDeposit(
     address provider,
     uint256 keyId,
     bytes32 arn,
     uint256 amount
 ) external returns (uint256) {
     uint256 trustId = requireTrustedActor(keyId, provider, COLLATERAL_PROVIDER);
     return trustId;
 }
 
 /**
  * requireTrustedActor
  *
 * Given a key and an actor, panic if the key isn't real,
 * it's not root when it needs to be, or the trust
 * doesn't trust the actor against a given ledger.
 *
 * This method assumes the message sender is the ledger.
 *
 * @param keyId the key Id for the operation
 * @param actor the actor address to check
 * @param role  the role you need the actor to be trusted to play
 * @return the valid trust ID associated with the key
 */
 function requireTrustedActor(
    uint256 keyId,
    address actor,
    uint8 role
 ) internal view returns (uint256) {
    // make sure the key is valid. you can't always ensure
    // that the actor is checking this
    (bool valid,,uint256 trustId,,) = locksmith.inspectKey(keyId);
    require(valid, "INVALID_KEY");

    // make sure the actor is trusted
    // we assume the message sender is the ledger
    require(actorRegistry[msg.sender][trustId][role].contains(actor), 
        'UNTRUSTED_ACTOR');

    return trustId;
}
```

### notarizeWithdrawal

The wallet's central ledger will call this for notarization when a collateral provider attempt to withdrawal to a wallet's ledger. This will fail if the key is invalid, the withdrawal allowance for that key holder, provider, and asset is insufficient or if the collateral provider is not trusted by the wallet's root key holder.&#x20;

The notary does not check the ledger for sufficient balance. The ledger will fail to create a valid entry on its own. The notaries role is the ensure everything else is proper.

<pre class="language-solidity"><code class="lang-solidity"><strong> /**
</strong><strong>  * notarizeWithdrawal
</strong><strong>  *
</strong>  * @param provider the provider that is trying to withdrawal
  * @param keyId    key to withdrawal the funds from
  * @param arn      asset resource hash of the withdrawn asset
  * @param amount   the amount of that asset withdrawn.
  * @return the valid trust ID for the key
  */
  function notarizeWithdrawal(
      address provider,
      uint256 keyId,
      bytes32 arn,
      uint256 amount
  ) external returns (uint256) {
      // make sure the key is valid and the provider is trusted
      uint256 trustId = requireTrustedActor(keyId, provider, COLLATERAL_PROVIDER);

      // make sure the withdrawal amount is approved by the keyholder
      // and then reduce the amount
      require(withdrawalAllowances[msg.sender][keyId][provider][arn] >= amount,
          'UNAPPROVED_AMOUNT');
      withdrawalAllowances[msg.sender][keyId][provider][arn] -= amount;
      return trustId;
 }
</code></pre>

### notarizeDistribution

This method is called by the ledger when a scribe is attempting to re-distribute funds amongst keys. The notarization will fail if the involved scribe or collateral provider isn't trusted, or if the keys do not all belong to the same ring.

```solidity
 /**
  * notarizeDistribution
  *
  * @param scribe      the address of the scribe that is supposedly trusted
  * @param provider    the address of the provider whose funds are to be moved
  * @param arn         the arn of the asset being moved
  * @param sourceKeyId the root key that the funds are moving from
  * @param keys        array of keys to move the funds to
  * @param amounts     array of amounts corresponding for each destination keys
  * @return the trustID for the rootKey
  */
  function notarizeDistribution(address scribe, address provider, bytes32 arn,
      uint256 sourceKeyId, uint256[] calldata keys, uint256[] calldata amounts) external returns (uint256) {

      // the scribe needs to be trusted
      uint256 trustId = requireTrustedActor(sourceKeyId, scribe, SCRIBE);

      // we also want to make sure the provider is trusted
      require(actorRegistry[msg.sender][trustId][COLLATERAL_PROVIDER].contains(provider),
          'UNTRUSTED_PROVIDER');

      // check to ensure the array sizes are 1:1
      require(keys.length == amounts.length, "KEY_AMOUNT_SIZE_MISMATCH");

      // this method will fully panic if its not valid.
      locksmith.validateKeyRing(trustId, keys, true);

      return trustId;
  }
```

### notarizeEventRegistration

This method is called by the `TrustEventLog` to ensure that a dispatcher whom is registering an event is trusted by the specified wallet owner. This prevents spam, attacks, or otherwise random events from popping up in the event stream. In this implementation, two default dispachers are included (`KeyOracle` and `AlarmClock`)./\*\*

```solidity
/**
 * notarizeEventRegistration
 *
 * @param dispatcher  registration address origin
 * @param trustId     the trust ID for the event
 * @param eventHash   the unique event identifier
 * @param description the description of the event
 */
 function notarizeEventRegistration(address dispatcher, uint256 trustId, bytes32 eventHash, bytes32 description) external {
     // we want to make sure the dispatcher is trusted
     // note: here we are using the event log as the "ledger".
     require(actorRegistry[msg.sender][trustId][EVENT_DISPATCHER].contains(dispatcher),
         'UNTRUSTED_DISPATCHER');
 }
```
