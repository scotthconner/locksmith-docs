---
description: Building a Virtualized Wallet Identity
---

# 📬 Virtual Key Address

## Design Ethos

The `VirtualKeyAddress` is a per-key-instance contract that gives each individual key an on-line contract address to send and receive funds from. Operators of the virtual account must hold a specific key.

Operators have virtualized access to use all of the funds in the trust's collateral providers, and to external parties will see the `msg.sender` be the virtual address of the key's "inbox" and not the EOA account holding the key and signing the transaction.

This contract can use any funds from any asset the key has rights to for single or multi-call transaction.

The contract achieves and maintains the authorization and security by requiring a **soulbound** **key** to be attached to the contract so the contract can act as a definitive key holder with collateral providers and the `Notary`. For non-receive functions, both the caller and the contract must hold the same identity key.&#x20;

<figure><img src="../../.gitbook/assets/Locksmith Architecture - Page 4 (4).png" alt=""><figcaption><p>Sending funds only requires gas and a key in the traditional wallet.</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/Locksmith Architecture - Page 4 (6).png" alt=""><figcaption><p>Sending funds to the contracts works like a traditional wallet.</p></figcaption></figure>

The VirtualKeyAddress has further extendability to be able to:

* Block senders or receivers based on address.
* Block dustings or unwanted NFTs, tokens.
* Provide multi-key multi-sig

This functionality is a trio of three contracts.

* VirtualKeyAddress: The per-instance contract that enables each key to have a unique address and provides key security.&#x20;
* PostOffice: This is a registry that maintains a mapping of key IDs to contract addresses. This enables discoverability and management for UX. It also helps the root key holder from creating duplicate key addresses by accident.
* KeyAddressFactory: This agent takes possession of a root key long enough to create a copy of the required key, deploy the inbox contract, soulbind the new key to the inbox address, and register it with the post office. The root key is given back to the message sender at the end of the transaction.

## VirtualKeyAddress

### Storage

The virtualized wallet keeps an internal model of the transaction history on-chain. It is modeled as such. One thing to note is receives of ERC20s will be upon "accept" because there is no reliable callback protocol.

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
bytes32 public ethArn; // what is the native gas arn?

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
// this is gas heavy and might warrant an optional disable flag
Transaction[] public transactions;
uint256 public transactionCount;
```

#### ethArn

Computed at initialization, stores the ethArn for easy use. Native gas is handled differently than tokens.

#### ownerKeyId

This is the key ID that legitimately owns the contract and is the only one who can upgrade it. The locksmith reference is the source of truth.

#### keyId

This is the key ID that the contract assumes the identity of. It will require that for operations that use vaulted collateral to hold this key for successful operation. The contract also assumes that a key with this ID is soulbound to the contract giving it the authority to act as a key holder on behalf of the caller.

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
/**
 * acceptToken
 *
 * @param token    the contract address of the token to accept
 * @param provider the collateral provider you want to sweep the funds to
 */ 
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

This method allows the key holder to call in and dynamically orchestrate multiple calls as a virtual identity. The key holder first prepares funds into the contract by specifying which assets should be made available. Then, once in the contract, the funds can be used by calls to execute as an abstracted account. This includes things like single-transaction swaps on Uniswap, among basically anything else valuable an account would want to do and compress into a single transaction.

The function takes two structures - which include enough metadata to describe both the funds needed for the transactions as well as the encoded data for the function calls.

```solidity
/**
 * FundingPreparation
 *
 * A funding preparation is a signal to the virtual address
 * that your multi-call set will likely require funds to be
 * in the Virtual address to successfully complete.
 *
 * The wallet should use this to help prep the contract balance
 * for the rest of the calls.
 */
 struct FundingPreparation {
    address provider;       // the address of the provider to use funds from.
    bytes32 arn;            // the asset resource name of the asset in question
    uint256 amount;         // the amount of the asset needed for the multi-call
 }

/**
 * Call
 *
 * A call is simply a smart contract or send call you want to instruct
 * the virtual address to complete on behalf of the key-holder.
 */
