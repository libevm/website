---
title: Yul+ Smart Contract Development
date: 2021-12-18
---

While open sourcing my sandwich bot's [smart contracts](https://twitter.com/libevm/status/1469934003939463168) in Solidity, I got Yul-pilled by [OkCupid](https://twitter.com/cupidhack/status/1469475023849017344).

[Apparently](https://www.youtube.com/watch?v=rz5TGN7eUcM), Yul+ generates smaller bytecode than Solidity. Smaller bytecode footprint = less gas used = less ETH burned = ETH less deflationary. #Winning? Maybe.

![](https://i.imgur.com/E534MeW.png)
<sub>[source](https://twitter.com/cupidhack/status/1469475023849017344/)</sub>

However, according to serial CEO [@samczsun](https://twitter.com/samczsun) (CEO of FTX, Binance, and (formerly) Tron foundation), the additional bytecode generated was mostly constructor logic and some metadata junk -- meaning that the gas efficiency during **runtime** should more or less be the same as **inline assembly solidity**.

![](https://i.imgur.com/l2zRZbI.png)

Being the ~~naive~~ curious chap I was, I decided to try out Yul+ for myself, and see if those Yul+ coomers were really onto something or if its was .

But, before I could really write any code, I was already cucked by the dev tooling environment -- apart from [ControlCplusControlV's Yul+ plugin for hardhat](https://github.com/ControlCplusControlV/hardhat-Yul) (which I couldn't get working), there wasn't really anything substancial out there in the wild. The only other thing I could really find was the [fuel-v1-contracts](https://github.com/FuelLabs/fuel-v1-contracts), but the build environment and tooling was incredibly custom and specific to their setup.

I don't really enjoy re-inventing the wheel and wanted to try leverage an existing dev-tool and add support for Yul+. And, to be fair, I tried to get ControlCplusControlV's plugin to work, however after getting various errors and wanting to KMS too many times I gave up.

That is, until I remembered that [dapp tools](https://dapp.tools/) had [FFI](https://en.wikipedia.org/wiki/Foreign_function_interface) support.

## FFI

FFI stands for Foreign Function Interface, what that means is that you can **execute arbitrary shell commands** within the system shell, the output of which can be parsed by [dapp.tools](https://dapp.tools). This feature is disabled by default due to security concerns and can be enabled with the `--ffi` flag, i.e.:

```
dapp test --ffi
```

**Important**: The stdout generated needs to be abi-encoded for [dapp.tools](https://dapp.tools) to parse it.

## Text Editor

To get syntax highlighting with Yul+ files, I use VSode and the Solidity plugin with the following settings (thanks random flashbots user).

```json
{
    "files.associations": {
        "*.yul": "solidity",
        "*.yulp": "solidity",
    },
    "solidity.enabledAsYouTypeCompilationErrorCheck": false,
    "editor.tabSize": 2
}
```

## Yul+

The official [Yul+](https://github.com/FuelLabs/yulp) 'compiler' is written JavaScript, and actually compiles Yul+ down to Yul before turning it into bytecode.

Thus, as of right now you'll still need a custom JS script to compile your Yul+ source code into EVM compatible bytecode.

## Shoehorning Yul+ to work with dapp.tools

Why [dapp.tools](https://dapp.tools)?

Well,

- God-like debugger
- Informative stacktraces
- Rocks-solid EVM emulation (thanks Haskell!)

That cool.... but doesn't [dapp.tools](https//dapp.tools) only support Solidiy? Well, yes, until FFI dropped. With FFI you can now compile your contracts from whatever medium, and as long as you can log out the EVM compatible bytecode to stdout, you can use [daap.tools](https://dapp.tools) as your development framework.

### Compiling

We currently know that:

- Yul+ requires JS to compile
- FFI takes in arbitrary shell commands

Thus, its a no-brainer to write your Yul+ compile script in JS. For reference, here's mine:

```javascript
// scripts/compile.js
const yulp = require('yulp')
const solc = require('solc')
const fs = require('fs')
const path = require('path')

const CONTRACTS_DIR = path.join(__dirname, '..', 'src')
const OUT_DIR = path.join(__dirname, '..', 'out')

const sourceCode = fs.readFileSync(path.join(CONTRACTS_DIR, 'Sandwich.yulp'), { encoding: 'ascii' })
const source = yulp.compile(sourceCode)

const output = JSON.parse(solc.compile(JSON.stringify({
    "language": "Yul",
    "sources": { "Sandwich.yul": { "content": yulp.print(source.results) } },
    "settings": {
        "outputSelection": { "*": { "*": ["*"], "": ["*"] } },
        "optimizer": {
            "enabled": true,
            "runs": 0,
            "details": {
                "yul": true
            }
        }
    }
})))

if (!fs.existsSync(OUT_DIR)) {
    fs.mkdirSync(OUT_DIR)
}

const abi = source.signatures.map(v => v.abi.slice(4, -1)).concat(source.topics.map(v => v.abi.slice(6, -1)))
const bytecode = output.contracts["Sandwich.yul"]["Sandwich"]["evm"]["bytecode"]["object"]

fs.writeFileSync(path.join(OUT_DIR, 'sandwich.out.json'), JSON.stringify(output))
fs.writeFileSync(path.join(OUT_DIR, 'sandwich.abi'), JSON.stringify(abi))
fs.writeFileSync(path.join(OUT_DIR, 'sandwich.bytecode'), bytecode)
```

The above compile script only works for a single file -- `sandwich.yulp`. However with a bit of tweaking you can make it work for any number of contracts with any names.

### Deploying via FFI

Now that we have the bytecode, we now need another script to retrieve the bytecode and output it to stdout in **abi encoded** format.

```javascript
// scripts/getBytecode.js
const { ethers } = require("ethers");
const path = require('path')
const fs = require("fs");
const OUT_DIR = path.join(__dirname, "..", "out");

const bytecode = '0x' + fs.readFileSync(path.join(OUT_DIR, "sandwich.bytecode"), {
  encoding: "ascii",
});

process.stdout.write(ethers.utils.defaultAbiCoder.encode(["bytes"], [bytecode]))
```

With that, we can now setup [dapp.tools](https://dapp.tools) with FFI to:

- Compile Yul+ contracts:

```javascript
// compile yul+ contracts
function compileYulp() internal {
    string[] memory cmds = new string[](2);
    cmds[0] = "node";
    cmds[1] = "scripts/compile.js";

    hevm.ffi(cmds);
}
```

- Deploy contracts via bytecode:

```javascript
function deployContract(bytes memory code) internal returns (address addr) {
    assembly {
        addr := create(0, add(code, 0x20), mload(code))
        if iszero(addr) {
            revert (0, 0)
        }
    }
}

function getSandwichYulpBytecode() internal returns (bytes memory) {
    string[] memory cmds = new string[](2);
    cmds[0] = "node";
    cmds[1] = "scripts/getBytecode.js";

    bytes memory bytecode = abi.decode(hevm.ffi(cmds), (bytes));
    return bytecode;
}

function test_yulp_sandwich_deploy() public {
    bytes memory bytecode = getSandwichYulpBytecode();
    bytes memory bytecodeWithArgs = abi.encodePacked(
        bytecode,
        abi.encode(address(this))
    );

    address ysandwich = deployContract(bytecodeWithArgs);
}
```

## End Result

And bada bing bada boom, you can now leverage existing Solidity tooling onto your Yul+ development pipeline with very minimal custom scripting.

Focus on writing code, not ricing your desktop. Little known fact: Vitalik Buterin uses the default floating window manager on Ubuntu, so do what you will with that information.

And as always, follow my [twitter](https://twitter.com/libevm) for more content like this!

## Links

- [Example repository](https://github.com/libevm/subway/tree/master/contracts)
- [Twitter](https://twitter.com/libevm)