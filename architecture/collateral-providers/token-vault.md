---
description: Simple ERC-20 Storage
---

# 🪙 Token Vault

## Design Ethos

ERC-20s operate differently than Ether. Their interface and protocol for moving funds is divergent enough from the gas token to warrant its own interface. As such, the TokenVault takes the same mode as EtherVault, with additions required for the ERC-20 standard.

## Storage

```solidity
// Locksmith verifies key-holdership.
ILocksmith public locksmith;

// The Locksmith provides access to mutate the ledger.
ILedger public ledger;

// witnessed token addresses
// trust => [registered addresses]
mapping(uint256 => EnumerableSet.AddressSet) private witnessedTokenAddresses;
mapping(bytes32 => address) public arnContracts;

// we need to keep track of the deposit balances safely
mapping(address => uint256) tokenBalances;
```

### witnessedTokenAddresses

ARNs are designed to be opaque because they don't inform the asset's behavior, just its uniqueness. For that reason, the TokenVault ensures that it keeps track of every ERC20 token address it's witnessed for each individual trust model.

### arnContracts

During the deposit flow we compute the ARN using the token contract address. We store a mapping between the ARN and the contract for easy access.

### tokenBalances

Similar in nature to EtherVault's etherBalance, is tracked for each token address.

## Operations

The operations support the ability to deposit and withdrawal directly or through the `ICollateralProvider` ARN interface.

### deposit

The message sender calls this after approving the vault's contract the proper amount to move funds from the sender to the contract. The contract ensures the caller is holding the claimed key, ensures the message caller is properly funded and then moves the assets into the vault. After they safely arrive, the deposit is recorded on the ledger. This operation will fail if the vault isn't trusted.

<pre class="language-solidity"><code class="lang-solidity">/**
 * deposit
 *
 * @param keyId the ID of the key that the depositor is using.
 * @param token the address of the ERC20 token contract.
 * @param amount the amount to deposit
<strong> */
</strong><strong>function deposit(uint256 keyId, address token, uint256 amount) external {
</strong>    // stop right now if the message sender doesn't hold the key
    require(IKeyVault(locksmith.getKeyVault()).keyBalanceOf(msg.sender, keyId, false) > 0, 'KEY_NOT_HELD');

    // generate the token arn
    bytes32 tokenArn = AssetResourceName.AssetType({
        contractAddress: token,
        tokenStandard: 20,
        id: 0
    }).arn();

    // store the contract address to enable withdrawals
    arnContracts[tokenArn] = token;

    // make sure the caller has a sufficient token balance.
    require(IERC20(token).balanceOf(msg.sender) >= amount,
        "INSUFFICIENT_TOKENS");

    // transfer tokens in the target token contract. if the
    // control flow ever got back into the callers hands
    // before modifying the ledger we could end up re-entrant.
    IERC20(token).transferFrom(msg.sender, address(this), amount);

    // track the deposit on the ledger
    (,,uint256 finalLedgerBalance) = ledger.deposit(keyId, tokenArn, amount);

    // increment the witnessed token balance
    tokenBalances[token] += amount;

    // jam the vault if the ledger's balance
    // provisions doesn't match the vault balance
    assert(finalLedgerBalance == tokenBalances[token]);
        
    // update the witnessed token addresses, so we can easily describe
    // the trust-level tokens in this vault.
    (,,uint256 trustId,,) = locksmith.inspectKey(keyId);
    witnessedTokenAddresses[trustId].add(token);
}
</code></pre>

### withdrawal

The withdrawal method supports multiple interfaces because it is a multi-asset vault. For this, it facilitates both the direct ERC20 token withdrawal interface, and the ARN-based interface for wallet orchestration.

The direct token method ultimately computes the ARN and calls the internal method.

```solidity
/**
 * withdrawal
 *
 * Given a key, attempt to withdrawal ERC20 from the vault. This will only
 * succeed if the key is held by the user, the key has the permission to
 * withdrawal, the rules of the trust are satisified (whatever those may be),
 * and there is sufficient balance. If any of those fail, the entire
 * transaction will revert and fail.
 *
 * @param keyId  the key you want to use to withdrawal with/from
 * @param token  the token contract representing the ERC20
 * @param amount the amount of ether, in gwei, to withdrawal from the balance.
 */
 function withdrawal(uint256 keyId, address token, uint256 amount) external {
     // generate the ARN, and then withdrawal
     _withdrawal(keyId, AssetResourceName.AssetType({
            contractAddress: token,
            tokenStandard: 20,
            id: 0
     }).arn(), token, amount);
 }
```

The `ICollateralProvider` interface method is a direct bridge to the internal implementation:

```solidity
function arnWithdrawal(uint256 keyId, bytes32 arn, uint256 amount) external {
    // grab the address for the contract. If this ends up being address(0), the
    // ledger should fail to withdrawal, so there is no need to check it here
    _withdrawal(keyId, arn, arnContracts[arn], amount);
}
```

&#x20;With the implementation that feeds both interfaces here:

```solidity
/**
 * _withdrawal
 *
 * @param keyId  the key to withdrawal from the ledger
 * @param arn    the asset idenitifier to withdrawal from the ledger
 * @param token  the token address to use to move assets.
 * @param amount the amount of assets to remove from the ledger, and send.
 */
 function _withdrawal(uint256 keyId, bytes32 arn, address token, uint256 amount) internal {
     // stop right now if the message sender doesn't hold the key
     require(IKeyVault(locksmith.getKeyVault()).keyBalanceOf(msg.sender, keyId, false) > 0, 'KEY_NOT_HELD');

     // withdrawal from the ledger *first*. if there is an overdraft,
     // the entire transaction will revert.
     (,, uint256 finalLedgerBalance) = ledger.withdrawal(keyId, arn, amount);

     // decrement the witnessed token balance
     tokenBalances[token] -= amount;

     // jam the vault if the ledger's balance doesn't
     // match the vault balance after withdrawal
     assert(tokenBalances[token] == finalLedgerBalance);

     // We trust that the ledger didn't overdraft so
     // send at the end to prevent re-entrancy.
     IERC20(token).transfer(msg.sender, amount);
}
```

### getTokenTypes

The `TokenVault` also provides an introspection method to audit all of the token addresses that have been witnessed as deposits for a given trust model.

```solidity
/**
 * getTokenTypes
 *
 * Given a specific key, will return the contract addresses for all
 * ERC20s held in the vault.
 *
 * @param keyId the key you are using to access the trust
 * @return the token registry for that trust
 */
function getTokenTypes(uint256 keyId) external view returns(address[] memory) {
    (,,uint256 trustId,,) = locksmith.inspectKey(keyId);
    return witnessedTokenAddresses[trustId].values();
}
```

