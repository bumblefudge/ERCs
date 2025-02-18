---
title: Overview of Wallet-Dapp Connections
description: A guide to various EIPs, CAIPs, paradigms, and features
author: Bumblefudge (@bumblefudge), Vandan Parikh (@vandan)
discussions-to: https://ethereum-magicians.org/t/tbd
status: Draft
type: Informational
category: ERC
created: 2025-02-17
---

## Abstract

Since the launch of Ethereum, the development and interface-design of wallets, user-agents, signing, and end-user security have been more competitive than coordinated.
In recent years, however, wallet UX has been experiencing something of a [perhaps still under-coordinated] renaissance, incorporating smart accounts, built-in simulations and reputation systems, and progressively abstracting multichain (and even multi-VM) mechanics.
This document seeks to situate recent developments into a "family tree" of wallet<>dapp connections and features that build on each, to inform the development and strategic planning of wallets and applications alike.

## Specification

### EIP-1193 and Chain-Focus

The dominant model since the early days of Ethereum has been for dapps to search the Domain Object Model (hereafter DOM, the browser's JSON memory namespace for a given browser<>domain connection), and for users to install  one or more browser extensions that inject a `window.ethereum` object for dapps to find, establishing a conventional interface (the EIP-1193 connection) to all websites at that conventional DOM location.
Over time, many RPC method extensions accrued to this model, but the initial connection was not expressive about the wallet's capabilities so they tend towards "try/fail" approaches to additional interfaces.
While Metamask now supports [EIP-6963], the original [EIP-1193] connection was pioneered by Metamask, and the latter is often referred to as a "Metamask-style" wallet connection for that reason.

In the early days of Ethereum multi-chain development, a manual flow for adding additional chains beyond mainnet and interacting with one chain at a time per dapp led to the establishment of a "chain focus" approach, where an end-user set the "focused", i.e. currently-active, chain before connecting to a given dapp.
Many browser-based and mobile wallets inherited this "chain focus" paradigm and even passed it on to hardware wallets, passing along chainId from the "session" when constructing transactions and other messages to be signed.

Over time, this gave way to many dapps proposing to wallets their preferred or unique chains, via the [EIP-3085] RPC method `wallet_addEthereumChain`.
Adding and switching between chains (only one can have "focus" at a time) creates its own security and user-experience hurdles, as does managing multiple addresses, often also allowing only one at a time to have "focus" in user-experience terms.

While EIP-1193 style injection is considered the gold-standard and most widely used across EVM dapp development environments, it also has serious shortcomings.
The "protocol pollution" issue is a major one, leading to malicious wallets impersonating or even replacing Metamask (by analogy to `user-agent` string impersonation, a long-standing weakness of the HTTP security model) by overwriting or front-running the `window.ethereum` polyfill.
Multiple browser extensions could attempt to inject a polyfill there, but race conditions ensued, in that the first to define could also "freeze" the object and block overwriting.
This interface injected into every page also allows malicious pages to intercept and observe the polyfill, potentially deanonymizing users even without interaction.

Additionally, as dapps using this connection mode traditionally pass wallets fully-formed transactions to confirm and sign rather than forming them more interactively, a widespread pattern of exposing at least one address and chain (usually the one with "focus") at time of connection as a method of authentication.
Even without connecting to any malicious dapps, this presents real privacy risks.

### EIP-6963

A concerted community effort lead to the development of [EIP-6963] across 2023, which lead to a pollution-safe event-firing channel outside the DOM over which one (or more!) browser-extension wallets can self-identify to dapps.
This allowed multiple browser-based (or even just browser-registered) wallets to safely coexist in a given browser, and also for dapps to connect to multiple of them concurrently if desired.

In addition to the event loop specified, this standard also defined the JSON object with which wallets self-identify to counterparties listening over that event-loop channel.
This announcement is defined as a [simple dictionary with four mandatory properties][https://eips.ethereum.org/EIPS/eip-6963#reference-implementation]: uuid, human-readable name, icon, and reverse domain-name.
Note that this declaration object does not include anything expressive about wallet capabilities, including non-EVM ones; these are still queried or announced over the wallet<>dapp connection via RPC methods, as before.

### EIP-5792 and `get_capabilities`

One such capability, as well as a generic RPC method for dapps to requesting and wallets to announce information about such capabilities, was defined in [EIP-5792].
Specifically, the main subject of this specification was how a dapp could, over a live connection of the type established via 1193 or 6963 or otherwise, propose a bundle of related transactions that the live account would authorize for signing and execution by a related on-chain account.
This combination of an on-chain element (such as that defined in [EIP-4337]) and a user-agent or "signer" authorized to interact with that on-chain element, is often referred to as a "smart wallet" or "hybrid wallet".

Note that the capability/feature flags passed over this `get_capabilities` RPC method are [explicitly partitioned](https://eips.ethereum.org/EIPS/eip-5792#wallet_getcapabilities-example-return-value) by `chainId`, formatted as a bytestring rather than as an ASCII string.

While [EIP-5792] only defines one such flag, additional ones (perhaps with more complex data shapes) are expected to be defined in forthcoming EIPs.

### EIP-7715 and `wallet_grantPermissions`

While the above abstracts some of the mechanics by which a live connection and an on-chain abstraction interact, they are more explicitly modeled in [EIP-7715], which expresses permissions not between dapp and wallet but between onchain wallet and one or more user-agents that can forward transactions or messages to that onchain wallet. How, conversely, onchain wallets can delegate permissions to other onchain wallets or to signers is specified in [EIP-7710].

In addition to defining a schema for [permissions envelopes](https://eip.tools/eip/7715#permission-schema), the permissions themselves are importantly isomorphic to [EIP-5792] capabilities;
the specification also details how these capabilities negotiated between an on-chain wallet and its controller(s) can also be passed to dapps, over the `get_capabilities` RPC method defined in [EIP-5792] or over a [CAIP-25] connection.

### CAIP-25 and friends

CAIP-25 takes a more expressive approach than EIP-1193, passing back complex structured objects to negotiate wallet<>dapp connections as an interactive communication.
Importantly, these objects can be iterated over time with successive requests to expand or detract the scope of multiple concurrent and partitioned sub-connections, which can be on different chains or even in different VMs and thus distinct sets of RPC methods and ground-assumptions about finality, transaction flow, etc.
Each of these partitioned permissioning namespaces can include 0 or more addresses, different sets of enabled methods, and even free-form metadata in the form of `scopeProperties` (per partitioned scope) and `sessionProperties` (universal across all of them).
The biggest deployment to date of a [CAIP-25]-based connection is the websocket-based connection that the Wallet-Connect SDK has bootstrapped since v2.0, so this connection is sometimes referred to as a "Wallet-Connect connection".

[CAIP-27] defines an envelope for wallet<>dapp RPC calls, routing them to the appropriate "permission partition" (whether across a relay architecture, variously stateful components, variously on-chain component, etc).

### Browser Extensions and Manifest V3

TBD

## Rationale

TBD

## Security Considerations

TBD
## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

[CAIP-25]: https://chainagnostic.org/CAIPs/caip-25
[CAIP-27]: https://chainagnostic.org/CAIPs/caip-27
[EIP-1193]: https://eips.ethereum.org/EIPS/eip-1193
[EIP-3085]: https://eips.ethereum.org/EIPS/eip-3085
[EIP-5792]: https://eips.ethereum.org/EIPS/eip-5792
[EIP-6963]: https://eips.ethereum.org/EIPS/eip-6963
[EIP-7702]: https://eips.ethereum.org/EIPS/eip-7702
[EIP-7710]: https://eips.ethereum.org/EIPS/eip-7710
[EIP-7715]: https://eips.ethereum.org/EIPS/eip-7715
