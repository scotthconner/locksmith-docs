---
description: Asset Rights Decoupling for Virtualized Wallets
---

# 📖 Ledger

## Design Ethos

Having an extensible asset storage interface drives some important goals:

* **Decouple storage from Rights.** The on-chain storage location shouldn't have to care about the internal logic of sub-account distribution. Decoupling these concerns enables other programs to move assets between keys for various reasons without requiring the collateral provider's involvement.
* **Asset and Use Case Diversity.** Support different types of assets, storage, and yield schemes. Specialty NFT vaults, staking providers, exchanges, and bridges can all provide collateral management equally.
* **Unified Management.** View and manage sub-account access, recover locked up assets, and transfer rights for all of your funds, wherever they are stored to anyone within your trust with a key. If there was a RocketPool provider, a loved one recovering the wallet after a death could locate the staked ETH easily within the wallet.

### Ledger and Collateral Relationship

To do this we must separate the Collateral Provider asset storage from the ledgered account. A trusted relationship between the `ICollateralProvider` and the`Ledger` is managed by the `Notary`. The notary respects the root key holder and only enables the Collateral Provider to deposit funds for a proper key, or withdrawal from a key if the key-holder has allowed it.

A couple of trust assumptions are being made in this established relationship:

* The collateral provider trusts, verifies, and obeys the key allocation on the associated ledger.
* The root key holder trusts the collateral provider to faithfully service deposits and withdrawals.

By default, a Locksmith wallet comes with trusted relationships established with both the `EtherVault` and `TokenVault`. These contracts are simple asset storage vaults that service key deposits and withdrawals against the ledger.

### Asset Identity Protocol

Since the `Ledger` is not actually holding the assets, it identifies the assets with what is called an Asset Resource Name (ARN). It is defined as such:

```solidity
struct AssetType {
        // all EVM tokens originate from a contract, gas is zero.
        address contractAddress;

        // identifies the token standard for this asset
        // '0' is considered the native gas token,
        // or is otherwise considered 20, 721, 777, 1155, etc.
        uint256 tokenStandard;

        // for token types that have a non-fungible ID, this
        // field is used to denote that type of asset.
        uint256 id;
}
```

A short form ARN can be derived out of a simple hash scheme to guarantee uniqueness and identifiability. It is represented as a `bytes32`.

```solidity
/**
 * arn 
 *
 * Returns a UUID, or "asset resource name" for the asset type.
 * It is essentially the keccak256 of the asset type struct.
 * This is super convienent for mappings.
 *
 * @param asset the asset you want the arn/UUID for.
 * @return an opaque but unique identifer for this asset type.
 */
 function arn(AssetType memory asset) internal pure returns (bytes32) {
    return keccak256(abi.encode(asset.contractAddress, asset.tokenStandard, asset.id));
 }
```

The assets are tracked in a three-tiered Ledger object that keeps an account of key access rights for each asset actively stored for each collateral provider. The ledger balances are tracked, reconciled, and aggregated for each asset across each key, wallet, and Ledger during each entry. In this way, proof of solvency can be proven when the transaction is successfully added to a block.

<figure><img src="../../.gitbook/assets/Locksmith Architecture - Page 4.png" alt=""><figcaption></figcaption></figure>

The three-tiered structure provides the following immediate transaction guarantees:

* The Ledger verifies the collateral on the ledger is solvent for the provider, wallet owner, and key holder.
* The collateral provider has verified the ledger's account balance to satisfy it's ledger obligations.
* Moving funds within a wallet does not change collateral obligations for any provider.

## Storage

The storage for the ledger is condensed into the three tiered objects that all store the same data structure, a CollateralProviderContext.

```solidity
// the ledger only respects the notary
address public notary;

// ledger context
CollateralProviderLedger.CollateralProviderContext private ledgerContext;
uint256 public constant LEDGER_CONTEXT_ID = 0;

// trust context
mapping(uint256 => CollateralProviderLedger.CollateralProviderContext) private trustContext;
uint256 public constant TRUST_CONTEXT_ID = 1;

// key context
mapping(uint256 => CollateralProviderLedger.CollateralProviderContext) private keyContext;
uint256 public constant KEY_CONTEXT_ID = 2;ol
```

### notary

The Ledger is deloyed with the contract address of the Notary. This cannot be changed. Each operation that mutates the Ledger's balances in any way must be approved by the notary.

### CollateralProviderContext

The following data structure describes each context.

