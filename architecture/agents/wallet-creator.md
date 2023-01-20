---
description: Building a Virtualized Wallet Identity
---

# 📬 Virtual Key Address

## Design Ethos

The `VirtualKeyAddress` is a per-key-instance contract that gives each individual key an on-line contract address to send and receive funds from. Operators of the virtual account must hold a specific key.

Operators have virtualized access to use all of the funds in the trust's collateral providers, and to external parties will see the `msg.sender` be the virtual address of the key's "inbox" and not the EOA account holding the key and signing the transaction.

This contract can use any funds from any asset the key as rights to in single or multi-call transactions.

The contract achieves and maintains the authorization and security by requiring a **soulbound** **key** to be attached to the contract so the contract can act as a definitive key holder with collateral providers and the `Notary`. For non-receive functions, both the caller and the contract must hold the same identity key.

<figure><img src="../../.gitbook/assets/Locksmith Architecture - Page 4 (4).png" alt=""><figcaption><p>Sending funds only requires gas and a key in the traditional wallet.</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/Locksmith Architecture - Page 4 (6).png" alt=""><figcaption><p>Sending funds to the contracts works like a traditional walet, mostly.</p></figcaption></figure>

The VirtualKeyAddress has further extendability to be able to:

* Block senders or receivers based on address.
* Block dustings or unwanted NFTs, tokens.
* Provide multi-key multi-sig

This functionality is a trio of three contracts.

* VirtualKeyAdress: The per-instance contract that enables each key to have a unique address and provides key security.&#x20;
* PostOffice: This is a registry that maintains a mapping of key IDs to contract addresses. This enables discoverability and management for UX. It also helps the root key holder from creating duplicate key addresses by accident.
* KeyAddressFactory: This agent takes possession of a root key long enough to create a copy o the required key, create the inbox, soulbind the new key to the inbox address, and register it with the post office. The root key is given back to the message sender at the end of the transaction.

## VirtualKeyAddress

### Storage

The virtualized wallet keeps an internal model of the transaction history on-chain. It is modeled as such.

<pre class="language-solidity"><code class="lang-solidity">enum TxType { INVALID, SEND, RECEIVE, ABI }
<strong>
</strong><strong>struct Transaction {
</strong>    TxType transactionType; // what type of transaction is it?
    uint256 blockTime;      // when did this transaction happen?
    address operator;       // who is exercising the address?
    address target;         // who is the target of the action?
    address provider;       // what provider is involved?
    bytes32 arn;            // what asset is involved?
    uint256 amount;         // how much of that asset was involved?
}
</code></pre>

The contract storage is as follows:

```solidity
bytes32 public ethArn;

// owner and identity management
uint256 public ownerKeyId;     // the owner of this contract
uint256 public keyId;          // the virtual address "identity"
bool    public keyInitialized; // separates operation from key ID 0 by default

// Collateral provider configuration
IEtherCollateralProvider public defaultEthDepositProvider;
bool    public depositHatch; // this is used to prevent withdrawals from trigger deposits

// Platform references required
address public locksmith;

// chain storage for transaction history
Transaction[] public transactions;
uint256 public transactionCount;
```

#### ethArn

Computed at initialization, stores the ethArn for easy use.

#### ownerKeyId

This is the key ID from the locksmith that legitimately owns the contract and is the only one who can upgrade it.

#### keyId

This is the key ID that the contract assumes the identity of. It will require that for operations that use vaulted collateral to hold this key for successful operation. The contract aslo assumes that a key with this ID is soulbound to the contract given it the authority to act as a key holder on behalf of the caller.

#### keyInitialized

Determines if the key is properly initialized. Protects the keyId from looking like `0` when not initialized.

#### defaultEthDepositProvider

Wallet requires a default ether collateral provider to automatically service deposits. ERC20s do not provide callbacks and thus do not have a default provider.

#### depositHatch

This is a state variable that is used to model re-entrancy that comes from a collateral provider delivering the contract funds during a funding preparation step. Without this hatch, retrieving ether or 721s from vaults would look like incoming deposits from other senders and not preparation for an internal contract operation.

#### locksmith

The reference to the contract used to determine key possession.

#### transactions

The on-chain storage for this wallet's transaction history.

#### transactionCount

The number of transactions recorded in the history.

### Operations

#### requiresKey

This modifier is used for methods that require the message sender holds `keyId` according to the `locksmith`.

```solidity
/**
 * requiresKey
 *
 * An internal implementation that ensures that the operator
 * holds the key required for the given locksmith.
 */
 modifier requiresKey(uint256 key) {
     assert(keyInitialized);
     require(IERC1155(ILocksmith(locksmith).getKeyVault()).balanceOf(msg.sender, key) > 0,
         'INVALID_OPERATOR');
     _;
 }
```

#### send

This method is called by the key holder when they want to send the gas token from a collateral provider to another specific address outside of the trust model. This operations is logically equivalent to sending ether from Metamask to another address, but doing so from within the virtual wallet identity.