struct Call {
    address target;         // the address you want to operate on
    bytes   callData;       // Fully encoded call structure including function selector
    uint256 msgValue;       // the message value to use when calling
}
```

<pre class="language-solidity"><code class="lang-solidity"><strong> /**
</strong><strong>  * multicall
</strong><strong>  *
</strong><strong>  * @param assets    the assets you want to use for the multi-call
</strong>  * @param calls     the calls you want to make
  */
<strong>function multicall(FundingPreparation[] calldata assets, Call[] calldata calls) payable requiresKey(keyId) external {
</strong>    // go through each funding preparation and
    // dump the funds into this contract as needed.
    depositHatch = true;
    for(uint256 x = 0; x &#x3C; assets.length; x++) {
        prepareWithdrawalAllowance(assets[x].provider, assets[x].arn, assets[x].amount);
        ICollateralProvider(assets[x].provider)
            .arnWithdrawal(keyId, assets[x].arn, assets[x].amount);

        // record and emit entry
        logTransaction(TxType.ABI, address(this),
            assets[x].provider, assets[x].arn, assets[x].amount);
    }
    depositHatch = false;
        
    // generate each target call, and go!
    // Warning: This is re-entrant!!!!
    for(uint y = 0; y &#x3C; calls.length; y++) {
        // let's make sure the target is not the locksmith.
        // we don't want to enable automating permissions at root
        // mid-transaction. the call interface should prevent
        // callData from delegating with the keyholder being the caller.
        require(locksmith != calls[y].target, 'INVARIANT_CONTROL');

        (bool success,) = payable(calls[y].target).call{value: calls[y].msgValue}(calls[y].callData);
        assert(success);
    }
}
</code></pre>

#### prepareWithdrawalAllowance

This is an internal method that is used muiltiple places to ensure that withdrawals from storage providers are properly authorized at the `Notary`. Without this, the ledger will fail to notarize the transaction because the collateral provider isn't cleared to facilitate a transaction.

```solidity
/**
 * prepareWithdrawalAllowance
 *
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

The Post Office is a simple singleton mapping of KeyIDs to their associated Virtual Key Address contract addresses. This ensures that root key holders can find their trust's inbox addresses easily, and to prevent multiple inboxes per key.

### Storage

```solidity
address public locksmith;

// is an inbox address registered already?
mapping(address => bool) private inboxes;

// which address, if any, is the virtual inbox for a specific key ID?
mapping(uint256 => address) private keyIdentityInboxes;

// what are all the inboxes a key holder claims to own?
// ownership could easily rug on the factoried contract and
// leave this stale, but that would be a bug that could
// also be detected and explained as to how that happened.
// keyId => [inbox]
mapping(uint256 => EnumerableSet.AddressSet) private ownerKeyInboxes;
```

#### locksmith

The refernce to the locksmith that is used to verify key possession.

#### inboxes

An index mapping of each address to whether or not it is a valid inbox registered with the Post Office.

#### keyIdentityInboxes

For each Key that has an inbox, what is it's address?

#### ownerKeyInboxes

Find the key inboxes that are owned by an individual key. A key (like root), can own many inboxes. Ownership is separate from the inbox "identity" (`keyId`)

### Operations

There are two introspection methods, `getInboxesForKey()`, and `getKeyInboxes()` which will not be covered in depth. They facilitate seeing what inbox addresses are owned by a given key, and what contract address exists, if any, for a given key Id.

#### registerInbox

This method is called by the inbox owner, who must be root, to register with the Post Office. The VirtualKeyAddress must already be deployed and configured.

Most of the work is to ensure that the request is valid. The address must be a unique address and identity key. The virtual address contract must also be owned by a root key, that the message sender also holds. The key identity for the virtual address needs to be a key within the root key's trust.

```solidity
/**
 * registerInbox
 *
 * @param inbox the address of the inbox to register.
 */
function registerInbox(address payable inbox) external {
    // make sure the inbox isn't already registered
    require(!inboxes[inbox], 'DUPLICATE_ADDRESS_REGISTRATION');

    // determine what key the inbox thinks its owned by
    uint256 ownerKey = IVirtualAddress(inbox).ownerKeyId();
    uint256 keyId = IVirtualAddress(inbox).keyId();

    // ensure that the owner key is a root key, and that
    // the keyId is within the ring.
    (bool ownerValid,, uint256 ownerTrustId, bool ownerIsRoot,) = ILocksmith(locksmith).inspectKey(ownerKey);
    (bool targetValid,, uint256 targetTrustId,,) = ILocksmith(locksmith).inspectKey(keyId);
    require(ownerValid && ownerIsRoot, 'OWNER_NOT_ROOT');
    require(targetValid && (targetTrustId == ownerTrustId), 'INVALID_INBOX_KEY');

    // ensure that the message sender is holding the owner key
    require(IKeyVault(ILocksmith(locksmith).getKeyVault()).keyBalanceOf(msg.sender, ownerKey, false) > 0,
        'KEY_NOT_HELD');

    // make sure the key isn't already registered
    require(keyIdentityInboxes[keyId] == address(0), 'DUPLICATE_KEY_REGISTRATION');

    // register the inbox
    inboxes[inbox] = true;
    ownerKeyInboxes[ownerKey].add(inbox);
    keyIdentityInboxes[keyId] = inbox;

    emit addressRegistrationEvent(InboxEventType.ADD, msg.sender, ownerKey, inbox);
}
```

#### deregisterInbox

There may be circumstances where an inbox will want to be removed. This will remove the inbox from the index. This may be in the case where no upgrade path exists for a given virtual address implementation, so it may need to be fully replaced.

```solidity
/**
 * deregisterInbox
 *
 * @param ownerKeyId the key holder that once claimed to own it
 * @param inbox      the address of the IVirtualAddress to deregister
 */
