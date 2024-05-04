---
title: Bridging in Juicebox v4
date: 2024-05-03
tags: ["juicebox"]
---

Juicebox v4 introduces the [`BPSucker`](https://github.com/bananapus/nana-suckers) contracts to facilitate bridging project tokens and assets across EVM chains. Here's what you'll need to know if you're building a frontend or service which interacts with them.

<!--more-->

{{< toc >}}

## Basics

`BPSucker` contracts are deployed in pairs, with one on each network being bridged to or from – for now, suckers bridge between Ethereum mainnet and a specific L2. The basic [`BPSucker`](https://github.com/Bananapus/nana-suckers/blob/master/src/BPSucker.sol) contract implements core sucker logic, and is extended by network-specific suckers with custom logic for each L2's bridging solution:

| Sucker                                                                                               | Networks                      | Description                                                                                                                                                                                          |
| ---------------------------------------------------------------------------------------------------- | ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`BPOptimismSucker`](https://github.com/Bananapus/nana-suckers/blob/master/src/BPOptimismSucker.sol) | Ethereum Mainnet and Optimism | Uses the [OP Standard Bridge](https://docs.optimism.io/builders/app-developers/bridging/standard-bridge) and the [OP Messenger](https://docs.optimism.io/builders/app-developers/bridging/messaging) |
| [`BPBaseSucker`](https://github.com/Bananapus/nana-suckers/blob/master/src/BPBaseSucker.sol)         | Ethereum Mainnet and Base     | A thin wrapper around `BPOptimismSucker`                                                                                                                                                             |
| [`BPArbitrumSucker`](https://github.com/Bananapus/nana-suckers/blob/master/src/BPArbitrumSucker.sol) | Ethereum Mainnet and Arbitrum | Uses the [Arbitrum Inbox](https://docs.arbitrum.io/build-decentralized-apps/cross-chain-messaging)                                                                                                   |

## Bridging

Let's imagine the "OhioDAO" project is deployed on Ethereum mainnet and Optimism, with the $OHIO project token and a `BPOptimismSucker` on each network. Each sucker maintains a mapping of its local tokens to equivalent remote tokens (on its peer's chain). We'll assume that OhioDAO's suckers maps mainnet ETH to Optimism ETH, and vice versa. Here's how a user can bridge $OHIO and ETH from mainnet to Optimism:

First, `jimmy.eth` pays OhioDAO 1 ETH on Ethereum mainnet by calling [`JBMultiTerminal.pay(...)`](https://github.com/Bananapus/nana-core/blob/main/src/JBMultiTerminal.sol#L273):

```solidity
JBMultiTerminal.pay{value: 1 ether}({
    projectId: 12, // OhioDAO's project ID
    token: 0x000000000000000000000000000000000000EEEe, // JBConstants.NATIVE_TOKEN
    amount: 1 ether,
    beneficiary: 0x1234..., // jimmy.eth
    minReturnedTokens: 0,
    memo: "OhioDAO rules",
    metadata: 0x // no metadata
});
```

OhioDAO's ruleset has a `weight` of `1e18`, so Jimmy receives 1 $OHIO in return (1e18, with 18 fixed decimals). Before he can bridge his $OHIO to Optimism, Jimmy has to call the $OHIO contract's `ERC20.approve(...)` function to allow the `BPOptimismSucker` to use his balance:

```solidity
JBERC20.approve({
    spender: 0x5678..., // BPOptimismSucker's address
    value: 1e18 // Jimmy's $OHIO balance
});
```

Now, Jimmy can prepare his $OHIO for bridging by calling `BPOptimismSucker.prepare(...)`:

```solidity
BPOptimismSucker.prepare({
    projectTokenAmount: 1e18, // Jimmy's $OHIO balance
    beneficiary: 0x1234..., // jimmy.eth
    minTokensReclaimed: 0,
    token: 0x000000000000000000000000000000000000EEEe // JBConstants.NATIVE_TOKEN
});
```

Once this is called, the sucker:

- Transfers Jimmy's $OHIO to itself.
- Redeems the $OHIO using OhioDAO's primary ETH terminal.
- Adds a new leaf to its ETH outbox tree.

An "outbox tree" is part of what makes the suckers scale well – instead of sending a transaction for each token being bridged, the sucker stores a sparse merkle tree for each local to remote token mapping. We'll call the tree which tracks ETH claims "the ETH outbox tree". The `prepare(...)` function inserts a leaf into that tree, which is a keccak256 hash of the beneficiary's address, the number of project tokens which was redeemed, and the number of terminal tokens that were claimed by that redemption. Only the outbox tree's _root_ needs to be bridged over. Users can claim their tokens on the remote chain by providing a proof that their leaf is in the tree, which gets verified against that root.

To bridge the outbox root over, Jimmy calls `BPOptimismSucker.toRemote(...)`, which takes one argument – the token whose outbox tree should be bridged. Jimmy wants to bridge the ETH outbox tree, so he passes in `0x000000000000000000000000000000000000EEEe` (`JBConstants.NATIVE_TOKEN`). After a few minutes, the sucker will have bridged over the outbox root and the ETH it got by redeeming Jimmy's $OHIO. This calls the peer (on Optimism) sucker's `BPOptimismSucker.fromRemote(...)` function, and updates its ETH inbox tree with the new root.

Jimmy can claim his $OHIO on Optimism by calling `BPOptimismSucker.claim(...)`, which takes a single [`BPClaim`](https://github.com/Bananapus/nana-suckers/blob/master/src/structs/BPClaim.sol) as its argument:

```solidity
struct BPClaim {
    address token;
    BPLeaf leaf;
    // Must be `BPSucker.TREE_DEPTH` long.
    bytes32[32] proof;
}
```

And here's the [`BPLeaf`](https://github.com/Bananapus/nana-suckers/blob/master/src/structs/BPLeaf.sol) struct:

```solidity
/// @notice A leaf in the inbox or outbox tree of a `BPSucker`. Used to `claim` tokens from the inbox tree.
struct BPLeaf {
    uint256 index;
    address beneficiary;
    uint256 projectTokenAmount;
    uint256 terminalTokenAmount;
}
```

These claims can be difficult for integrators to put together, as they require tracking insertions and building merkle proofs for each one. To make this easier, I wrote the [`juicerkle`](https://github.com/Bananapus/juicerkle) service. To use it, `POST` a claims request to `/claims`:

```go
// Schema for incoming claims requests
type ClaimsRequest struct {
	ChainId     *big.Int       `json:"chainId"`     // The chain ID of the sucker contract
	Sucker      common.Address `json:"sucker"`      // The sucker contract address
	Token       common.Address `json:"token"`       // The token address of the inbox tree being claimed from
	Beneficiary common.Address `json:"beneficiary"` // The address of the beneficiary to get the claims for
}
```

Jimmy's claims request might look something like this:

```js
{
    "chainId": 10, // Optimism, the chain we're claiming on
    "sucker": "0x5678...", // The address of the BPOptimismSucker on Optimism
    "token": "0x000000000000000000000000000000000000EEEe", // The token we're claiming (ETH)
    "beneficiary": "0x1234..." // The beneficiary (jimmy.eth)
}
```

The `juicerkle` service will return an array of `BPClaim` structs for each leaf in the ETH inbox tree which has Jimmy's address as its beneficiary and hasn't been claimed yet. The response looks like this:

```js
[
  {
    Token: "0x000000000000000000000000000000000000eeee",
    Leaf: {
      Index: 0,
      Beneficiary: "0x1234...", // jimmy.eth
      ProjectTokenAmount: 1000000000000000000, // 1e18
      TerminalTokenAmount: 1000000000000000000, // 1e18
    },
    Proof: [
      // proof in here
    ],
  },
  // More claims...
];
```

Jimmy can pass this response to `BPOptimismSucker.claim(...)` to claim his $OHIO on Optimism. If the sucker's `ADD_TO_BALANCE_MODE` is set to `ON_CLAIM`, the ETH which Jimmy's $OHIO was redeemed for on Ethereum mainnet will immediately be added to the project's balance. Otherwise, it will be added once someone calls `BPOptimismSucker.addOutstandingAmountToBalance(...)`.

## Launching Suckers

## Using the Relayer
