# Cairo for Solidity Developers

Cairo is a Rust-inspired language that compiles to bytecode, which runs on the Cairo Virtual Machine. The Cairo Virtual Machine is a zero-knowledge virtual machine (ZKVM) used by the Starknet blockchain for executing smart contracts. In this tutorial series, we do not assume prior experience with Rust or zero-knowledge proofs. However, this tutorial series assumes prior experience with Solidity. We expect that the reader knows how to code an ERC-20 and [ERC-721](https://rareskills.io/post/erc721) and has a conceptual idea of how dApps like Uniswap V2 work.

Cairo has been carefully designed to help Solidity developers learn the language quickly, and this series highlights their similarities so that Solidity developers can reuse the mental models gained from Solidity to quickly understand Cairo.

## Getting Started with Cairo

To get started writing Cairo smart contracts, you'll need **Scarb** (Cairo's package manager and build tool) and **Starknet Foundry** (a toolchain for developing, deploying, and testing Cairo smart contracts).

### Installation

The easiest way to install these tools is using `starkup`, which automatically installs Scarb, Starknet Foundry, and the Cairo compiler in one command.

However, at the time of writing, `starkup` installs Scarb version `2.11.4` and snforge version `0.48.1`.

**For this tutorial series, we need Scarb `v2.12.0` and snforge `v0.48.0`** to avoid compatibility issues, as some syntax in this series may cause other versions to behave differently or fail to compile certain examples.

To use these specific versions, we recommend installing via `asdf` because it allows precise version control and the ability to manage multiple versions of different programming tools. If you don't have `asdf` installed, follow the [installation guide](https://asdf-vm.com/guide/getting-started.html).

***Note: This guide uses `asdf` v0.16.0 or newer. If you're using an older version, we recommend updating to the latest version.***

**Set up Scarb version `v2.12.0`:**

- Add the Scarb plugin to asdf:

```bash
asdf plugin add scarb
```

- Install Scarb version 2.12.0:

```bash
asdf install scarb 2.12.0
```

- Set the version for 2.12.0:

```bash
# Set globally for all projects
asdf set scarb 2.12.0
```

**Setup Starknet Foundry `v0.48.0`:**

- Add the Starknet Foundry plugin to asdf:

```bash
asdf plugin add starknet-foundry
```

- Install version 0.48.0:

```bash
asdf install starknet-foundry 0.48.0
```

- Set the version for 0.48.0:

```bash
# Set globally for all projects
asdf set starknet-foundry 0.48.0
```

- Restart your shell for the changes to take effect:

```rust
exec zsh  # or exec bash if using bash
```

Verify the installations:

```bash
scarb --version
snforge --version
```

