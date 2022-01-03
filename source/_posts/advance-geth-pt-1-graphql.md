---
title: Advance Geth Usage Part 1 - GraphQL
date: 2022-01-03
---

Based on my very narrowed interactions on the blue bird app, there seems to be a great amount of interest from the community on more nuanced usage of Geth.

If you're new here and you're asking why you should care about GraphQL on Geth, please refer to [this twitter thread](https://twitter.com/libevm/status/1467376978697211904). Building on that, this blog post will show an example on how to batch retrieve Uniswap V2 Pair reserves with a single request.

_Hajimemashou_.

# Enable GraphQL On Geth

Graphql isn't enabled on Geth by default, and needs to be enabled with the `--graphql` flag. Note that you'll also need to enable the HTTP server to run graphql.

The example command below enables geth with graphql running at `http://localhost:8545/graphql`.

```bash
geth --graphql --http --http.api web3,net,eth,debug --http.addr "0.0.0.0" --http.port 8545
```

# Storage Slots

Due to how data is stored in the EVM, we cannot retrieve data via function calls (e.g. balanceOf(0x0...)) and will need to specify the exact "slot" (location) of the data.

This is made more explicit in the Account (contract / EOA) schema:

```graphql
# Account is an Ethereum account at a particular block.
type Account {
    # Address is the address owning the account.
    address: Address!
    # Balance is the balance of the account, in wei.
    balance: BigInt!
    # TransactionCount is the number of transactions sent from this account,
    # or in the case of a contract, the number of contracts created. Otherwise
    # known as the nonce.
    transactionCount: Long!
    # Code contains the smart contract code for this account, if the account
    # is a (non-self-destructed) contract.
    code: Bytes!
    # Storage provides access to the storage of a contract account, indexed
    # by its 32 byte slot identifier.
    storage(slot: Bytes32!): Bytes32!
}
```

Where the only way to retrieve storage data from an Account is via a 32 byte storage slot number.

As the purpose of this blogpost is graphql, not EVM storage nuances, all you really need to know is that the `reserve` data structure in the official univ2 pair lives on storage slot `8`.

For a deeper dive into EVM storage slots and how to calculate them, check out this [great blogpost from Adrian Hetman](https://www.adrianhetman.com/unboxing-evm-storage/).

# GraphQL Schema

Assuming the role as an arbitrager, our goal is to calculate the possibility of an arb between two pairs (USDC and WETH).

But, before we can do any computation, we first need to extract the reserves of the USDC and WETH pairs on Sushiswap and Uniswap(v2).

- [USDC/ETH UniswapV2 LP](https://etherscan.io/address/0xB4e16d0168e52d35CaCD2c6185b44281Ec28C9Dc)
- [USDC/ETH Sushiswap LP](https://etherscan.io/address/0x397FF1542f962076d0BFE58eA045FfA2d347ACa0)

As we only care about data at the latest block height, we can ultilize the `Block` and `Account` schema to extract out the pairs' reserves.

```graphql
type Block {
    ...
    # Account fetches an Ethereum account at the current block's state.
    account(address: Address!): Account
    ...
}

```

As an example, we can use the below schema to retrieve the reserves for the USDC and WETH pairs on Sushiswap and Uniswap(v2):

```graphql
{
    block() {
        sushi_weth_usdc: account(address: "0xB4e16d0168e52d35CaCD2c6185b44281Ec28C9Dc") {
            reserve: storage(slot: "0x0000000000000000000000000000000000000000000000000000000000000008")
        }
        uni_weth_usdc: account(address: "0x397FF1542f962076d0BFE58eA045FfA2d347ACa0") {
            reserve: storage(slot: "0x0000000000000000000000000000000000000000000000000000000000000008")
        }
    }
}
```

Once we have the schema we can now query it via whatever language you're comfortable in (I'm using nodejs):

```javascript
const { request } = require('graphql-request')

const main = async() => {
    const resp = await request('http://localhost:8545/graphql', `{ block() {
            sushi_weth_usdc: account(address: "0xB4e16d0168e52d35CaCD2c6185b44281Ec28C9Dc") {
                reserve: storage(slot: "0x0000000000000000000000000000000000000000000000000000000000000008")
            }
            uni_weth_usdc: account(address: "0x397FF1542f962076d0BFE58eA045FfA2d347ACa0") {
                reserve: storage(slot: "0x0000000000000000000000000000000000000000000000000000000000000008")
            }
        }
    }`)

    console.log(resp)
}

main()
```

Giving us the reserve data from TWO pairs from a SINGLE request:

```bash
$ playground node index.js
{
  block: {
    sushi_weth_usdc: {
      reserve: '0x61d2b4110000000005d62391f25da3fd555f00000000000000005f70b59d0b3a'
    },
    uni_weth_usdc: {
      reserve: '0x61d2b3ed00000000082d0ff59adc09cba0ef000000000000000085ae29a2b530'
    }
  }
}
```

# Decode Raw Storage Data

Now obviously the output isn't particularly useful until you decode it, of which you can find an example in my [geth-examples repository](https://github.com/libevm/geth-examples/).

I don't want to put any more code on this blogpost otherwise its gonna look like a C++ tutorial.


# Links

- [GraphQL interface to Ethereum node data -- EIP-1767](https://eips.ethereum.org/EIPS/eip-1767)
- [Geth examples - graphql](https://github.com/libevm/geth-examples/blob/master/graphql/index.js)
- [Unboxing EVM Storage](https://www.adrianhetman.com/unboxing-evm-storage/)