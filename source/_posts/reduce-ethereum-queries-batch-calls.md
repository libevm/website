---
title: Reducing the Number of Queries to Ethereum
date: 2022-12-23
---

For most Dapps to function, they need to have the latest state -- that means querying the Ethereum blockchain <strong>every block</strong>. So, optimize your queries now before you land yourself a 5 figure AWS bill because you chose an elastic VPS 😎.

Luckily I will help you avoid that and teach you how to retrieve any amount of non-dependent data from Ethereum within a single query. Lets go!

## Multicall2

To achieve this, we will be using [multicall2](https://github.com/makerdao/multicall/blob/master/src/Multicall2.sol) today, and its made by the [MakerDAO](https://makerdao.com/) team so you already know its big brained.

Multicall2 is a contract that is [deployed on Ethereum mainnet](https://etherscan.io/address/0x9695fa23b27022c7dd752b7d64bb5900677ecc21#code) that lives on address `0x9695fa23b27022c7dd752b7d64bb5900677ecc21`. If you'd like to interface with it on your local testnet, all you have to do is fork mainnet or redeploy the contract.

### Querying multiple balances

Suppose we'd like to retrieve how many $USDC is held by this list of addresses, we can do so in <strong>one query</strong> using Mutlicall2.

Make sure your Multicall2 abi [looks like so](https://gist.github.com/libevm/a696a190da47302a10daa4478b93c98a), once that is confirmed, we can construct the payload:

```javascript
const usdc = new ethers.Contract(
    '0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48',
    erc20Abi,
    provider
)

// Construct your calldata to multicall2
const batchCalldata = usdcHolders.map(address => {
    return {
        target: usdc.address,
        callData: usdc.interface.encodeFunctionData("balanceOf", [address])
    }
})
```

Next, this is important, use `callStatic` on the multicall2 contract. What this does behind the scenes is that instead of sending a `eth_sendTransaction` RPC method, it sends a `eth_call`, which is what we need since we're retrieving data and not mutating the blockchain state.

We put false as the first argument to ensure that the batch call doesn't revert in the event that one of the call fails.

```javascript
// Important! Use .callStatic otherwise you'll be
// sending a read transaction if its not read-only
const outputE = await multicall2.callStatic.tryAggregate(
    false,
    batchCalldata
)
```

Once you have your encoded output, you can iterate through each of them and decode them.

```javascript
const outputs = outputE.map(({ success, returnData }, i) => {
    const address = usdcHolders[i]

    if (!success) {
        console.log(`Failed to retrieve usdc.balanceOf for ${address}`)
        return [address, ethers.constants.Zero]
    }

    // Returns an array since we're decoding function result
    // Retreieves the first (and only) item from the array which
    // corresponds to the usdc balance
    const amount = usdc
        .interface
        .decodeFunctionResult("balanceOf", returnData)[0]
    console.log(`usdc.balanceOf(${address}) => ${ethers.utils.formatUnits(amount, 6)}`)
    return [address, amount]
})
```

And there you have it, how to retrieve all non-dependent data in one go.

```bash
$ node index.js
usdc.balanceOf(0xdcef968d416a41cdac0ed8702fac8128a64241a2) => 165259804.908115
usdc.balanceOf(0x467d543e5e4e41aeddf3b6d1997350dd9820a173) => 158786499.413318
usdc.balanceOf(0x1b7baa734c00298b9429b518d621753bb0f6eff2) => 155078679.406831
usdc.balanceOf(0x1a8c53147e7b61c015159723408762fc60a34d17) => 144660359.994582
usdc.balanceOf(0x813c661adf40806666dd0b01d527a3ab3e2a0633) => 131350000.0
usdc.balanceOf(0xaae2c00a079bbff45e09083d72bc2be225936bc7) => 131350000.0
usdc.balanceOf(0x47e6946e48a5ffaa0a361aa97e77774a25c4c150) => 131350000.0
usdc.balanceOf(0x5127f639f29ddbafa96de964d611273a0705be96) => 131350000.0
usdc.balanceOf(0xdcf0ed820fd6219a55c6d42251910864baac0da2) => 131350000.0
usdc.balanceOf(0x56dbfadef270c7c95d9e84db7e921b544a7c4145) => 131350000.0
usdc.balanceOf(0x22f6657450b80d9ba5fec998d8edcbdab149bd42) => 127000000.00155
```

## Links

- [Multicall2-example (ethers.js)](https://github.com/libevm/multicall2-example)
- [Multicall2-contract](https://github.com/makerdao/multicall/blob/master/src/Multicall2.sol)