---
description: Trust-less Time Delays
---

# ⏰ Alarm Clock

## Design Ethos

One of the first use cases  that comes with storing assets on-chain is locking them up for a specific amount of time. Combined with events, this enables a lot of use cases that can move funds without the immediate interaction of the trust owner.

This implementation of a Dispatcher models an Alarm Clock that can be preemptively "snoozed." The root key registers an event, that is eligible to fire by anyone at a specified time. Optionally, it can be configured to be repeatedly "snoozed" and extending the event eligibility.

This enables a few immediate use cases:

* Proof-of-life inheritance, proof-of-presence recovery
* Coming of age inheritance
* One-time Vesting schedules

## Storage

```solidity
ILocksmith public locksmith;
ITrustEventLog public trustEventLog;

struct Alarm {
    bytes32 eventHash;      // the event to fire upon challenge
    uint256 alarmTime;      // when the event can be fired
    uint256 snoozeInterval; // alarm time extension, or '0' if disabled
    uint256 snoozeKeyId;    // the key that can extend the alarm time, if interval isn't zero
}

// eventHash => Alarm
mapping(bytes32 => Alarm) public alarms;
```

### locksmith

The reference the contract uses to verify key possession.

### trustEventLog

The reference the contract use to register and fire events.

### struct Alarm

This structure models the alarm that is created by the key holder.

#### eventHash

The unique identifier for the event.

#### alarmTime

The epoch timestamp that signifies when the alarm is elligible to be challenged and fired. This can be changed if the alarm is snooze-able.

#### snoozeInterval

If non-zero, specifies how much time will be added to the alarm's deadline when successfully snoozed.

#### snoozeKeyId

The key Id that is enabled to snooze the alarm. Must belong to the trust model of the root key holder establishing the alarm.

#### alarms

A registery of alarms attached to their specified event hash.

## Operations

### createAlarm

A root key holder can call this method to create an alarm clock event.

This method will revert if the root key isn't held by the caller, the snooze key ID is not within the trust's key ring, or if there happens to be a duplicate event for some reason.

If the snooze interval is zero, then the snoozeKeyId is considered invalid and ignored. The deadline will be considered immutable.&#x20;

If the snooze internval is non-zero, then only a message sender that holds `snoozeKeyId` is able to extend the deadline by `snoozeInterval` amount.

```solidity
/**
 * createAlarm
 *
 * @param rootKeyId      the root key to use to create the event.
 * @param description    a small description of the event
 * @param alarmTime      the timestamp of when the alarm clock should go off
 * @param snoozeInterval the internval to increment the alarm time by when snoozed.
 * @param snoozeKeyId    the key ID from the trust to use to snooze the alarm
 * @return the event hash created for the alarm
 */
function createAlarm(uint256 rootKeyId, bytes32 description, uint256 alarmTime,
        uint256 snoozeInterval, uint256 snoozeKeyId) external returns (bytes32) {

        // ensure the caller is holding the rootKey
        require(IKeyVault(locksmith.getKeyVault()).keyBalanceOf(msg.sender, rootKeyId, false) > 0, 'KEY_NOT_HELD');
        (bool rootValid,, uint256 rootTrustId,bool isRoot,) = locksmith.inspectKey(rootKeyId);
        require(isRoot, 'KEY_NOT_ROOT');

        // if the snooze interval is zero, the alarm effectively cannot
        // be snoozed. If it cannot be snoozed, no real checks need
        // to be done against the snoozeKeyId because its largely considered
        // invalid input anyway.
        if (0 < snoozeInterval) {
            // if we have a non-zero snooze interval, we want to make sure
            // the snooze key is within the trust desginated by the root key
            (bool keyValid,, uint256 keyTrustId,,) = locksmith.inspectKey(snoozeKeyId);
            require(rootValid && keyValid && rootTrustId == keyTrustId, 'INVALID_SNOOZE_KEY');
        }

        // register it in the event log first. If the event hash is a duplicate,
        // it will fail here and the entire transaction will revert.
        bytes32 finalHash = trustEventLog.registerTrustEvent(rootTrustId,
            keccak256(abi.encode(rootKeyId, description, alarmTime, snoozeInterval, snoozeKeyId)),
            description);

        // if we get this far, we know its not a duplicate. Store it
        // here for introspection.
        alarms[finalHash] = Alarm(finalHash, alarmTime, snoozeInterval, snoozeKeyId);

        // emit the oracle creation event
        emit alarmClockRegistered(msg.sender, rootTrustId, rootKeyId,
            alarmTime, snoozeInterval, snoozeKeyId, finalHash);

        return finalHash;
}
```

### snoozeAlarm

If the snooze internval is non-zero, the `snoozeKeyId` holder can call this method to extend the alarm clock's deadline, but only if the deadline is within the snooze interval itself.

```solidity
/**
 * snoozeAlarm
 *
 * @param eventHash   the event you want to snooze the alarm for.
 * @param snoozeKeyId the key the message sender is presenting for permission to snooze 
 * @return the resulting snooze time, if successful.
 */
function snoozeAlarm(bytes32 eventHash, uint256 snoozeKeyId) external returns (uint256) {
        Alarm storage alarm = alarms[eventHash];

        // ensure that the alarm for the event is valid
        require(alarm.eventHash == eventHash, 'INVALID_ALARM_EVENT');

        // ensure that the alarm can be snoozed
        require(alarm.snoozeInterval > 0, 'UNSNOOZABLE_ALARM');

        // ensure the snooze key used by the caller is the correct one
        require(alarm.snoozeKeyId == snoozeKeyId, 'WRONG_SNOOZE_KEY');

        // ensure the caller is holding the proper snooze key
        require(IKeyVault(locksmith.getKeyVault()).keyBalanceOf(msg.sender, snoozeKeyId, false) > 0, 'KEY_NOT_HELD');

        // ensure the event isn't already fired
        require(!trustEventLog.firedEvents(eventHash), 'OVERSNOOZE');

        // ensure that the snooze attempt isn't *too* early, defined by:
        // being late, or within an interval of the alarm time. this prevents
        // a keyholder from snooze-spamming the goal-post into obvilion
        require((block.timestamp >= alarm.alarmTime) ||
            (block.timestamp + alarm.snoozeInterval) >= alarm.alarmTime, 'TOO_EARLY');

        // determine if the snooze is early, or late, and set the proper
        // new alarm time given that all requirements have been met.
        alarm.alarmTime = alarm.snoozeInterval + ((block.timestamp > alarm.alarmTime) ?
            block.timestamp : alarm.alarmTime);

        // the alarm has been snoozed.
        emit alarmClockSnoozed(msg.sender, eventHash, snoozeKeyId, alarm.alarmTime);
        return alarm.alarmTime;
    }
```

### challengeAlarm

If an alarm's deadline has passed, any caller can challenge the alarm and formally trigger the event.

```solidity
function challengeAlarm(bytes32 eventHash) external {
        Alarm storage alarm = alarms[eventHash];

        // ensure that the alarm for the event is valid
        require(alarm.eventHash == eventHash, 'INVALID_ALARM_EVENT');

        // ensure that the alarm has expired
        require(alarm.alarmTime <= block.timestamp, 'CHALLENGE_FAILED');

        // fire the event to the trust event log. this will fail
        // if the event has already been fired with 'DUPLICATE_EVENT'
        trustEventLog.logTrustEvent(eventHash);

        emit alarmClockChallenged(msg.sender, eventHash, alarm.alarmTime,
            block.timestamp);
}
```