```solidity
struct CollateralProviderContext {
        // all active arns in this context
        EnumerableSet.Bytes32Set arnRegistry;

        // all active collateral providers in this context
        EnumerableSet.AddressSet collateralProviderRegistry;

        // indexing: a list of arns per provider
        mapping(address => EnumerableSet.Bytes32Set) providerArnRegistry;
        // indexing: a list of providers per arn
        mapping(bytes32 => EnumerableSet.AddressSet) arnProviderRegistry;

        // context asset balance
        mapping(bytes32 => uint256) contextArnBalances;
        
        // per-provider arn balance
        mapping(address => mapping(bytes32 => uint256)) contextProviderArnBalances;
}
```

### ledgerContext

This context keeps track of the each per-asset obligation on each collateral provider for the entire ledger across all accounts. Both the Ledger and Collateral Provider check this resulting balance during a deposit or withdrawal to _verify_ accurate books and revert with solvency isn't met.

### trustContext

The trust model context keeps a tabulation of wallet-level balances per provider and asset summed across all keys in the collection. The ledger context should always be equal to or bigger than all wallet balances summed together.

### keyContext

The key context keeps track of the balance rights for a specific key ID on a per asset and collateral provider basis. The summation of all keys rights within a wallet's collection should equal exactly the wallet's balance for each asset.

## Operations

There are a fair mix of introspection operations on top of the core ledger mutations. These include `getContextArnRegistry()`, `getContextProviderRegistry()`, `getContextArnBalances()`, `getContextBalanceSheet()`, and `getContextArnAllocations().`Each of these take a context level (0, 1, 2), and provide back ledger, wallet, or key level assets, providers, and balances in various forms.

### deposit

A collateral provider will call deposit when funds are deposited into a wallet owner's key account, either by a key-holder themselves in a vault-like situation, or as part of the operating agreement between the provider and the wallet owner (for instance, as part of a yield construct).

All deposits for a given trust model must be approved by the Ledger's `Notary`. This ultimately means that the root key holder trusts the provider to store collateral.

The returned vector can be used for inspection and reversion by the collateral provider for solvency guarantees.

```solidity
/**
 * deposit
 * 
 * @param rootKeyId the root key to deposit the funds into
 * @param arn       asset resource hash of the deposited asset
 * @param amount    the amount of that asset deposited.
 * @return final resulting provider arn balance for that key
 * @return final resulting provider arn balance for that trust
 * @return final resulting provider arn balance for the ledger
 */
 function deposit(uint256 rootKeyId, bytes32 arn, uint256 amount) external returns(uint256, uint256, uint256) {
        // make sure the deposit is measurable
        require(amount > 0, 'ZERO_AMOUNT');

        // make sure the provider (the message sender) is trusted
        uint256 trustId = INotary(notary).notarizeDeposit(msg.sender, rootKeyId, 
               arn, amount);

        // make the deposit at the ledger, trust, and key contexts
        uint256 ledgerBalance = ledgerContext.deposit(msg.sender, arn, amount);
        uint256 trustBalance  = trustContext[trustId].deposit(msg.sender, arn, amount);
        uint256 keyBalance    = keyContext[rootKeyId].deposit(msg.sender, arn, amount);
        
        return (keyBalance, trustBalance, ledgerBalance);
    }
```

The deposit method on the context objects are library functions on the struct:

```solidity
/**
 * deposit
 *
 * Use this method to deposit funds from a collateral provider
 * into a context.
 *
 * @param c        the context you want to deposit to
 * @param provider the provider of the collateral
 * @param arn      the arn you want to deposit
 * @param amount   the amount of asset you want to deposit
 * @return the context arn's resulting balance
 */
 function deposit(CollateralProviderContext storage c, address provider, bytes32 arn, uint256 amount) internal returns (uint256) {
        // register the provider and the arn
        c.collateralProviderRegistry.add(provider);
        c.arnRegistry.add(arn);
        c.providerArnRegistry[provider].add(arn);
        c.arnProviderRegistry[arn].add(provider);

        // add the amount to the context and provider totals
        c.contextArnBalances[arn] += amount;
        c.contextProviderArnBalances[provider][arn] += amount;

        // invariant protection: the context balance should be equal or
        // bigger than the provider's balance.
        assert(c.contextArnBalances[arn] >= c.contextProviderArnBalances[provider][arn]);

        return c.contextProviderArnBalances[provider][arn];
 }
```

### withdrawal

Similarly, when a collateral provider is servicing a withdrawal and wants to register the removal of funds with the ledger they call withdrawal. The notary must approve the withdrawal, which requires that the provider is trusted and that the key holder rights in question previously attested to this withdrawal amount.