function deregisterInbox(uint256 ownerKeyId, address payable inbox) external {
    // fail if the inbox isn't registered
    require(inboxes[inbox], 'MISSING_REGISTRATION');

    // fail if the inbox's keyID doesn't match the registraton
    uint256 keyId = IVirtualAddress(inbox).keyId();
    require(keyIdentityInboxes[keyId] == inbox, 'CORRUPT_IDENTITY');

    // fail if the message sender isn't holding the key
    require(IKeyVault(ILocksmith(locksmith).getKeyVault()).keyBalanceOf(msg.sender, ownerKeyId, false) > 0,
        'KEY_NOT_HELD');

    // we don't actually care if they still own the inbox on-chain,
    // just that they want to de-register a valid entry for *them*
    require(ownerKeyInboxes[ownerKeyId].remove(inbox), 'REGISTRATION_NOT_YOURS');

    // clean up the bit table
    inboxes[inbox] = false;
    keyIdentityInboxes[keyId] = address(0);
    emit addressRegistrationEvent(InboxEventType.REMOVE, msg.sender, ownerKeyId, inbox);
}
```

## KeyAddressFactory

This is a simple contract that, upon receiving a root key, will create a brand new contract using `CREATE2` to build a virtual address for a given key. It does this by using the root key to create a copy of the identity key, deploys the new contract, soulbinds the copied key to the new contract address, registers the address with the `PostOffice`, and then finally returns the root key to the message sender at the end of the transaction.

### Storage

```
IPostOffice public postOffice;

struct InboxRequest {
    uint256 virtualKeyId;
    address defaultEthDepositProvider;
}
```

#### postOffice

This is a reference to the post office contract that the factory will use to register the inbox with.

#### struct InboxRequest

These two fields specify the request as serialized data along with sending the actual root key required to do the work in a single transaction. The `virtualKeyId` is the identity of the key that will be soulbound to the generated address, and the `defaultEthDepositProvider` is the trusted provider where gas will be sent when received.

### Operations

There is only a single operation, `onERC1155Received`, and assumes the NFT sent into the contract belongs to a valid Locksmith.

#### onERC1155Received

```solidity
function onERC1155Received(
    address,
    address from,
    uint256 keyId,
    uint256 count,
    bytes memory data
) public virtual override returns (bytes4) {
    
    // make sure the count is exactly 1 of whatever it is.
    require(count == 1, 'IMPROPER_KEY_INPUT');

    // recover the dependencies
    IKeyVault keyVault = IKeyVault(msg.sender);
    address locksmith = keyVault.locksmith();

    // make sure the locksmith's between the Post Office
    // and the key sent in are the same
    require(locksmith == postOffice.locksmith(), 'LOCKSMITH_MISMATCH');

    // grab the encoded information
    InboxRequest memory request = abi.decode(data, (InboxRequest));

    // deploy the implementation
    address inbox = address(new VirtualKeyAddress());

    // deploy the proxy, and call the initialize method through it
    ERC1967Proxy proxy = new ERC1967Proxy(inbox,
        abi.encodeWithSignature('initialize(address,address,uint256,uint256)',
            locksmith, request.defaultEthDepositProvider, keyId, request.virtualKeyId));

    // mint a soul-bound key into the new proxy
    ILocksmith(locksmith).copyKey(keyId, request.virtualKeyId, address(proxy), true);

    // add the proxy to the registry - this will revert
    // the transaction if its a duplicate. this will also revert
    // if the key configuration is bad for some reason.
    postOffice.registerInbox(payable(proxy));

    // send the key back!
    IERC1155(msg.sender).safeTransferFrom(address(this), from, keyId, 1, "");
    
    return this.onERC1155Received.selector;
}
```
