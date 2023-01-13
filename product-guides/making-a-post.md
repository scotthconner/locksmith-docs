---
description: Locksmith Trust Model makes the backbone of wallet security.
---

# 🔑 Key Management

## Locksmith Trust Model

Locksmith uses a singleton ERC-1155 token contract to model permissions called the "Key Vault", and another contract, called the "Locksmith," which adds a trust model on top of the permissions.

Together, these two contracts deliver the Locksmith Trust Model:&#x20;

<figure><img src="../.gitbook/assets/Locksmith Architecture - Page 2.png" alt=""><figcaption><p>A wallet has a single root key.</p></figcaption></figure>

## Root Key

A **root key** is generated when a caller invokes the Locksmith to create a new "Trust." By default, every new trust mints a root key. The root key provides permissions to the system that no other key can do on behalf of their trust:

* Mint, copy, bind, or burn keys within the trust.
* Register or de-register trusted Collateral Providers, Scribes, or Dispatchers with the Notary.
* Create Key Inboxes and register them with the post office.
* Set up rules within the Trustee, Alarm Clock, or Key Oracle feature contracts.

In effect, the root key controls not only the set of active permissions themselves, but who has them, as well as what actors are capable of doing what within the system. In effect, the wallet bends to the will of any active root key holder.
