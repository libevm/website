---
title: Extending on Reth - Building Custom APIs
date: 2023-09-01
---

# Prelude

This post assumes you have a basic understanding of Rust's syntax and features such as pattern matching, macros, generics, etc. [Rust by example](https://doc.rust-lang.org/rust-by-example/) is a good place to start if you're new.

---

The hottest tool in town is [reth](https://github.com/paradigmxyz/reth) by [Paradigm](https://paradigm.xyz/). I'm usually someone who prefers to wait until tools are a bit more mainstream before adopting them as you get the benefit of the wisdom of the crowd. If you have an issue, there is a high degree of probability that someone else has solved that.

However, the fact that it only consumed 2TB for a full archival node (as opposed to geth's 14TB+) was too tempting for me to not dive into it. And so I did, and I highly recommend anyone to check it out, even if you don't know Rust.

Pros:
- The codebase is clean, well documented and organized
- The project owner are super responsive ([tg group](https://t.me/paradigm_reth))
- Lots of examples to work off from

Cons:
- Rust learning curve takes a few days

# Personal Motivation

The thing that caught my eye was this slide from the [Rust x Ethereum day livestream](https://www.youtube.com/watch?v=5avJdbtpyY8):

![](https://i.imgur.com/UB2Q53q.png)

I cannot stress how much of a nicety it is to be able to extend on reth **while receiving upstream changes**.

Traditionally, forks of geth (such as flashbot's [builder](https://github.com/flashbots/builder/)) would fork out a couple day's worth of engineering time just to merge in upstream changes from Geth. **With reth that is completely eliminated**.

# Extending Reth

Lets add a custom API `eth_getGasUsedByBlock` that accepts a blockTag ("latest", "pending", or block number) and returns the gas used for that particular block.

The goal is to be able to query it via curl like so:

```bash
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_getGasUsedByBlock","params":["0x42069"],"id":67}' localhost:8545  
```

## CLI

We will first begin by adding an additional CLI arg to `reth node` - a boolean enabled by passin in an additional flag `--extended-eth-namespace` to enable our custom API:

```rust
// main.rs
struct RethNodeCliExtended;

impl RethCliExt for RethNodeCliExtended {
    type Node = RethExtended;
}

/// Our custom cli args extension that adds one flag to reth default CLI.
#[derive(Debug, Clone, Default, clap::Args)]
struct RethExtended {
    /// Enables builder mode
    #[clap(long, short)]
    pub extend_eth_namespace: bool,
}
```

Note that the `long, short` in `clap` stands for enabling the toggling of the arg in a long format (`--extended-eth-namespace`) or a short format (`-e`). I know I got confused as I thought it meant that it was a long/short type.

## API

Next up, we will be adding in the API endpoint `eth_getGasUsedByBlock` and the logic needed.

```rust
// gasused.rs
#[rpc[server, namespace="eth"]]
pub trait CustomEthNamespace {
    #[method(name = "getGasUsedByBlock")]
    fn get_gas_used_by_block(&self, block_number: BlockNumberOrTag) -> RpcResult<u64>;
}

pub struct CustomEthNamespaceExt<P> {
    provider: P,
}

impl<P> CustomEthNamespaceExt<P>
where
    P: BlockReaderIdExt + ReceiptProvider + Clone + Unpin + 'static,
{
    pub fn new(provider: P) -> CustomEthNamespaceExt<P> {
        Self { provider }
    }
}

impl<P> CustomEthNamespaceServer for CustomEthNamespaceExt<P>
where
    P: BlockReaderIdExt + ReceiptProvider + Clone + Unpin + 'static,
{
    fn get_gas_used_by_block(&self, bn: BlockNumberOrTag) -> RpcResult<u64> {
        match self.provider.block_by_number_or_tag(bn) {
            Ok(Some(b)) => Ok(b.gas_used),
            _ => {
                error!("unable to retrieve block {bn:?}");
                Err(ErrorObjectOwned::owned(
                    -1,
                    "Invalid blockTag provided",
                    None::<()>,
                ))
            }
        }
    }
}
```

## Entrypoint

Finally, we define the entrypoint, which is as simple as:

```rust
// main.rs
fn main() {
    // Parse args
    Cli::<RethNodeCliExtended>::parse().run().unwrap();
}
```

# Fin

And thats it! It is *that* easy to extend on reth. No more engineering days lost on merging conflicts, just pull and goooooo.

Big kudos to the paradigm team, it was such a pleasure to go through the codespace. 

- [reth-custom-api-eth_getGasUsedByBlock](https://github.com/libevm/reth-custom-api-example/)
- [reth examples (official)](https://github.com/paradigmxyz/reth/tree/main/examples)