```solidity
/**
 * withdrawal
 *
 * @param keyId  key to withdrawal the funds from
 * @param arn    asset resource hash of the withdrawn asset
 * @param amount the amount of that asset withdrawn.
 * @return final resulting provider arn balance for that key
 * @return final resulting provider arn balance for that trust
 * @return final resulting provider arn balance for the ledger
 */
 function withdrawal(uint256 keyId, bytes32 arn, uint256 amount) external returns(uint256, uint256, uint256) {
     // make sure the withdrawal is measurable
     require(amount > 0, 'ZERO_AMOUNT');

     // make sure the withdrawal can be notarized with the key-holder
     uint256 trustId = INotary(notary).notarizeWithdrawal(msg.sender, keyId, arn, 
        amount);

     // make the withdrawal at the ledger, trust, and key contexts
     uint256 ledgerBalance = ledgerContext.withdrawal(msg.sender, arn, amount);
     uint256 trustBalance  = trustContext[trustId].withdrawal(msg.sender, arn, amount);
     uint256 keyBalance    = keyContext[keyId].withdrawal(msg.sender, arn, amount);

     // invariant protection
     assert((ledgerBalance >= trustBalance) && (trustBalance >= keyBalance));

     return (keyBalance, trustBalance, ledgerBalance);
 }
```

And similarly, the library function:

```solidity
/**
 * withdrawal
 *
 *
 * @param c        the context you want to withdrawal from
 * @param provider the provider of the collateral
 * @param arn      the arn you want to withdrawal
 * @param amount   the amount of asset you want to remove
 * @return the context arn's resulting balance
 */
 function withdrawal(CollateralProviderContext storage c, address provider, bytes32 arn, uint256 amount) internal returns (uint256) {
        // make sure we are not overdrafting
        require(c.arnRegistry.contains(arn) && 
        c.contextProviderArnBalances[provider][arn] >= amount, "OVERDRAFT");

        // remove the amount from the context and provider totals
        c.contextArnBalances[arn] -= amount;
        c.contextProviderArnBalances[provider][arn] -= amount;

        // remove the arn from the provider arn registry if the amount is zero
        if(c.contextProviderArnBalances[provider][arn] == 0) {
            c.providerArnRegistry[provider].remove(arn);
            c.arnProviderRegistry[arn].remove(provider);
        }
        // remove the arn from the registry if the amount is zero
        if(c.contextArnBalances[arn] == 0) {
            c.arnRegistry.remove(arn);
        }

        // invariant protection: the context balance should be equal or
        // bigger than the provider's balance.
        assert(c.contextArnBalances[arn] >= c.contextProviderArnBalances[provider][arn]);

        return c.contextProviderArnBalances[provider][arn];
}
```

### distribute

Scribes call distribute when they want to re-arrange key rights for existing assets on the ledger. In this way, distributions are not withdrawals. Distributions must be notarized, which requires that the scribe is trusted by the root key holder, and that the distribution keys all belong to the same trust model.

Note that the distribution API takes a provider. Assets can only be atomically re-distributed inside of a given collateral provider. Moving collateral between providers requires a full withdrawal and a deposit at the new custody location. This use can be supported in a single transaction through a `VirtualKeyAddress` agent multi-call.

```solidity
/**
 * distribute
 *
 *
 * @param provider    the provider we are moving collateral for
 * @param arn         the asset we are moving
 * @param sourceKeyId the source key we are moving funds from
 * @param keys        the destination keys we are moving funds to
 * @param amounts     the amounts we are moving into each key
 * @return final resulting balance of that asset for the root key
 */
  function distribute(
      address provider,
      bytes32 arn,
      uint256 sourceKeyId,
      uint256[] calldata keys,
      uint256[] calldata amounts
  ) external returns (uint256) {
      // notarize the distribution
      uint256 trustId = INotary(notary).notarizeDistribution(
          msg.sender, provider, arn, sourceKeyId, keys, amounts);

      // service all the deposits to get the total
      // final withdrawal sum
      uint256 moveSum;
      for(uint256 x = 0; x < keys.length; x++) {
          // don't move any funds to same key
          require(keys[x] != sourceKeyId, 'SELF_DISTRIBUTION');
          
          // move the funds if its greater than zero
          if (amounts[x] > 0 ) {
              keyContext[keys[x]].deposit(provider, arn, amounts[x]);
              moveSum += amounts[x];
          }
      }

      // attempt to withdrawal the funds from the source key
      // and return the final resulting balance to the scribe
      return keyContext[sourceKeyId].withdrawal(provider, arn, moveSum);
  }
```

###
