---
title: Abusing eth_call for Data Retrieval
date: 2023-01-03
---

<img src="https://i.imgur.com/rPAORR0.jpg"/>

In my [previous article](https://www.libevm.com/2022/12/23/reduce-ethereum-queries-batch-calls/) I went through a lesser known method of using [Multicall2](https://github.com/makerdao/multicall/blob/master/src/Multicall2.sol) to perform batch requests to Ethereum. This method however had two limitations:

- Multicall2 contract has to be deployed on target network
- Data requirement can't be dependent, i.e.
    - `getThreshold(msg.sender, getBalance(msg.sender))`

That caught the attention of [DrGorilla](https://twitter.com/drgorilla_md), who graciously [shared his method of data retrieval during contract creation](https://twitter.com/DrGorilla_md/status/1606432095726878722).

<img src="https://i.imgur.com/GV96QB6.jpg"/>

Returning values during contract creation doesn't do much on its own, but when combined with `eth_call`, application specific contracts for data retrieval can now be crafted, without deploying them! 

Basically, this method allows you to retrieve data in any shape way or form, without deploying anything!

I thought this was really cool so I thought I'd write an article about it.

## Returning Data in Constructor

Normally, you can't return anything in the constructor, but with a little `assembly` magic, we're able to do so. Credits to [DrGorilla](https://twitter.com/drgorilla_md) for this.

```javascript
constructor(...) {
    MyType memory returnData = getSomeData(...);

    // insure abi encoding, not needed here but increase reusability for different return types
    // note: abi.encode add a first 32 bytes word with the address of the original data
    bytes memory _abiEncodedData = abi.encode(returnData);

    assembly {
        // Return from the start of the data (discarding the original data address)
        // up to the end of the memory used
        let dataStart := add(_abiEncodedData, 0x20)
        return(dataStart, sub(msize(), dataStart))
    }
}
```

With that, we are able to retrieve values from the constructor, and decode it quite easily with `defaultAbiEncoder`.

## ENS Example

Suppose given a list of addresses, we'd like to retrieve its ENS address. To do so, we first have to:

1. Locate the resolver
2. Query `.name` on the resolver

The data is dependent as we can't do step 2 without the data from step 1.

```javascript
constructor (address[] memory addresses) {
    string[] memory r = new string[](addresses.length);
    for (uint256 i = 0; i < addresses.length; i++) {
        bytes32 node = keccak256(abi.encodePacked(ADDR_REVERSE_NODE, sha3HexAddress(addresses[i])));

        // Get resolver for address
        address resolverAddress = ens.resolver(node);
        if (resolverAddress != address(0x0)) {
            Resolver resolver = Resolver(resolverAddress);

            // Get name
            string memory name = resolver.name(node);
            if (bytes(name).length == 0) {
                continue;
            }

            bytes32 namehash = Namehash.namehash(name);
            address forwardResolverAddress = ens.resolver(namehash);
            if (forwardResolverAddress != address(0x0)) {
                Resolver forwardResolver = Resolver(forwardResolverAddress);
                address forwardAddress = forwardResolver.addr(namehash);
                if (forwardAddress == addresses[i]) {
                    r[i] = name;
                }
            }
        }
    }
}
```

Note: The code snippet is retrieved from [this contract address](https://etherscan.io/address/0x3671aE578E63FdF66ad4F3E12CC0c0d71Ac7510C)

We can then create the payload for `eth_call` and decode the results like so:

```javascript
// Obtain the bytecode needed tp deploy contract
const { data } = FreeENSFactory.getDeployTransaction(ADDRESS_WITH_ENS)

// `eth_call`
const retDataE = await provider.call({ data })

// Format it to ens names
const ensDomains: string[] = ethers.utils.defaultAbiCoder.decode(["string[]"], retDataE)[0]

ensDomains.forEach((x, idx) => {
    console.log(`${ADDRESS_WITH_ENS[idx]} => ${x}`)
})
```

Which will yield:

```bash
$ ts-node scripts-ts/ens.ts
0x8858Ea3b4080bCf6d7B6f5189daE9d8914027Bd0 => xaixvault.eth
0x57001BD30496045ACb1E9bBd507440b301C1d9E3 => bbb.eth
0x60516a59443acc6635B1c952544337De7Cb70eb1 => tb12.eth
0x2536c09E5F5691498805884fa37811Be3b2BDdb4 => xaix.eth
```

## Drawbacks

Your contract (including calldata) can only be a [maximum of 24kb](https://eips.ethereum.org/EIPS/eip-170).

## Links

- [eth_call_abuser](https://github.com/libevm/eth_call_abuser)