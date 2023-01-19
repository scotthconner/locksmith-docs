---
description: Simple and Secure Ether Storage
---

# 🔷 Ether Vault

## Design Ethos

The `EtherVault` is a small contract designed to do two things. The first, is to take deposits and store ether in the contract. The only other function it has is to deliver ether withdrawals by checking the message sender's key, and checking the `Ledger` for proper balances before dispersement. It does not properly store or resolve other tokens or NFTs.

## Storage

```solidity
// Locksmith verifies key-holdership.
ILocksmith public locksmith;

// The ledger enforces asset rights
ILedger public ledger;

// We hard-code the arn into the contract.
bytes32 public ethArn;

// keep track of full contract balance for invariant control
 uint256 public etherBalance;
```

The ethArn is defined in the initializer:

```solidity
ethArn = AssetResourceName.AssetType({
    contractAddress: AssetResourceName.GAS_TOKEN_CONTRACT,
    tokenStandard: AssetResourceName.GAS_TOKEN_STANDARD,
    id: AssetResourceName.GAS_ID}).arn();
```

## Operations

The `EtherVault` is a simple vault that mostly facilitates deposits and withdrawals.&#x20;

### deposit

The message sender will send ether to this method and claim a key with which they are claiming to use to deposit. The vault will check to ensure the message sender is in fact holding the key at the time of the call. It then registers the deposit for the key and asset (ETH/gas) based on the message value. The vault keeps track of the ether balance separately from `address(this).balance` to ensure to protect `selfdestruct` or other various means of having more in the contract than is tracked on the ledger.

```solidity
/**
 * deposit
 *
 * @param keyId the ID of the key that the depositor is using.
 */
 function deposit(uint256 keyId) payable external {
     // Make sure the depositor is holding the key in question
     require(IKeyVault(locksmith.getKeyVault()).keyBalanceOf(msg.sender, keyId, false) > 0,
            'KEY_NOT_HELD');

     // track the deposit on the ledger
     // this will revert the entire transaction if this
     // contract isn't trusted
     (,,uint256 finalLedgerBalance) = ledger.deposit(keyId, ethArn, msg.value);

     // once the ledger is updated, keep track of the ether balance
     // here as well. we can't rely on address(this).balance due to
     // self-destruct attacks
     etherBalance += msg.value;

     // jam the vault if the ledger's balance
     // provisions doesn't match the vault balance
     assert(finalLedgerBalance == etherBalance);
 }
```

### withdrawal

There are a few ways the contract enables withdrawals.

```solidity
function withdrawal(uint256 keyId, uint256 amount) external;
function arnWithdrawal(uint256 keyId, bytes32 arn, uint256 amount) external {
    // ensure that the arn is the ethereum arn
    require(arn == ethArn, 'INVALID_ARN');

    // use a standard withdrawal
    _withdrawal(keyId, amount);
}
```

Internally, both interfaces use `_withdrawal`. This method will only succeed if the key is held by the message sender, the key has permission to withdrawal the requested amount from the ledger.

<pre class="language-solidity"><code class="lang-solidity">/**
 * _withdrawal
 *
 * @param keyId  the keyId that identifies both the permissioned 'actor'
 *               and implicitly the associated trust
 * @param amount the amount of ether, in gwei, to withdrawal from the balance.
 */
<strong>function _withdrawal(uint256 keyId, uint256 amount) internal {
</strong>    // stop right now if the message sender doesn't hold the key
    require(IKeyVault(locksmith.getKeyVault()).keyBalanceOf(msg.sender, keyId, false) > 0,
        'KEY_NOT_HELD');

    // withdrawal from the ledger *first*. if there is an overdraft,
    // the entire transaction will revert.
    (,, uint256 finalLedgerBalance) = ledger.withdrawal(keyId, ethArn, amount);

    // decrement the ether balance. We don't want to rely
    // on address(this).balance due to self-destruct attacks
    etherBalance -= amount;

    // jam the vault if the ledger's balance doesn't
    // match the vault's preceived balance after withdrawal
    assert(finalLedgerBalance == etherBalance);

    // let's also make sure nothing else weird is happening, and
    // that even if we are taking self-destruct attacks, that the
    // vault is solvent.
    assert(address(this).balance >= etherBalance);

    // We trust that the ledger didn't overdraft so
    // send at the end to prevent re-entrancy.
    (bool sent,) = msg.sender.call{value: amount}("");
    assert(sent); // failed to send ether.
}
</code></pre>
