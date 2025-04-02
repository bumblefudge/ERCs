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

## Introduction

We can establish some terminology upfront to make the following history more clear:

1. **Virtual Machine** refers to broad family tree of each blockchain or DAG ecosystem, commonly called a "protocol" that governs all variants, forks, and "Layer 2"/layer-X instances of that protocol. By this logic, all BTC forks are considered instances of the Bitcoin protocol, and all Polkadot chains (even if not addressable from a public coordination chain) are instances of the Polkadot VM.
2. **Chains** refer to discreet units of addressable state, which in Ethereum and most blockchain virtuam machines are called "chains" or "ledgers" and are addressable by some static or dynamic numbering/naming system. In some virtual machines like DAGs and DHTs, the unit of addressable partition can be subgraphs or shards rather than monotonic chains.
3. **On-Chain Data** refer to actors or resources specific to (and canonical for) a given "chain". Reading, writing, or otherwise interacting with these resources or actors is only possible in the context of a live connection to a participating node of that network.

Also:

* **Multi-Chain** refers to transactions, sessions, or other interactions (on-chain or off-) involving two chains (or subgraphs) of a given protocol, whether these involve oracles, bridges, dual-chain nodes, or multiple nodes. For example, a transaction altering state on both Ethereum Mainnet and Base is considered a multi-chain transaction.
* **Multi-VM** refers to transactions, sessions, or other interactions involving two chains (or subgraphs) of independent protocols. For example, swapping an asset on Ethereum for one on Solana is a multi-VM interaction, and one rarely specified interoperably or subject to direct public discussion and influence.

### Scopes in the Chain-Agnostic Model

The following diagram conveys the CASA URI scheme for multi-VM and multi-chain addressing, by analogy to familiar web URLs:

```bash
┌────────────────────────────────────────────────────────┐
│Virtual Machine: assumptions,about runtime  actors, etc │
│Ex: btc, eip155 (ethereum), solana, cosmos              │
│┌─────────────────────────────────────────────────────┐ │
││"Chains": Addressable authorities for public data    │ │
││Ex: Mainnet, test-nets, private ledgers, sub-graphs  │ │
││┌──────────────────────────────────────────────────┐ │ │
│││On-chain entities: Addressable state              │ │ │
│││Ex: Contracts, registries, wallets, transactions  │ │ │
│││┌───────────────────────────────────────────────┐ │ │ │
││││On-chain sub-entities: VM-specific data        │ │ │ │
││││Ex: A specific NFT or registry entry, metadata │ │ │ │
││││                                               │ │ │ │
│││└───────────────────────────────────────────────┘ │ │ │
││└──────────────────────────────────────────────────┘ │ │
│└─────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────┐
│Protocols (HTTPS, smtp, ftp, etc)                       │
│┌─────────────────────────────────────────────────────┐ │
││Authority: Live server at the root of all URLs       │ │
││┌──────────────────────────────────────────────────┐ │ │
│││Online entities: Resources, inboxes, endpoints    │ │ │
│││ addressed via an authority                       │ │ │
│|│┌───────────────────────────────────────────────┐ │ │ |
││││Sub-Resources and "Assets"                     │ │ │ │
│││└───────────────────────────────────────────────┘ │ │ │
││└──────────────────────────────────────────────────┘ │ │
│└─────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

Applying the general URI scheme defined above, actors and resources are addressed heirarchically by various URI subtypes, all of which follow the general pattern:

`{protocol}:{chain/state identifier}:{entity}[/subentity][?query metadata]`

Subtypes are defined in [CAIPs] (CAIP-10 for accounts, CAIP-19 for assets, etc), and, as needed, each CAIP is profiled per virtual machines, i.e., per ["namespace"][namespaces].

## Specification

### EIP-1193 and Chain-Focus

The dominant model since the early days of Ethereum has been for dapps to search the Domain Object Model (hereafter DOM, the browser's JSON memory partition for a given browser<>domain connection), and for users to install  one or more browser extensions that inject a `window.ethereum` object for dapps to find, establishing a conventional interface (the EIP-1193 connection) to all websites at that conventional DOM location.
Over time, many RPC method extensions accrued to this model, but the initial connection was not expressive about the wallet's capabilities so they tend towards "try/fail" approaches to additional interfaces.
While Metamask now supports [EIP-6963], the original [EIP-1193] connection was pioneered by Metamask, and the latter is often referred to as a "Metamask-style" wallet connection for that reason.

In the early days of Ethereum multi-chain development, a manual flow for adding additional chains beyond mainnet and interacting with one chain at a time per dapp led to the establishment of a "chain focus" approach, where an end-user set the "focused", i.e. currently-active, chain in their wallet interface before or sometimes after connecting to a given dapp.
Many browser-based and mobile wallets inherited this "chain focus" paradigm and even passed it on to hardware wallets, passing along `chainId` and other contextual parameters from the interactive session when constructing transactions and other messages to be signed.

Over time, this gave way to many dapps proposing to wallets their preferred or unique chains, via the [EIP-3085] RPC method `wallet_addEthereumChain`.
Adding and switching between chains (only one can have "focus" at a time) creates its own security and user-experience hurdles. Managing multiple addresses tended towards a "focus" model, adding to the state maintenance and user-experience assumptions expected of wallets (or middleware like hardware-wallet support software).

While EIP-1193 style injection is considered the gold-standard and most widely used across EVM dapp development environments, it also has serious shortcomings.
The "protocol pollution" issue is a major one, leading to malicious wallets impersonating or even replacing Metamask (by analogy to `user-agent` string impersonation, a long-standing weakness of the HTTP security model); this is mostly achieved by overwriting or front-running the `window.ethereum` polyfill.
Multiple browser extensions could attempt to inject a polyfill there, but race conditions ensued, in that the first to define could also "freeze" the object and block overwriting.
This interface injected into every page also allows malicious pages to intercept and observe the polyfill, potentially deanonymizing users even without interaction.

Additionally, as dapps using this connection mode traditionally pass wallets fully-formed transactions to confirm and sign rather than forming them more interactively, a widespread pattern has developed whereby wallets expose (or are even expected by dapps to expose automatically) at least one address and chain (usually the ones with "focus") at time of connection.
This default behavior is sometimes used as "authentication" of that address, which is strictly speaking unsafe as a malicious wallet could claim to control any address; it is also unreliable, as a privacy-maximizing wallet could simply autogenerate a false address to expose to each application.
Even without malice on the part of applications or wallets, this presents something of an anti-pattern for privacy, without little incentive to buck tyhe trend.

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
Over time, a generic (`chainId 0x00`) was added for capabilities that a wallet can provide to any chain in the virtual machine, following the "ChainId 0" convention established years prior by SAFE implementations and integrations.

While [EIP-5792] only defines one such flag, additional ones (perhaps with more complex data shapes) are expected to be defined in forthcoming EIPs.

Note: Since all capabilities are either expressed as chain-specific or VM-specific, it is possible to translate roundtrip and losslessly between the EIP-5792 expression and a CAIP-25 native expression.
The goal here is that EIP-5792 will help applications abstract over the complexities of multiple possible wallet types and wallet-connection types, so that applications can just encode their core logic at the level of batches of on-chain calls, and shape these batches with capability-discovery signals possible over any connection type.

### EIP-7715 and `wallet_grantPermissions`

While the above abstracts some of the mechanics by which a live connection and an on-chain abstraction interact, they are more explicitly modeled in [EIP-7715], which expresses permissions not between dapp and wallet but between onchain wallet and one or more user-agents that can forward transactions or messages to that onchain wallet. How, conversely, onchain wallets can delegate permissions to other onchain wallets or to signers is specified in [EIP-7710].

In addition to defining a schema for [permissions envelopes](https://eip.tools/eip/7715#permission-schema), the permissions themselves are importantly isomorphic to [EIP-5792] capabilities;
the specification also details how these capabilities negotiated between an on-chain wallet and its controller(s) can also be passed to dapps, over the `get_capabilities` RPC method defined in [EIP-5792] or over a [CAIP-25] connection.

### CAIP-25 and friends

CAIP-25 takes a more expressive approach than EIP-1193, passing complex structured objects to and from the wallet (or other user-agent) to negotiate wallet<>app connections as an interactive communication.
These objects are essentially partitioned by "authority" (even if multiple chains are accessed via the same node, authorizations are partitioned into per-chain objects) according to the [chain-agnostic scope model](#scopes-in-the-chain-agnostic-model) diagrammed above.
Importantly, these objects can be iterated over time with successive requests to expand or detract the scope of multiple concurrent and partitioned sub-connections, which can be on different chains or even in different VMs and thus distinct sets of RPC methods and ground-assumptions about finality, transaction flow, etc.
Each of these partitioned permissioning namespaces can include 0 or more addresses, different sets of enabled RPC methods, and even free-form metadata in the form of `scopeProperties` (per partitioned scope) and `sessionProperties` (universal across all of them).
The biggest deployment to date of a [CAIP-25]-based connection is the websocket-based connection that the Wallet-Connect SDK has bootstrapped since version 2.0 of their connection specification and network. For this reason, a CAIP-25 connection is sometimes referred to as a "Wallet-Connect connection".

[CAIP-27] defines an envelope for wallet<>dapp RPC calls, routing them to the appropriate "permission partition" (whether across a relay architecture, variously stateful components, variously on-chain components, etc).
This enables concurrent, segmented channels to nodes of multiple chains (or even chains operating different virtual machines and RPC dictionaries), multiplexed by the wallet (or even across multiple wallets, or a multi-device wallet abstraction).
This breaks from the long-dominant model codified in [ERC-3326] of wallets maintaining "focus" (in user-experience terms) on one chain at a time, allowing instead for multiple chains to be involved in a given transaction approval or user interaction smoothly, getting up-to-the-current-block information about all the relevant chains in parallel.

It is important to note that each of these parallel connections has a unique and very explicitly defined scope of one or more specific chains (identified in ways specific to each "namespace" of chains, usually the identification system of a given virtual machine or "layer 1" chain).
These identifiers are defined in [CAIP-2] and are tuples of a "namespace" (usually a "shortname" for a registry of chainIds) and a chainId within that namespace.
You could think of these are context-specified chainIds, to disambiguate in case of collisions such as `mainnet` or `1` being used in multiple distinct protocol ecosystems.
The specification for these "shortnames" (including information on how community developers can contribute light/summary documentation of each namespace and the applicability to it of each CAIP) can be found in [CAIP-104].

For each distinct, partioned connection within a [CAIP-25] connection, a kind of configuration object exists, specified in [CAIP-217], outlining the RPC methods (and notifications) authorized, the [chain-specified CAIP-10][CAIP-10] addresses authorized for that scope.
Note that if identical configurations are specified for multiple chains, a "compact" expression is possible listing these in a top-level array of strings called `references` of a connection scoped to an entire namespace, rather than having multiple identical objects each scoped to a single chain.
These configuration objects are rigidly typed and unknown properties should be considered unsafe and dropped.
Free-form extensibility does exist, however, in the `scopedProperties` object, where configuration flags, intents, capability objects, etc can be passed; these are partitioned by connection, and any superfluous objects that do not correspond to an authorized connection should be dropped.

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
      "eip155:666666": {
        "methods": ["eth_sendTransactionGoblinMode", "eth_signTransaction", "get_balance", "personal_sign"],
        "notifications": ["accountsRandomized", "replayAttackImminent"]
      },
    },
    "scopedProperties": {
      "eip155:42161": {
        "extension_foo": "bar"    
      },
      "eip155:666666": {
        "goblinMode": "true"    
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
4. an entire optional connection was dropped from the response as it was not authorized by the wallet, and the corresponding `scopedProperties` object annotating this connection was also dropped.

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
[RFC-3986]: https://datatracker.ietf.org/doc/html/rfc3986
[CAIPs]: https://chainagnostic.org
[namespaces]: https://namespaces.chainagnostic.org
