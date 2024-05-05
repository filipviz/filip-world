---
title: Bridging in Juicebox v4
date: 2024-05-03
tags: ["juicebox"]
---

Juicebox v4 introduces the [`BPSucker`](https://github.com/bananapus/nana-suckers) contracts for bridging project tokens and funds (terminal tokens) across EVM chains. Here's what you'll need to know if you're building a frontend or service which interacts with them.

<!--more-->

{{< toc >}}

## Basics

`BPSucker` contracts are deployed in pairs, with one on each network being bridged to or from – for now, suckers bridge between Ethereum mainnet and a specific L2. The [`BPSucker`](https://github.com/Bananapus/nana-suckers/blob/master/src/BPSucker.sol) contract implements core logic, and is extended by network-specific implementations adapted to each L2's bridging solution:

| Sucker                                                                                               | Networks                      | Description                                                                                                                                                                                                                                |
| ---------------------------------------------------------------------------------------------------- | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [`BPOptimismSucker`](https://github.com/Bananapus/nana-suckers/blob/master/src/BPOptimismSucker.sol) | Ethereum Mainnet and Optimism | Uses the [OP Standard Bridge](https://docs.optimism.io/builders/app-developers/bridging/standard-bridge) and the [OP Messenger](https://docs.optimism.io/builders/app-developers/bridging/messaging)                                       |
| [`BPBaseSucker`](https://github.com/Bananapus/nana-suckers/blob/master/src/BPBaseSucker.sol)         | Ethereum Mainnet and Base     | A thin wrapper around `BPOptimismSucker`                                                                                                                                                                                                   |
| [`BPArbitrumSucker`](https://github.com/Bananapus/nana-suckers/blob/master/src/BPArbitrumSucker.sol) | Ethereum Mainnet and Arbitrum | Uses the [Arbitrum Inbox](https://docs.arbitrum.io/build-decentralized-apps/cross-chain-messaging) and the [Arbitrum Gateway](https://docs.arbitrum.io/build-decentralized-apps/token-bridging/bridge-tokens-programmatically/get-started) |

Suckers use two [merkle trees](https://en.wikipedia.org/wiki/Merkle_tree) to track project token claims associated with each terminal token it supports:

- The _outbox tree_ tracks tokens on the local chain – the network that the sucker is on.
- The _inbox tree_ tracks tokens which have been bridged from the peer chain – the network that the sucker's peer is on.

For example, a sucker which supports bridging ETH and USDC would have four trees – an inbox and outbox tree for each token.

To insert project tokens into the outbox tree, users call `BPSucker.prepare(…)` with:

1. The amount of project tokens to bridge, and
2. the terminal token to bridge with them.

The sucker redeems those project tokens to reclaim the chosen terminal token from the project's primary terminal for it. Then the sucker inserts a claim with this information into the outbox tree.

Anyone can bridge an outbox tree to the peer chain by calling `BPSucker.toRemote(…)`. The outbox tree then _becomes_ the peer sucker's inbox tree for that token. Users can claim their tokens on the peer chain by providing a merkle proof which shows that their claim is in the inbox tree.

## Bridging Tokens

Imagine that the "OhioDAO" project is deployed on Ethereum mainnet and Optimism:

- It has the $OHIO ERC-20 project token and a `BPOptimismSucker` deployed on each network.
- Its suckers map\* mainnet ETH to Optimism ETH, and vice versa.

\* Each sucker has mappings from terminal tokens on the local chain to associated terminal tokens on the remote chain.

_Here's how Jimmy can bridge his $OHIO tokens (and the corresponding ETH) from mainnet to Optimism._

First, Jimmy pays OhioDAO 1 ETH on Ethereum mainnet by calling [`JBMultiTerminal.pay(…)`](https://github.com/Bananapus/nana-core/blob/main/src/JBMultiTerminal.sol#L273):

```solidity
JBMultiTerminal.pay{value: 1 ether}({
    projectId: 12,
    token: 0x000000000000000000000000000000000000EEEe,
    amount: 1 ether,
    beneficiary: 0x1234…,
    minReturnedTokens: 0,
    memo: "OhioDAO rules",
    metadata: 0x
});
```

- `projectId` 12 is OhioDAO's project ID.
- The (terminal) `token` is ETH, represented by [`JBConstants.NATIVE_TOKEN`](https://github.com/Bananapus/nana-core/blob/main/src/libraries/JBConstants.sol)
- The `beneficiary` `0x1234…` is Jimmy's address.

OhioDAO's ruleset has a `weight` of `1e18`, so Jimmy receives 1 $OHIO in return (`1e18` $OHIO). Before he can bridge his $OHIO to Optimism, Jimmy has to call the $OHIO contract's `ERC20.approve(…)` function to allow the `BPOptimismSucker` to use his balance:

```solidity
JBERC20.approve({
    spender: 0x5678…,
    value: 1e18
});
```

The `spender` `0x5678…` is the `BPOptimismSucker`'s Ethereum mainnet address, and the `value` is Jimmy's $OHIO balance. Jimmy can now prepare his $OHIO for bridging by calling `BPOptimismSucker.prepare(…)`:

```solidity
BPOptimismSucker.prepare({
    projectTokenAmount: 1e18,
    beneficiary: 0x1234…,
    minTokensReclaimed: 0,
    token: 0x000000000000000000000000000000000000EEEe
});
```

Once this is called, the sucker:

- Transfers Jimmy's $OHIO to itself.
- Redeems the $OHIO using OhioDAO's primary ETH terminal.
- Adds a claim with this information to its ETH outbox tree.

Specifically, the `prepare(…)` function inserts a leaf into the ETH outbox tree – the leaf is a keccak256 hash of the beneficiary's address, the amount of $OHIO which was redeemed, and the amount of ETH reclaimed by that redemption.

To bridge the outbox tree over, Jimmy (or someone else) calls `BPOptimismSucker.toRemote(…)`, which takes one argument – the terminal token whose outbox tree should be bridged. Jimmy wants to bridge the ETH outbox tree, so he passes in `0x000000000000000000000000000000000000EEEe`. After a few minutes, the sucker will have bridged over the outbox tree and the ETH it got by redeeming Jimmy's $OHIO, which calls the peer sucker's `BPOptimismSucker.fromRemote(…)` function. The Optimism OhioDAO sucker's ETH inbox tree is updated with the new merkle root which contains Jimmy's claim.

Jimmy can claim his $OHIO on Optimism by calling `BPOptimismSucker.claim(…)`, which takes a single [`BPClaim`](https://github.com/Bananapus/nana-suckers/blob/master/src/structs/BPClaim.sol) as its argument. `BPClaim` looks like this:

```solidity
struct BPClaim {
    address token;
    BPLeaf leaf;
    // Must be `BPSucker.TREE_DEPTH` long.
    bytes32[32] proof;
}
```

Here's the [`BPLeaf`](https://github.com/Bananapus/nana-suckers/blob/master/src/structs/BPLeaf.sol) struct:

```solidity
/// @notice A leaf in the inbox or outbox tree of a `BPSucker`. Used to `claim` tokens from the inbox tree.
struct BPLeaf {
    uint256 index;
    address beneficiary;
    uint256 projectTokenAmount;
    uint256 terminalTokenAmount;
}
```

These claims can be difficult for integrators to put together – they would have to track every insertion and build merkle proofs for each one. To make this easier, I wrote the [`juicerkle`](https://github.com/Bananapus/juicerkle) service which returns all of the available claims for a specific beneficiary. To use it, `POST` a json request to `/claims`:

| Field         | JS Type  | Description                                                                |
| ------------- | -------- | -------------------------------------------------------------------------- |
| `chainId`     | `int`    | The network ID for the sucker contract being claimed from.                 |
| `sucker`      | `string` | The address of the sucker being claimed from.                              |
| `token`       | `string` | The address of the terminal token whose inbox tree is being claimed from. |
| `beneficiary` | `string` | The address of the beneficiary we're getting the available claims for.     |

Jimmy's claims request looks like this:

```js
{
    "chainId": 10,
    "sucker": "0x5678…",
    "token": "0x000000000000000000000000000000000000EEEe",
    "beneficiary": "0x1234…" // jimmy.eth
}
```

The `chainId` is Optimism's network ID. Jimmy's getting his claims for the ETH inbox tree of the `BPOptimismSucker` at `0x5678…`. The `juicerkle` service will look through the entire inbox tree and return all of Jimmy's available claims as `BPClaim` structs. The response looks like this:

```js
[
  {
    Token: "0x000000000000000000000000000000000000eeee",
    Leaf: {
      Index: 0,
      Beneficiary: "0x1234…", // jimmy.eth
      ProjectTokenAmount: 1000000000000000000, // 1e18
      TerminalTokenAmount: 1000000000000000000, // 1e18
    },
    Proof: [
      [
        229, 206, 51, 48, 16, 242, 169, 29, 47, 33, 39, 105, 34, 55, 172, 232,
        217, 243, 168, 149, 38, 202, 133, 68, 191, 119, 165, 97, 59, 232, 212,
        14
      ],
      [
        33, 40, 178, 36, 156, 7, 175, 252, 47, 196, 238, 239, 170, 52, 239, 153,
        66, 111, 173, 24, 113, 164, 25, 185, 54, 47, 170, 32, 232, 56, 97, 254
      ],
      // More 32-byte chunks…
    ],
  },
  // More claims…
];
```

Jimmy calls `BPOptimismSucker.claim(…)` with this to claim his $OHIO on Optimism. If the sucker's `ADD_TO_BALANCE_MODE` is set to `ON_CLAIM`, the bridged ETH associated with Jimmy's $OHIO is immediately added to OhioDAO's balance. Otherwise, it will be added once someone calls `BPOptimismSucker.addOutstandingAmountToBalance(…)`.

## Launching Suckers

There are a few requirements for launching a sucker pair:

1. Projects must already be deployed on both chains. The project IDs don't have to match.
2. Both projects must have a 100% redemption rate for the suckers to redeem project tokens for terminal tokens. That is, [`JBRulesetMetadata.redemptionRate`](https://github.com/Bananapus/nana-core/blob/main/src/structs/JBRulesetMetadata.sol) must be `10_000`, which is [`JBConstants.MAX_REDEMPTION_RATE`](https://github.com/Bananapus/nana-core/blob/main/src/libraries/JBConstants.sol).
3. Both projects must allow owner minting for the suckers to mint bridged project tokens. That is, [`JBRulesetMetadata.allowOwnerMinting`](https://github.com/Bananapus/nana-core/blob/main/src/structs/JBRulesetMetadata.sol) must be `true`.
4. Both projects must have an ERC-20 project token. If one doesn't, launch it with [`JBController.deployERC20For(…)`](https://github.com/Bananapus/nana-core/blob/main/src/JBController.sol#L620).

Suckers are deployed through the [`BPSuckerRegistry`](https://github.com/Bananapus/nana-suckers/blob/master/src/BPSuckerRegistry.sol) on each chain. In the process of deploying the suckers, the sucker registry maps local tokens to remote tokens, so we'll have to give it permission:

```solidity
JBPermissionsData memory mapTokenPermission = JBPermissionsData({
    operator: 0x9ABC…,
    projectId: 12,
    permissionIds: [28], // JBPermissionIds.MAP_SUCKER_TOKEN == 28
});

JBPermissions.setPermissionsFor({
    account: 0x1234…,
    permissionsData: mapTokenPermission
});
```

In this example, the project owner `0x1234…` gives the `BPSuckerRegistry` at `0x9ABC…` permission to map tokens for project 12's suckers. Now the owner can deploy the suckers:

```solidity
BPTokenMapping memory ethMapping = BPTokenMapping({
    localToken: 0x000000000000000000000000000000000000EEEe,
    minGas: 100_000, // 100k gas minimum
    remoteToken: 0x000000000000000000000000000000000000EEEe,
    minBridgeAmount: 25e15, // 0.025 ETH
});

BPSuckerDeployerConfig memory config = BPSuckerDeployerConfig({
    deployer: 0xcdef…,
    mappings: [ethMapping]
});

BPSuckerRegistry.deploySuckersFor({
    projectId: 12,
    salt: 0xfce167d38e3d9c2a0375c172d979c39c696f2450616565c1c3284e00f0fac074,
    configurations: [config]
});
```

- The [`BPTokenMapping`](https://github.com/Bananapus/nana-suckers/blob/master/src/structs/BPTokenMapping.sol) maps local mainnet ETH to remote Optimism ETH.
  - To prevent spam, the mapping has a `minBridgeAmount` – ours blocks attempts to bridge less than 0.025 ETH.
  - To prevent transactions from failing, our `minGas` requires a gas limit greater than 100,000 wei.
  - These are good starting values, but you may need to adjust them – if your token has expensive transfer logic, you may need a higher `minGas`.
- The [`BPSuckerDeployerConfig`](https://github.com/Bananapus/nana-suckers/blob/master/src/structs/BPSuckerDeployerConfig.sol) uses the [`BPOptimismSuckerDeployer`](https://github.com/Bananapus/nana-suckers/blob/master/src/deployers/BPOptimismSuckerDeployer.sol) at `0xcdef…` to deploy the sucker.
  - You can only use approved sucker deployers through the registry. Check for `SuckerDeployerAllowed` events or contact the registry's owner to figure out which deployers are approved.
- We call `BPSuckerRegistry.deploySuckersFor(…)` with the project's ID (12), a randomly generated 32-byte salt, and the configuration.
  - **For the suckers to be peers, the `salt` has to match on both chains and the same address must call `deploySuckersFor(…)`.**

The suckers are deployed! We have to give the sucker permission to mint bridged project tokens:

```solidity
JBPermissionsData memory mintPermission = JBPermissionsData({
    operator: 0x1357…,
    projectId: 12,
    permissionIds: [9], // JBPermissionIds.MINT_TOKENS == 9
});

JBPermissions.setPermissionsFor({
    account: 0x1234…,
    permissionsData: mintPermission
});
```

In this example, the project owner `0x1234…` gives their new `BPSucker` at `0x1357…` permission to mint project 12's tokens.

Repeat this process on the other chain to deploy the peer sucker, and the project should be ready for bridging.

## Using the Relayer

_This tech is still under construction – expect this to change._

Bridging from L1 to L2 is straightforward. Bridging from L2 to L1 usually requires an extra step to finalize the withdrawal, which is different for each L2. For OP Stack networks like Optimism or Base, this is the [withdrawal flow](https://docs.optimism.io/stack/protocol/withdrawal-flow):

> 1.  The **withdrawal initiating transaction**, which the user submits on L2.
> 2.  The **withdrawal proving transaction**, which the user submits on L1 to prove that the withdrawal is legitimate (based on a merkle patricia trie root that commits to the state of the L2ToL1MessagePasser's storage on L2)
> 3.  The **withdrawal finalizing transaction**, which the user submits on L1 after the fault challenge period has passed, to actually run the transaction on L1.

Users can do this manually, but it's a hassle. To simplify this process, 0xBA5ED wrote the [`bananapus-sucker-relayer`](https://github.com/Bananapus/bananapus-sucker-relayer), a tool which automatically proves and finalizes withdrawals from Optimism or Base to Ethereum mainnet. It listens for withdrawals and automatically completes the withdrawal process using [OpenZeppelin Defender](https://www.openzeppelin.com/defender).

To use the relayer, project creators have to create an OpenZeppelin Defender account, set up a relayer through their dashboard, and fund it with ETH (to pay gas fees). This relayer is still in development, so expect changes.

## Resources

1. The `nana-suckers` contracts use Nomad's [`MerkleLib`](https://github.com/nomad-xyz/nomad-monorepo/blob/main/solidity/nomad-core/libs/Merkle.sol) merkle tree implementation, which is based on the eth2 deposit contract. I couldn't find a comparable implementation in Golang, so I wrote one which you're welcome to use: the [`tree`](https://github.com/Bananapus/juicerkle/blob/master/tree/tree.go) package in the [`juicerkle`](https://github.com/Bananapus/juicerkle) project. It provides utilities for calculating roots, as well as building and verifying merkle proofs. I use this implementation in the `juicerkle` service to generate claims.
2. To thoroughly test `juicerkle` in practice, I built the end-to-end [`juicerkle-tester`](https://github.com/Bananapus/juicerkle-tester). As well as testing the `juicerkle` service, it serves as a useful bridging process walkthrough – it deploys appropriately configured projects, tokens, and suckers, and bridges between them.