![A terminal showing the scarb and snforge version commands being run](https://r2media.rareskills.io/CairoInstall/image6.png)

### Creating your first project

Create an empty directory with lowercase letters and underscores only (e.g., `hello_world`). Avoid capital letters or dashes (-), as Scarb package names must follow snake_case convention. Navigate into the directory, then run:

```bash
scarb init
```

When prompted, select `Starknet Foundry (default)` from the list of options.

![A terminal showing Starknet Foundry being selected](https://r2media.rareskills.io/CairoInstall/image1.png)

This creates a simple Cairo contract that stores and updates a single `balance` value (similar to Foundry's default Counter contract for Solidity).

### Project structure

When you open the project in your code editor, you'll see the following project structure:

```rust
hello_world/
├── src/
│   └── lib.cairo           # Your main contract code
├── tests/
│   └── test_contract.cairo # Test files go here
├── Scarb.toml              # Project configuration and dependencies
├── Scarb.lock              # Lock file for exact dependency versions
├── snfoundry.toml          # Starknet Foundry configuration
└── .gitignore              # Git ignore file
```

- `src/` is where contract files lives. `lib.cairo` is the main entry point; by default, Scarb generates a simple `HelloStarknet` contract for managing balance.
- `tests/` contains test files to verify contract functionality.
- `Scarb.toml` defines project dependencies, Cairo compiler version, package metadata, and build settings (similar to `package.json` in Node.js or `Cargo.toml` in Rust). This is where you manage what libraries your contract uses.
- `Scarb.lock` records exact dependency versions.
- `snfoundry.toml` configures Starknet Foundry settings: RPC endpoints, account configurations, and test execution options. While `Scarb.toml` manages your project and dependencies, `snfoundry.toml` configures the Foundry tooling.

## Setting up syntax highlighting

If you use VS Code or a fork of it, install the [Cairo 1.0 extension](https://marketplace.visualstudio.com/items?itemName=starkware.cairo1) for syntax highlighting. Once installed, VS Code will recognize `.cairo` files and provide features like autocomplete and error highlighting. Please note that fake VSCode extensions are a common social engineering tactic, so double-check the publisher.

![an image showing searching for Cairo language extension.](https://r2media.rareskills.io/CairoInstall/image5.png)

Open `src/lib.cairo` to see the code generated by Scarb. We will explain the syntax in the next chapter.

![The default code created by scarb](https://r2media.rareskills.io/CairoInstall/image2.png)

To compile your contract, run:

```rust
scarb build
```

This compiles your Cairo code and generates the compiled contract files in the `target/` directory. These files are what you'll deploy to Starknet.

You can then test the project with:

```bash
scarb test
```

## Cairo concepts similar to Solidity

Cairo smart contracts have a concept of “storage variables” that supports the types that Solidity developers are accustomed to, such as integers, strings, mappings, arrays, booleans, and so forth.

The following concepts have 1-1 or nearly 1-1 analogs to Solidity:

- storage variables and storage slots
- emitting events
- public, internal, and view functions
- require statements
- `msg.sender`, `block.timestamp`, and `block.number`
- constructor
- interfaces for declaring external functions
- contracts can call other contracts and use an ABI to know how to call the other contracts
- contracts can create other contracts
- transactions cost “gas” to disincentivize spam
- OpenZeppelin serves as the defacto “standard library” for the language

## Key differences from Solidity

Cairo contracts have the following capabilities and/or differences compared to Solidity:

- Cairo supports in-memory hashmaps (Solidity only supports storage mappings)
- Solidity in-memory arrays must have a defined size at the time they are declared, but Cairo does not have this restriction
- Cairo has a much more expressive control flow syntax that it inherits from Rust (such as pattern matching)
- Like Rust, Cairo is not object-oriented and therefore does not support inheritance. However, Cairo provides other ways to compose code together
- Solidity contracts upgrade through proxy patterns; however, Cairo contracts can upgrade their bytecode while keeping the storage intact
- There is no “native token” in Cairo and hence no `msg.value`. By default, gas is paid using the STRK token, which is an ERC-20 token. You can see the token on the [explorer here](https://voyager.online/contract/0x04718f5a0Fc34cC1AF16A1cdee98fFB20C31f5cD61D6Ab07201858f4287c938D).
- Starknet has account abstraction built into the protocol, so there is no such thing as an "Externally Owned Address (EOA)”

The last point may cause some confusion for developers coming from EVM compatible chains, but don’t worry, we will explore this in great detail later.

## Account Creation on Starknet

To understand the lifecycle of account creation on Starknet, you can create a wallet using [Ready](https://www.ready.co/) (formerly Argent) or [Braavos](https://braavos.app/download-braavos-wallet/) wallet. The video below shows the process of creating a Ready wallet using the browser extension.

<video src="https://r2media.rareskills.io/CairoInstall/video1.mp4" type="video/mp4" autoplay loop muted controls></video>

After installing the wallet, note that it defaults to mainnet. For this tutorial, switch to Sepolia testnet by clicking the network selector at the top of your Ready wallet and choosing "Sepolia". Then create a new account on Sepolia following the process shown in the video below:

<video src="https://r2media.rareskills.io/CairoInstall/video2.mp4" type="video/mp4" autoplay loop muted controls></video>

Next, copy your wallet address and paste it into a Starknet block explorer. You can use either [Starkscan](https://starkscan.co/) or [Voyager](https://voyager.online/). Make sure to switch the explorer to Sepolia testnet as well (at the top right of the page).

Since this is a newly created wallet with no transaction history, the explorer won't display any results yet.

![A wallet with no transaction history](https://r2media.rareskills.io/CairoInstall/image7.png)

To start transacting, we need STRK tokens to cover gas fees. Go to the [Starknet Faucet](https://faucet.starknet.io/), paste your wallet address, and request testnet tokens.

### Initializing an account method 1: Send a transaction

A Starknet account is initialized by sending its first transaction. Let’s send 1 STRK to ourselves from the wallet:

<video src="https://r2media.rareskills.io/CairoInstall/video3.mp4" type="video/mp4" autoplay loop muted controls></video>

Now, paste your address into the explorer search again. You'll see a contract has been deployed at your wallet address. This first transaction automatically deployed your account contract, as shown in the image below:

![A deploycontract transaction shown in the explorer](https://r2media.rareskills.io/CairoInstall/image3.png)

### Initializing an account method 2: Deploy the smart wallet directly

Alternatively, you can deploy the account contract directly. After receiving STRK tokens in your wallet, you'll see an option to deploy the account contract. The [GIF from Ready](https://support.argent.xyz/hc/en-us/articles/8802319054237-How-to-activate-deploy-my-Argent-Starknet-wallet) below illustrates this on mainnet (the process is the same for Sepolia):

![wallet initialization gif](https://r2media.rareskills.io/CairoInstall/image4.gif)

## Starknet does not have EOAs

In Starknet, there are no EOAs. Every address has bytecode and storage.

So then how do we distinguish between a smart contract (such as a DeFi app) and a contract intended to hold “our funds”?

Contracts that are intended to serve as wallets must implement a particular trait (interface) known as [SNIP-6](https://github.com/starknet-io/SNIPs/blob/main/SNIPS/snip-6.md). We will discuss this in detail in a later tutorial, but for now it suffices to say that an SNIP-6 contract must implement an `__execute__` function, which is how it receives its instructions from a user.

Here is the distinction:

In Ethereum, a private key is used to derive an Ethereum address, which other smart contracts can refer to as an account.

In Starknet, private keys do not turn into an address. Rather, they are used for authentication while calling the `__execute__` command.

This is another important distinction between Ethereum and Starknet. In Ethereum, the runtime validates the cryptographic signature of the wallet’s transaction. In Starknet, the wallet with the `__execute__` function can use any authentication method it wants — this means Starknet can support features like passkeys and quantum-secure cryptography authentication methods without a hard fork. All that is required is programming the smart contract account to support the desired cryptographic algorithm.

The "Starknet address" shown in your wallet is the smart contract address ("counterfactual address" if the contract doesn't exist yet). When the wallet app (the off-chain software on your device) generates your seed phrase, it derives a private key and predicts where the contract will be deployed later. The wallet displays this predicted address to you as your "address" for convenience, even though the contract doesn't exist on-chain until you initialize it.

It is not possible for your “wallet” (the off-chain software) to “own” anything on Starknet. Unlike Ethereum where signatures prove ownership directly, Starknet requires an on-chain smart contract to hold assets. Your wallet app simply creates signatures that your on-chain wallet contract verifies before executing transactions (such as sending tokens or calling other contracts). We will explore this concept in greater detail in a later chapter on account abstraction.

## Beware of older Cairo versions

At the time of writing, the current Cairo version is `2.12.0`. Be aware that internet searches and LLM queries often return code written for Cairo 1.x or earlier versions (0.x), which are incompatible with Cairo 2.x. The syntax has changed significantly between versions, so code from older versions will not work. Always check the Cairo version when copying code examples.

## Prompting techniques if you get stuck

Like other languages designed for blockchain, Cairo doesn’t have as significant a web presence as languages like JavaScript or Python.

- To fix compilation issues, ask how to resolve the issue in Rust. 80% of the compilation issues you will encounter would likely be the same compilation issue if you wrote the code in Rust.
- Scarb is very similar to Rust cargo. If you encounter issues with Scarb, do an internet search with your error message for “cargo” instead of Scarb to increase your chances of finding a relevant solution.

In the next chapter, we will write our first Cairo program.

*This article is part of a tutorial series on [Cairo Programming on Starknet](https://rareskills.io/cairo-tutorial)*
