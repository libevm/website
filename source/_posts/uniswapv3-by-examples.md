---
title: UniswapV3 By Examples
date: 2022-03-14
---

I've been trying to find some simple examples on how to interact with Uniswap V3, and couldn't so here is my take on it.

[Repository here](https://github.com/libevm/uniswapv3-by-examples/tree/main/src/test).

## Swaps

```javascript
import "@uniswap-v3-periphery/contracts/interfaces/ISwapRouter.sol";

ISwapRouter v3router = ISwapRouter(0xE592427A0AEce92De3Edee1F18E0157C05861564);

// Exact Input
// WETH -> DAI
v3router.exactInput(
    ISwapRouter.ExactInputParams({
        path: abi.encodePacked(Address.WETH, fee, Address.DAI),
        recipient: address(this),
        deadline: block.timestamp,
        amountIn: 10e18,
        amountOutMinimum: 0
    })
);

// Exact Output
// DAI <- WETH
v3router.exactOutput(
    ISwapRouter.ExactOutputParams({
        // Path is reversed
        path: abi.encodePacked(Address.DAI, fee, Address.WETH),
        recipient: address(this),
        deadline: block.timestamp,
        amountInMaximum: 20e18,
        amountOut: 10000e18
    })
);
```

## Quotes

```javascript
IQuoter v3Quoter = IQuoter(0xb27308f9F90D607463bb33eA1BeBb41C27CE5AB6);

uint256 amountOut1 = v3Quoter.quoteExactInputSingle(
    address(weth),
    address(snx),
    fee,
    1e18,
    0
);
```

## Adding Liquidity

```javascript
INonfungiblePositionManager nfpm = INonfungiblePositionManager(0xC36442b4a4522E871399CD717aBDD847Ab11FE88);

// Get pool current tick, make sure the ticks are correct
(, int24 curTick, , , , , ) = pool.slot0();
curTick = curTick - (curTick % tickSpacing);

int24 lowerTick = curTick - (tickSpacing * 2);
int24 upperTick = curTick + (tickSpacing * 2);

(
    uint256 tokenId,
    uint128 liquidityAdded,
    uint256 amount0Added,
    uint256 amount1Added
) = nfpm.mint(
        INonfungiblePositionManager.MintParams({
            token0: pool.token0(),
            token1: pool.token1(),
            fee: fee,
            tickLower: lowerTick,
            tickUpper: upperTick,
            amount0Desired: token0.balanceOf(address(this)),
            amount1Desired: token1.balanceOf(address(this)),
            amount0Min: 0e18,
            amount1Min: 0e18,
            recipient: address(this),
            deadline: block.timestamp
        })
    );
```

## Removing Liquidity

```javascript
INonfungiblePositionManager nfpm = INonfungiblePositionManager(0xC36442b4a4522E871399CD717aBDD847Ab11FE88);

(uint160 sqrtRatioX96, , , , , , ) = pool.slot0();
(
    ,
    ,
    ,
    ,
    ,
    int24 lowerTick,
    int24 upperTick,
    uint128 liquidity,
    ,
    ,
    ,

) = nfpm.positions(tokenId);
(uint256 amount0, uint256 amount1) = LiquidityAmounts
    .getAmountsForLiquidity(
        sqrtRatioX96,
        lowerTick.getSqrtRatioAtTick(),
        upperTick.getSqrtRatioAtTick(),
        liquidity
    );

(uint256 amount0Removed, uint256 amount1Removed) = nfpm
    .decreaseLiquidity(
        INonfungiblePositionManager.DecreaseLiquidityParams({
            tokenId: tokenId,
            liquidity: liquidity,
            amount0Min: amount0,
            amount1Min: amount1,
            deadline: block.timestamp
        })
    );

// NOTE: This is where tokenTransfer happens
// Collect the decreased liquidity
nfpm.collect(
    INonfungiblePositionManager.CollectParams({
        tokenId: tokenId,
        recipient: address(this),
        amount0Max: uint128(-1),
        amount1Max: uint128(-1)
    })
);

nfpm.burn(tokenId);
```