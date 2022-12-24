---
description: The Locksmith Wallet White Paper
---

# 💡 Abstract

This document describes a system that utilizes ERC-1155 NFT standards to model and secure permissions for an extensible on-chain smart wallet platform.

### Problem Statement

Managing a private key remains a challenge for introductory and expert crypto asset users alike.&#x20;

If a user is simply buying and holding crypto assets for a long time it could entail safely operating a Ledger or managing the security and legibility of written seed phrases. Fear of a damaged Ledger, house fire, flood, or hard-drive crash still motivates many users to hold their assets on centralized exchanges.

For more sophisticated users, managing crypto assets is harder. Keeping cold storage cold,  managing burner wallets and browser extensions, tracking funds across DeFi protocols, and interacting with new contracts safely all come with their own challenges and trade-offs.

The private keys in both scenarios remain the critical challenge - specifically when bringing others into the picture. Introducing inheritance, DeFi vendors, payment services, decentralized teams, or even unscrupulous smart contract owners all increase the surface area of private key exposure and wallet risk.

Solutions vary and some include removing the use of private key altogether. Examples include centralized exchanges, custodial outfits, and other wallets with off-chain identity verification techniques. These solutions allow for automation, but often come with operational trade-offs like centralization, KYC, withdrawal limits, or service level agreements.&#x20;

Other solutions may encrypt and store your private key, offering retrieval and decrypt workflows with various features. Private key storage and retrieval isn't always flexible enough for some circumstances. For example, if a private key is compromised and the key holder has to switch their private key and move assets. Or, if proceeds in an inheritance scenario have multiple beneficiaries a third party must be trusted to properly distribute the assets according to a will.

In both cases if the private key is compromised, the entire portfolio is compromised. For sophisticated users, this problem occurs with each active wallet in use. In both cases, you also have to trade-off between self-custody and programmability/automation.

The pain of balancing usability with security persists for all crypto users.&#x20;
