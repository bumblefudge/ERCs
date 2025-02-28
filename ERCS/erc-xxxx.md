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
This enables concurrent, segmented channels to nodes of multiple chains (or even chains operating different virtual machines and RPC dictionaries), multiplexed by the wallet (or even across multiple wallets).
This breaks from the long-dominant model codified in [ERC-3326] of wallets maintaining "focus" (in user-experience terms) on one chain at a time, allowing instead for multiple chains to be involved in a given transaction approval or user interaction smoothly, getting up-to-the-current-block information about all the relevant chains in parallel.

It is important to note that each of these parallel connections has a unique and very explicitly defined scope of one or more specific chains (identified in ways specific to each "namespace" of chains, usually the identification system of a given virtual machine or "layer 1" chain).
These identifiers are defined in [CAIP-2] and are tuples of a "namespace" (usually a "shortname" for a registry of chainIds) and a chainId within that namespace.
You could think of these are context-specified chainIds, to disambiguate in case of collisions such as `mainnet` or `1` being used in multiple distinct ecosystems.
The specification for these "shortnames" (including information on how community developers can contribute light/summary documentation of each namespace and the applicability to it of each CAIP) can be found in [CAIP-104].

For each distinct, partioned connection within a [CAIP-25] connection, a kind of configuration object exists, specified in [CAIP-217], outlining the RPC methods (and notifications) authorized, the [chain-specified CAIP-10][CAIP-10] addresses authorized for that scope.
Note that if identical configurations are specified for multiple chains, a "compact" expression is possible listing these in a top-level array of strings called `references` of a connection scoped to an entire namespace, rather than having multiple identical objects each scoped to a single chain.

#### Example CAIP-25 request

The following is an example taken from the [CAIP-25] specification, which is instructive for all of the above:

```JSON
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "wallet_createSession",
  "params": {
    "requiredScopes": {
      "eip155": {
        "references": ["1", "137"],
        "methods": ["eth_sendTransaction", "eth_signTransaction", "eth_sign", "get_balance", "personal_sign"],
        "notifications": ["accountsChanged", "chainChanged"]
      },
      "eip155:10": {
        "methods": ["get_balance"],
        "notifications": ["accountsChanged", "chainChanged"]
      },
      "eip155:0": {
        "methods": ["wallet_getPermissions", "wallet_creds_store", "wallet_creds_verify", "wallet_creds_issue", "wallet_creds_present"],
        "notifications": []
      },
      "cosmos": {
        ...
      }
    },
    "optionalScopes":{
      "eip155:42161": {
        "methods": ["eth_sendTransaction", "eth_signTransaction", "get_balance", "personal_sign"],
        "notifications": ["accountsChanged", "chainChanged"]
    },
    "scopedProperties": {
      "eip155:42161": {
        "extension_foo": "bar"    
      }
    },
    "sessionProperties": {
      "expiry": "2022-12-24T17:07:31+00:00",
      "caip154-mandatory": "true"
    }
  }
}
```
#### Example CAIP-25 response

```JSON
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "sessionId": "0xdeadbeef",
    "sessionScopes": {
      "eip155": {
        "references": ["1", "137"],
        "methods": ["eth_sendTransaction", "eth_signTransaction", "get_balance", "eth_sign", "personal_sign"]
        "notifications": ["accountsChanged", "chainChanged"],
        "accounts": ["eip155:1:0xab16a96d359ec26a11e2c2b3d8f8b8942d5bfcdb", "eip155:137:0xab16a96d359ec26a11e2c2b3d8f8b8942d5bfcdb"]
      },
      "eip155:10": {
        "methods": ["get_balance"],
        "notifications": ["accountsChanged", "chainChanged"],
        "accounts": []
      },
      "eip155:42161": {
        "methods": ["personal_sign"],
        "notifications": ["accountsChanged", "chainChanged"],
        "accounts":["eip155:42161:0x0910e12C68d02B561a34569E1367c9AAb42bd810"],
        "rpcDocuments": "https://example.com/wallet_extension.json"
      },
      "eip155:0": {
        "methods": ["wallet_getPermissions", "wallet_creds_store", "wallet_creds_verify", "wallet_creds_issue", "wallet_creds_present"],
        "notifications": []
      },
      "cosmos": {
        ...
      }
    },      
    "scopedProperties": {
      "eip155:42161": {
        "walletExtensionConfig": {
          "foo": "bar"
        }
      }
    },
    "sessionProperties": {
      "expiry": "2022-11-31T17:07:31+00:00",
      "globalConfig": {
          "foo": "bar"
      }
    }
  }
}
```

In this example:
1. the wallet has added `accounts` arrays to some, but not all, of the parallel connections it has authorized, some empty.
2. no `accounts` have been authorized for the `eip155:0` connection, which refers not to an Ethereum chain but to the dapp<>wallet connection itself, as per the "chainId 0" convention specified in [the Ethereum profile of CAIP-2](https://namespaces.chainagnostic.org/eip155/caip10#special-case-of-eoa).
3. the response merges connections requested as "required" and connections requested as "optional"; wallets can opt to fail on unsupported (or unrecognized) connections marked as required, but are encouraged to drop any unsupported (or unrecognized) optional connections silently.

### Future Work: Browser Extensions and Manifest V3

TBD

## Rationale

TBD

## Security Considerations

TBD
## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

[CAIP-2]: https://chainagnostic.org/CAIPs/caip-2
[CAIP-10]: https://chainagnostic.org/CAIPs/caip-10
[CAIP-25]: https://chainagnostic.org/CAIPs/caip-25
[CAIP-27]: https://chainagnostic.org/CAIPs/caip-27
[CAIP-104]: https://chainagnostic.org/CAIPs/caip-104
[CAIP-217]: https://chainagnostic.org/CAIPs/caip-27
[EIP-1193]: https://eips.ethereum.org/EIPS/eip-1193
[EIP-3085]: https://eips.ethereum.org/EIPS/eip-3085
[EIP-3326]: https://eips.ethereum.org/EIPS/eip-3326
[EIP-5792]: https://eips.ethereum.org/EIPS/eip-5792
[EIP-6963]: https://eips.ethereum.org/EIPS/eip-6963
[EIP-7702]: https://eips.ethereum.org/EIPS/eip-7702
[EIP-7710]: https://eips.ethereum.org/EIPS/eip-7710
[EIP-7715]: https://eips.ethereum.org/EIPS/eip-7715
[namespaces]: https://namespaces.chainagnostic.org
