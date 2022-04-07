---
title: UniswapV3 - Optimal Single Sided Supply
date: 2022-04-06
---

Suppose you want to supply liquidity to a UniswapV3 pool, consisting of tokenA and tokenB, but you only have tokenA, how many tokenA do you swap to tokenB?

Lets also assume that the liquidity range you'd like to supply overlaps with the current market price, so liquidity providing isn't completely in tokenA or tokenB.

How would you go about that?

## Liquidity Providing Woes

Due to the nature of Uniswap V3's concentrated liquidity, providing liquidity isn't as straightforward as having a 50/50 split (in dollar value terms) of tokenA and tokenB.

For example, if the current price of ETH is 3173.67 USDC, and I'd like to supply liquidity between 2757.3 USDC <> 4993.9 USDC, the ratio of ETH and USDC required is a bit skewed in dollar value terms.

![](https://i.imgur.com/qWE0ncE.png)

To supply 10 ETH (31,752 USDC) to the above price range will require only 10,625.5 USDC.

Lets say we don't have any USDC, and we only had 10 ETH. How much ETH should I swap to USDC to provide liquidity to that range? 1 ETH? 1.5 ETH?

## Keep It Simple Stupid

My first thought was that this could be solved using a numerical algorithm. Coming up with a one line magic formula will require you to know the entire state of the Uniswap V3 pool beforehand, and extracting all the data and converting it into something useful is an engineering feat in itself.

Plus, I just wanted something that worked, doesn't have to be the best, just quick and good enough. So, I went along with binary search. (Another use for binary search! Not just for calculating [sandwich attacks](https://github.com/libevm/subway/blob/master/bot/src/numeric.js#L91)).

Breaking it down into pseudocode, it looked something like:

```python
# Token a balance, we don't have token b
token_a_balance = 100

# Naive assumption
token_a_to_swap = token_a_balance / 2
token_b_obtained_from_swap = 0

# Binary search params
MAX_BINARY_SEARCH_ITERATIONS = 32
MAX_DUST = 1
low = 0
high = token_a_balance

for i in range(0, MAX_BINARY_SEARCH_ITERATIONS):
    # How many token_b do we receive from the swap
    token_b_obtained_from_swap = univ3.swap(token_a, token_b, token_a_to_swap)
    
    # How many token_a and token_b were used to supply liquidity
    cur_token_a_balance = token_a_balance - token_a_to_swap
    cur_token_b_balance = token_b_obtained_from_swap
    (token_a_used, token_b_used) = univ3.supply_liquidity(token_a, token_b, cur_token_a_balance, cur_token_b_balance)

    # Calculate leftovers
    leftover_token_a = cur_token_a_balance - token_a_used
    leftover_token_b = cur_token_b_balance - token_b_used

    # Termination condition: We only have dust
    if (leftover_token_a < MAX_DUST and leftover_token_b < MAX_DUST):
        break

    # If we have too many token A, we need to swap more a -> b
    if (leftover_token_a > MAX_DUST and leftover_token_b < MAX_DUST):
        (low, token_a_to_swap, high) = (
            token_a_to_swap,
            high + (token_a_to_swap / 2),
            high
        )

    # If we have too many token B, we need to swap less a -> b
    if (leftover_token_b > MAX_DUST and leftover_token_a < MAX_DUST):
        (low, token_a_to_swap, high) = (
            low,
            low + (token_a_to_swap / 2),
            token_a_to_swap
        )

print('Amount of token A to swap', token_a_to_swap)
```

Keep in mind that its a little bit more complicated in reality as you need deal with Uniswapv3's inner nitty gritty details such as ticks, slots, sqrtRatioX96, liquidity.... 

So, if you're interested in seeing the full version in solidity, do check out my [Univ3-Butler](https://github.com/libevm/univ3-butler/blob/main/src/SingleSidedLiquidityLib.sol#L65) repository. Keep in mind that the function is intended to be called offchain.

## Misery

After banging my head at this problem and independently coming up with a solution, I only found out about:

- [Uniswap's official documentation](https://docs.uniswap.org/sdk/guides/liquidity/swap-and-add) for the same exact issue
- [Uniswap V3 swap-to-add Desmos calculator](https://www.desmos.com/calculator/oiv0rti0ss)

I guess I now know if I need any Uniswap math, [Dan Robinson](https://twitter.com/danrobinson) probably already has a Desmos for that, so just add that to your search term.

I crawled, so you can walk anon.

## Links

- [Univ3-Butler](https://github.com/libevm/univ3-butler)
- [UniswapV3 SDK  - Swap and Add Liquidity Atomically](https://docs.uniswap.org/sdk/guides/liquidity/swap-and-add) for the same exact issue
- [UniswapV3 swap-to-add Desmos calculator](https://www.desmos.com/calculator/oiv0rti0ss)