```solidity
/**
 * send
 *
 * @param provider the provider address to withdrawal from.
 * @param amount   the raw gwei count to send from the wallet.
 * @param to       the destination address where to send the funds
 */
function send(address provider, uint256 amount, address to) external requiresKey(keyId) {
     // make sure we have enough allowance for the transaction,
     // and leave the allowance as it was before.
     prepareWithdrawalAllowance(provider, ethArn, amount);

     // disable deposits for ether. the money coming back will be used
     // to send as a withdrawal from the trust account
     depositHatch = true;

     // withdrawal the amount into this contract
     ICollateralProvider(provider).arnWithdrawal(keyId, ethArn, amount);

     // re-enable deposits on ether
     depositHatch = false;

     // and send it from here, to ... to.
     (bool sent,) = to.call{value: amount}("");
     assert(sent); // failed to send ether.

     // record and emit entry
     logTransaction(TxType.SEND, to, provider, ethArn, amount);
}
```

**receive**

This method acts as a callback when someone sends the virtual address ether. In this case, the contract takes the designated default ether collateral provider and deposits it directly into the vaults.

```solidity
receive() external payable {
    // don't deposit the money if this is a result
    // of a withdrawal.
    if (depositHatch) { return; }

    // deposit the entire message balance to default collateral provider
    defaultEthDepositProvider.deposit{value: msg.value}(keyId);

    // record and emit entry
    logTransaction(TxType.RECEIVE, address(this),
        address(defaultEthDepositProvider), ethArn, msg.value);
}
```

#### sendToken

This process is nearly identical to `send()`, however, uses the ERC20 standard and the `ICollateralProvider` `arnWithdrawal` interface to facilitate sending ERC20s.

```solidity
/**
 * sendToken
 *
 * @param provider the provider address to withdrawal from.
 * @param token    the contract address of the ERC-20 token.
 * @param amount   the amount of ERC20 to exchange
 * @param to       the destination address of the receiver
 */
 function sendToken(address provider, address token, uint256 amount, address to) external requiresKey(keyId) {
     // calculate the arn for the token
     bytes32 arn = AssetResourceName.AssetType({
         contractAddress: token,
         tokenStandard: 20,
         id: 0
     }).arn();

     // make sure the allowance is unperterbed by this motion
     prepareWithdrawalAllowance(provider, arn, amount);

     // withdrawal the amount into this contract
     ICollateralProvider(provider).arnWithdrawal(keyId, arn, amount);

     // and send it from here, to ... to.
     IERC20(token).transfer(to, amount);

     // record and emit entry
     logTransaction(TxType.SEND, to, provider, arn, amount);
}
```

#### acceptToken

Sending ERC20s do not enable a callback of an on-receive event. For traditional wallets, you also have to add unknown contract address to your wallet if they are not popular enough. Because of this, there is no protocol to automatically deposit ERC20s to a vault. Given a list of known contract addresses, it is possible for the VirtualKeyAddress user to detect tokens in their wallet and "accept" the token, in which the token is deposited into an ERC20 token provider attached to the trust.

This also has the side effect of automatically keeping scam ERC20s away from your private keys and other funds.&#x20;

```solidity
function acceptToken(address token, address provider) external requiresKey(keyId) returns (uint256) {
     ITokenCollateralProvider p = ITokenCollateralProvider(provider);

     // calculate the arn for the token
     bytes32 arn = AssetResourceName.AssetType({
          contractAddress: token,
          tokenStandard: 20,
          id: 0
     }).arn();

     // how much has been here?
     uint256 tokenBalance = IERC20(token).balanceOf(address(this));
     require(tokenBalance > 0, 'NO_TOKENS'); // no reason to waste gas

     // set the allowance for the vault to pull from here
     IERC20(token).approve(provider, tokenBalance);

     // deposit the tokens from this contract into the provider
     p.deposit(keyId, token, tokenBalance);

     // invariant control, we shouldn't have any tokens left
     assert(IERC20(token).balanceOf(address(this)) == 0);

     // record and emit entry
     // note: this will record the "operator" as the key-holder
     //       and not the person sending it. It's not entirely
     //       accurate but solving this problem requires off-chain.
     logTransaction(TxType.RECEIVE, address(this), provider, arn, tokenBalance);

     return tokenBalance;
}

```

#### multicall

#### prepareWithdrawalAllowance

```solidity
/**
     * prepareWithdrawalAllowance
     *
     * For the requested provider, add the amount to the arn's
     * withdrawal limit for the given key for the provider's
     * associated notary.
     *
     * @param provider the address of the collateral provider
     * @param arn      asset resource name to set the limit for
     * @param amount   increase the allowance by this amount
     */
    function prepareWithdrawalAllowance(address provider, bytes32 arn, uint256 amount) internal {
        ICollateralProvider p = ICollateralProvider(provider);
        address ledger = p.getTrustedLedger();
        INotary notary = INotary(ILedger(ledger).notary());

        // cater the withdrawal allowance as to not be perterbed afterwards
        uint256 currentAllowance = notary.withdrawalAllowances(ledger, keyId, provider, arn);
        notary.setWithdrawalAllowance(ledger, provider, keyId, arn, currentAllowance + amount);
    }
```

## PostOffice

### Storage



### Operations

## KeyAddressFactory



### Storage



### Operations
