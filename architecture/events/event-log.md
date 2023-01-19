---
description: Use Events to Feed Wallet Automation
---

# 📜 Event Log

## Design Ethos

In order to enable automation in a composable way, we needed to abstract out the logic gates into the concept of trigger an `Event`.

An event is a unique flag that has either not happened, or happened. The logic behind the event is separate from its observability, and can be found on a per trust-model basis in the `TrustEventLog`.

The `TrustEventLog` is a simple contract that behaves as a unified ledger of all registered and fired events for each trust model.&#x20;

Events must be registered by a trust `Dispatcher,` and fired by the same dispatcher. Dispatchers are trusted addresses identified by a root key holder to the `Notary` by calling `setTrustedLedgerRole` and using the `TrustEventLog` as the acting ledger. This is ensures that arbitrary addresses cannot register or fire events.

Benefits of this model include:

* **Action Composability:** If an event occurs, easily enable more than one thing discrete to happen which can implemented across different modules.
* **Event Composability:** An action taken can require multiple events to have fired and not need to be concerned about the conditions, or the dispatcher, of each event.
* **Oracles:** Events can take the form of oracles through off-chain human key holder attestations (KeyOracle), deterministic proof-of-life (AlarmClock), price oracles, etc.

## Storage

The contract holds a lot of information - including human readable information to facilitate UX workflows. This means that event registration, and to a degree, firing events, requires some additional gas to make this easy to see without requiring off-chain indexers.

Events need to have globally unique identifiers and are encoded as opaque `bytes32`.

```solidity
address public notary;

// eventHash => dispatcher
// holds the event hash to the dispatcher who has registered it
mapping(bytes32 => address) public eventDispatchers;

// holds an event description name
mapping(bytes32 => bytes32) public eventDescriptions;

// holds the event hashes for each trust
mapping(uint256 => bytes32[]) private trustEventRegistry;

// a nifty but expensive way of supporting dispatcher based queries
// trust => dispatcher => [events]
mapping(uint256 => mapping(address => bytes32[])) private trustDispatcherEvents;

// eventHash => hasFired?
// NOTE: This acts as an interface method for ITrustEventLog
mapping(bytes32 => bool) public firedEvents;olidi
```

### notary

The contract will check with the notary to ensure the registering dispatcher is approved by the key holder the dispatcher claims it is operating for.

### eventDispatchers

For each registered event, we want to know who the valid dispatcher is that can fire it.

### eventDescriptions

For each registered event, store a short `bytes32` encoded string that describes what it signifies to a human.

### trustEventRegistry

For each trust model, keep track of an append only array of registered event hashes.

### trustDispatcherEvents

A mapping of trust and dispatcher to events. This facilitates the view of knowing what events are coming from which dispatcher.

### firedEvents

This is the structure that enables other contracts to observe whether or not an event has occured based on its `bytes32` event hash.

## Operations

Outside of event introspection methods which are for UX, there are two methods for dispatchers to register and fire events. Events are managed on a per-trust model basis.

### registerTrustEvent

The dispatcher calls this method when they want to register a trust event with the log, ahead of the actual event occurring. This is useful to set up actions and permissions based on the events completion in the future.

```solidity
function registerTrustEvent(uint256 trustId, bytes32 eventHash, bytes32 description) external returns (bytes32) {
    // I'm sure their hash is super-special but let's sandbox
    // it into the dispatcher's address for our own sanity
    bytes32 finalHash = keccak256(abi.encode(msg.sender, eventHash));

    // we want to tell the notary about this to prevent
    // unauthorized event spam. this will revert if
    // the trust owner hasn't approved it.
    INotary(notary).notarizeEventRegistration(msg.sender, trustId, finalHash, 
         description);

     // make sure the hash isn't already registered
     require(address(0) == eventDispatchers[finalHash],
        "DUPLICATE_REGISTRATION");

     // invariant: make sure the event hasn't fired
     assert(!firedEvents[finalHash]);

     // register the event
     eventDispatchers[finalHash] = msg.sender;
     eventDescriptions[finalHash] = description;

     // the event is really bound to dispatchers, but
     // we want to keep track of a trust ID for introspection
     trustEventRegistry[trustId].push(finalHash);
     trustDispatcherEvents[trustId][msg.sender].push(finalHash);

     // emit the event
     emit trustEventRegistered(msg.sender, trustId, finalHash, description);

     return finalHash;
}
```

### logEvent

The dispatcher calls this method when it has dutifully determined the registered event has occurred. For this reason, it is important that the root key holder has trusted the `Dispatcher` to the `Notary`.

```solidity
function logTrustEvent(bytes32 eventHash) external {
    // make sure the message caller is the registration agent.
    // this will fail when an event isn't registered, or
    // the dispatcher isn't the one who registered the event.
    require(msg.sender == eventDispatchers[eventHash], 'INVALID_DISPATCH');

    // make sure this event hasn't been fired
    require(!firedEvents[eventHash], "DUPLICATE_EVENT");

    // the hash was previously registered by the message caller,
    // so set it to true.
    firedEvents[eventHash] = true;

    emit trustEventLogged(msg.sender, eventHash);
}
```
