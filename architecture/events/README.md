---
description: Events and Dispatchers
---

# ⚡ Event Dispatch

Composability requires the ability to send messages from one program to another. To enable different modules inside of Locksmith easily talk to each other, a `Dispatcher` and `Event` model is provided.

A `Dispatcher` is a module that has permission to reigster and fire event inside of a trust model. The `Dispatcher` is trusted by the Notary via a root key holders attestation.

The events are stored on an on-chain message bus, called the `TrustEventLog`.&#x20;
