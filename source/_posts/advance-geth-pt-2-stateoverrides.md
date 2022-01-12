---
title: Advance Geth Usage Part 2 - State Overrides
date: 2022-01-12
---

`eth_call` is a very common and widely used RPC method in geth. One lesser known feature about this RPC method (in geth) is the state override set.

![](https://i.imgur.com/4LxdmCV.png)

In short, the state override set allows you to deploy contracts and/or change the state of any contracts **on demand** while using `eth_call`.

This is an extremely useful for debugging against mainnet state. This blogpost will explore this concept while providing some very simple examples to make things more concrete.

## Mocking Approvals on Contracts

While playing around in MEV land, I found a couple of the lesser known DEXs **did not** have a 'view' function to calculate how many tokens I would get out given in. For example Uniswap's `getAmountOut`, or Curve's `get_dy`.

This was incredibly inconvenient while doing off-chain calculations as in order for me to get a result I needed to perform a staticCall on the actual swap method -- requiring me to first approve the token before I could get an answer.

In addition to that, the maximum amount of token in it could estimate was the balance of the account doing the calls...

_Mendokusai_.

Unfortunately replicating in "INSERT LANGUAGE HERE" seemed to be non-trivial. It was also not Uniswap/Sushiswap level big, so it wasn't worth investing significant engineering hours into this issue.

But still, if we had an easy way to estimate how much token out we would get for token in, it would create a new arbitrage path. If only I knew what I knew then.

### stateDiff

By using the `stateDiff` field (and some fancy index calculations) we can mock an approval while doing `eth_call`.

As the main goal of this blogpost is to shed light on state overrides, we will be skipping over the slot index calculations.

With that, here's an example of a stateDiff object:

```javascript
const dai = "0x6b175474e89094c44da98b954eedeac495271d0f";
const fromAddr = "0xde0B295669a9FD93d5F28D9Ec85E40f4cb697BAe";
const toAddr = "0x52bc44d5378309ee2abf1539bf71de1b7d7be3b5";
const slot = 3; // Allowance slot on DAI (differs from contract to contract)

// Nested mappings
// https://ethereum.stackexchange.com/questions/102037/storage-limit-of-2-level-mapping/102220
const temp = ethers.utils.solidityKeccak256(
  ["uint256", "uint256"],
  [fromAddr, slot]
);
const index = ethers.utils.solidityKeccak256(
  ["uint256", "uint256"],
  [toAddr, temp]
);

// State Diff example
const stateDiff = {
  [dai]: {
    stateDiff: {
      [index]: ethers.constants.MaxUint256.toHexString(),
    },
  },
};
```

To perform a call via `ethers.js` we can do it like so:

```javascript
// Call with no state overrides
const call1 = await provider.send("eth_call", callParams);

// Call with no state overrides
const call2 = await provider.send("eth_call", [...callParams, stateDiff]);
```

And voila:

```bash
$ node state-override/allowance-erc20.js
Allowance of from -> to without stateDiff 0x0000000000000000000000000000000000000000000000000000000000000000
Allowance of from -> to *with* stateDiff  0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
```

## There's More!

There's more that you can do than overriding specific slots, you can also change the balance (ETH), and code of a particular address! For a more descriptive example, check out this [repository by devanonon](https://github.com/devanonon/TokenChecks/blob/master/scripts/tokenOverride.ts#L42).

## Links

- [StateDiff minimal example in JS](https://github.com/libevm/geth-examples/blob/master/state-override/allowance-erc20.js)
- [Nested mappings slot](https://ethereum.stackexchange.com/questions/102037/storage-limit-of-2-level-mapping/102220)
- [Code override example, devanonon](https://github.com/devanonon/TokenChecks/blob/master/scripts/tokenOverride.ts#L42)