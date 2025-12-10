# ERC-20 Token on Starknet

ERC-20 tokens on Starknet work the same way as on Ethereum. In fact, **STRK** (Starknet's fee token) is itself an ERC-20 token; there's no special "*native*" token at the protocol level.

Both [ETH](https://voyager.online/token/0x049D36570D4e46f48e99674bd3fcc84644DdD6b96F7C741B1562B82f9e004dC7) and [STRK](https://voyager.online/token/0x04718f5a0Fc34cC1AF16A1cdee98fFB20C31f5cD61D6Ab07201858f4287c938D) on Starknet exist as standard ERC-20 contracts, just like any other token one creates.

In this tutorial, you'll learn how to build and test an ERC-20 token contract on Starknet. The tutorial assumes familiarity with the ERC-20 standard but explains each implementation step and Cairo syntax along the way.

The preferred way to create ERC-20 tokens is to use the OpenZeppelin library. This will be covered in an upcoming tutorial on “Components.” The purpose of this tutorial is to tie together everything we have learned previously.

## Project Setup

Create a new scarb project and navigate into the directory:

```bash
scarb new erc20
cd erc20
```

## Contract Interface

The ERC-20 interface defines the blueprint that every fungible token must follow. It specifies the required functions for checking token balances, transferring tokens, managing spending permissions, and retrieving token metadata.

All ERC-20 tokens on Starknet implement the following interface in Cairo:

```rust
use starknet::{ContractAddress};

#[starknet::interface]
pub trait IERC20<TContractState> {
    fn total_supply(self: @TContractState) -> u256;
    fn balance_of(self: @TContractState, account: ContractAddress) -> u256;
    fn allowance(self: @TContractState, owner: ContractAddress, spender: ContractAddress) -> u256;
    fn transfer(ref self: TContractState, recipient: ContractAddress, amount: u256) -> bool;
    fn transfer_from(ref self: TContractState, sender: ContractAddress, recipient: ContractAddress, amount: u256) -> bool;
    fn approve(ref self: TContractState, spender: ContractAddress, amount: u256) -> bool;
    fn name(self: @TContractState) -> ByteArray;
    fn symbol(self: @TContractState) -> ByteArray;
    fn decimals(self: @TContractState) -> u8;
}
```

This interface mirrors Ethereum’s ERC-20 standard but uses Cairo-specific syntax and conventions. In the next section, we’ll look at how it differs from Solidity.

### How Cairo ERC-20 Interface Syntax Differs from Solidity

**State References**: Notice that in the Cairo `IERC20` interface above, the `self: @TContractState` for view functions and `ref self: TContractState` for functions that change state. The `@` symbol creates a read-only snapshot of the contract state, while `ref` allows state modifications. For example, checking *STRK* balance uses `@` (view-only), but transferring *STRK* uses `ref` (modifies balances).

The `<TContractState>` is a generic type that lets this same interface work with any ERC-20 contract's storage layout.

**Types**: Cairo uses `u256` for token amounts (similar to Solidity's `uint256`) and `ContractAddress` instead of Ethereum's `address` type. The token interface’s `name` and `symbol` functions return a `ByteArray` rather than a string

These functions implement the same balances, transfers, allowances, and metadata as Ethereum’s ERC-20 standard, differing only in syntax.

# Building the ERC-20 Token Contract

We'll build the ERC-20 contract step by step in Cairo, starting with the basic structure and gradually adding functionality to it, while testing the major functions as we go.

In the `src/lib.cairo` file, create an empty contract module and interface that we'll build upon:

```rust
#[starknet::interface]
pub trait IERC20<TContractState> {
    // We'll add functions here as we implement them
}

#[starknet::contract]
pub mod ERC20 {}
```

## Storage Setup

Next, we'll define the storage variables that will hold balances, allowances, metadata, and ownership data. We'll import `ContractAddress` from Starknet for address types and `Map` for Cairo's version of storage mappings. The storage variables will track:

- `balances`: how many tokens each address owns
- `allowances`: how much each address can spend from another address's balance
- `token_name`, `symbol`, `decimal` are standard ERC-20 metadata
- `total_supply`: total tokens in circulation
- `owner`: the contract owner address

```rust
#[starknet::interface]
pub trait IERC20<TContractState> {
    // We'll add functions here as we implement them
}

#[starknet::contract]
pub mod ERC20 {
    use starknet::ContractAddress;
    use starknet::storage::Map;

    #[storage]
    pub struct Storage {
        // Maps each account address to their token balance
        balances: Map<ContractAddress, u256>,

        // Maps (owner, spender) pairs to approved spending amounts
        allowances: Map<(ContractAddress, ContractAddress), u256>,

        // Token metadata
        token_name: ByteArray,
        symbol: ByteArray,
        decimal: u8,

        // Total number of tokens that exist
        total_supply: u256,

        // Address that can mint new tokens
        owner: ContractAddress,
    }
}
```

Here's how the mappings compare to Solidity:

```solidity
mapping(address => uint256) balances;
mapping(address => mapping(address => uint256)) allowances;
```

Cairo uses tuples `(ContractAddress, ContractAddress)` for nested mappings instead of Solidity's nested structure:

```rust
balances: Map<ContractAddress, u256>,  // owner -> amount
allowances: Map<(ContractAddress, ContractAddress), u256>, // (owner, spender) -> amount
```

We'll create a "Rare Token" with symbol "RST". Since the name, symbol, and decimals typically don't change, we'll set them in the constructor. We also import  `StoragePointerWriteAccess` to enable write access to storage:

```rust
#[starknet::interface]
pub trait IERC20<TContractState> {
    // We'll add functions here as we implement them
}

#[starknet::contract]
pub mod ERC20 {
    use starknet::ContractAddress;
    use starknet::storage::{Map, StoragePointerWriteAccess};

    #[storage]
    pub struct Storage {
        balances: Map<ContractAddress, u256>,
        allowances: Map<(ContractAddress, ContractAddress), u256>, //  (owner, spender) -> amount
        token_name: ByteArray,
        symbol: ByteArray,
        decimal: u8,
        total_supply: u256,
        owner: ContractAddress,
    }

     //NEWLY ADDED
    #[constructor]
    fn constructor(ref self: ContractState, owner: ContractAddress) {
        // Set the token's metadata
        self.token_name.write("Rare Token");
        self.symbol.write("RST");
        self.decimal.write(18);

        // Set owner
        self.owner.write(owner);  // Usually the deployer's address
    }
}
```

The constructor initializes the token with name "Rare Token", symbol "RST", 18 decimals (standard for most tokens), and the owner address. The `ref self: ContractState` parameter lets us modify the contract's storage.

> **You might wonder why we passed the owner address as a parameter instead of using `get_caller_address()` to automatically set the deployer as owner.**
This design choice is intentional and relates to how contract deployment works on Starknet.  When deploying a contract that uses `get_caller_address()` in its constructor, the Universal Deployer Contract (UDC) deploys the contract, not your account directly. Therefore, `get_caller_address()` returns the UDC's address, not your account's address. The UDC is explained in detail in the "Deploying Contracts on Starknet" chapter later in this series.
>

## Events Declaration

Add the following events after the storage section to track transfers and approvals:

```rust
// Define the events that this contract can emit
#[event]
#[derive(Drop, starknet::Event)]
pub enum Event {
    Transfer: Transfer,    // Emitted when tokens are transferred
    Approval: Approval,    // Emitted when spending approval is granted
}

// Event emitted whenever tokens are transferred between addresses
#[derive(Drop, starknet::Event)]
pub struct Transfer {
    #[key]  // Indexed field (can be filtered when querying events)
    from: ContractAddress,     // Address sending the tokens
    #[key]  // Indexed field (can be filtered when querying events)
    to: ContractAddress,       // Address receiving the tokens
    amount: u256,              // Number of tokens transferred
}

// Event emitted when an owner approves a spender to use their tokens
#[derive(Drop, starknet::Event)]
pub struct Approval {
    #[key]  // Indexed field (can be filtered when querying events)
    owner: ContractAddress,    // Address that owns the tokens
    #[key]  // Indexed field (can be filtered when querying events)
    spender: ContractAddress,  // Address approved to spend the tokens
    value: u256,               // Amount approved for spending
}
```

The `Event` enum contains all events the contract can emit: `Transfer` and `Approval`.

- `Transfer` events track token movements with `from` and `to` addresses plus the `amount`.
- `Approval` events also track spending permissions with the `owner` who grants permission, the `spender` who receives it, and the approved `value`.

The parameters are indexed so we can easily query transfers from specific addresses or approvals for particular owners.

## Contract Implementation

Now let's implement the contract functions. Since external contracts and users need to interact with our ERC20 token, we need to make our implementation callable from outside the contract. We do this by adding the `#[abi(embed_v0)]` attribute, which embeds the implementation into the contract's ABI:

```rust
#[abi(embed_v0)]
impl ERC20Impl of super::IERC20<ContractState> {
    // Implementation functions go here
}
```

The `ERC20Impl` implements the `IERC20` interface we defined earlier, with `ContractState` representing the contract's storage.

### Metadata Functions: `name`, `symbol`, and `decimals`

The metadata functions return basic token details such as name, symbol, and decimal precision. Let’s start by implementing the functions.

Add their function signatures to the interface:

```rust
#[starknet::interface]
pub trait IERC20<TContractState> {
    fn name(self: @TContractState) -> ByteArray;
    fn symbol(self: @TContractState) -> ByteArray;
    fn decimals(self: @TContractState) -> u8;
}
```

Then implement the functions inside the `ERC20Impl` block:

```rust
#[abi(embed_v0)]
    impl ERC20Impl of super::IERC20<ContractState> {
        // Returns the full name of the token
        fn name(self: @ContractState) -> ByteArray {
            self.token_name.read()
        }

        // Returns the token's symbol/ticker
        fn symbol(self: @ContractState) -> ByteArray {
            self.symbol.read()
        }

        // Returns the number of decimal places for the token
        fn decimals(self: @ContractState) -> u8 {
            self.decimal.read()
        }

        // other functions goes here
    }
```

Each function reads a stored value set in the constructor during contract initialization. Update the imports inside the contract module to include `StoragePointerReadAccess`  to enable these reads:

```rust
use starknet::storage::{
    Map, StoragePointerWriteAccess, StoragePointerReadAccess
};
```

Here's the complete code up to this point:

```rust
#[starknet::interface]
pub trait IERC20<TContractState> {
    fn name(self: @TContractState) -> ByteArray;
    fn symbol(self: @TContractState) -> ByteArray;
    fn decimals(self: @TContractState) -> u8;
}

#[starknet::contract]
pub mod ERC20 {
    use starknet::ContractAddress;
    use starknet::storage::{Map, StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    pub struct Storage {
        balances: Map<ContractAddress, u256>,
        allowances: Map<(ContractAddress, ContractAddress), u256>,
        token_name: ByteArray,
        symbol: ByteArray,
        decimal: u8,
        total_supply: u256,
        owner: ContractAddress,
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    pub enum Event {
        Transfer: Transfer,
        Approval: Approval,
    }

    #[derive(Drop, starknet::Event)]
    pub struct Transfer {
        #[key]
        from: ContractAddress,
        #[key]
        to: ContractAddress,
        amount: u256,
    }

    #[derive(Drop, starknet::Event)]
    pub struct Approval {
        #[key]
        owner: ContractAddress,
        #[key]
        spender: ContractAddress,
        value: u256,
    }

    #[constructor]
    fn constructor(ref self: ContractState, owner: ContractAddress) {
        self.token_name.write("Rare Token");
        self.symbol.write("RST");
        self.decimal.write(18);
        self.owner.write(owner);
    }

    #[abi(embed_v0)]
    impl ERC20Impl of super::IERC20<ContractState> {
        // Returns the full name of the token
        fn name(self: @ContractState) -> ByteArray {
            self.token_name.read()
        }

        // Returns the token's symbol/ticker
        fn symbol(self: @ContractState) -> ByteArray {
            self.symbol.read()
        }

        // Returns the number of decimal places for the token
        fn decimals(self: @ContractState) -> u8 {
            self.decimal.read()
        }
    }
}
```

## Test Setup

Navigate to `test/test_contract.cairo` in your project directory. Clear the boilerplate tests, leaving only the basic imports:

```rust
use starknet::ContractAddress;
use snforge_std::{declare, ContractClassTrait, DeclareResultTrait};
```

The contract constructor requires an owner address as a parameter:

```rust
#[constructor]
fn constructor(ref self: ContractState, owner: ContractAddress) {
    self.token_name.write("Rare Token");
    self.symbol.write("RST");
    self.decimal.write(18);
    self.owner.write(owner);
}
```

Since the constructor expects an owner address, we need to provide one when deploying the contract in our tests. To handle this, create a `deploy_contract` helper function that takes the owner address as a parameter and passes it to the constructor.

Also, import the dispatcher in the test to interact with the deployed contract, so altogether we have:

```rust
use starknet::ContractAddress;
use snforge_std::{declare, ContractClassTrait, DeclareResultTrait};

//NEWLY ADDED BELOW//
use erc20::IERC20Dispatcher;
use erc20::IERC20DispatcherTrait;

// Helper function to deploy the ERC20 contract with a specified owner
fn deploy_contract(name: ByteArray, owner: ContractAddress) -> ContractAddress {
    let contract = declare(name).unwrap().contract_class(); // Declare the contract class
    let constructor_args = array![owner.into()];  // Pass owner to constructor
    let (contract_address, _) = contract.deploy(@constructor_args).unwrap(); // Deploy the contract and return its address
    contract_address
}
```

The `IERC20Dispatcher` and `IERC20DispatcherTrait` dispatchers allow us to call contract functions from the tests.

The `deploy_contract` function declares the contract class, passes the owner address to the constructor via `constructor_args`, and returns the deployed contract's address for us to interact with.

Since each test requires a contract deployment, define an `OWNER` constant to provide a consistent test owner address instead of creating a new one each time:

```rust
const OWNER: ContractAddress = 'OWNER'.try_into().unwrap();
```

This way, each test can simply call `deploy_contract("ERC20", OWNER)` to deploy the contract with a consistent owner address.

### Testing Constructor Initialization

The next step is to confirm that the constructor initializes metadata correctly. The test below deploys the contract, calls its metadata functions (`name()`, `symbol()`, `decimal()`), and checks the returned values:

```rust
use starknet::ContractAddress;

use snforge_std::{declare, ContractClassTrait, DeclareResultTrait};

use erc20::IERC20Dispatcher;
use erc20::IERC20DispatcherTrait;

fn deploy_contract(name: ByteArray, owner: ContractAddress) -> ContractAddress {
    let contract = declare(name).unwrap().contract_class();
    let constructor_args = array![owner.into()];
    let (contract_address, _) = contract.deploy(@constructor_args).unwrap();
    contract_address
}

const OWNER: ContractAddress = 'OWNER'.try_into().unwrap();

// NEWLY ADDED BELOW
#[test]
fn test_token_constructor() {
    // Deploy the ERC20 contract with OWNER as the owner
    let contract_address = deploy_contract("ERC20", OWNER);

    // Create a dispatcher to interact with the deployed contract
    let erc20_token = IERC20Dispatcher { contract_address };

    // Retrieve token metadata from the contract
    let token_name = erc20_token.name();
    let token_symbol = erc20_token.symbol();
    let token_decimal = erc20_token.decimals();

    // Verify that the constructor set the correct values
    assert(token_name == "Rare Token", 'Wrong token name');
    assert(token_symbol == "RST", 'Wrong token symbol');
    assert(token_decimal == 18, 'Wrong token decimal');
}
```

After deploying the contract in `test_token_constructor`, we create an `IERC20Dispatcher` instance with the deployed contract's address to interact with the contract. Then call each metadata function and assert that the token name, symbol, and decimals match what was set in the constructor. If any value doesn't match, the test will fail with the corresponding error message.

Run `scarb test test_token_constructor` to confirm that the test passes. You can also test with incorrect values to see the expected errors.

### Implementing `total_supply`

To track how many tokens exist, we’ll include a total supply function in the interface and implement it to read and return the total number of tokens created.

Add the function signature to the interface:

```rust
#[starknet::interface]
pub trait IERC20<TContractState> {
    fn name(self: @TContractState) -> ByteArray;
    fn symbol(self: @TContractState) -> ByteArray;
    fn decimals(self: @TContractState) -> u8;

    //NEWLY ADDED
    fn total_supply(self: @TContractState) -> u256;
}
```

Then implement it in the contract:

```rust
#[abi(embed_v0)]
    impl ERC20Impl of super::IERC20<ContractState> {
        // ...previous functions....

        fn total_supply(self: @ContractState) -> u256 {
            // Read the total_supply value from contract storage
            self.total_supply.read()
        }
    }
```

To test the `total_supply` function, we need to first mint tokens and then confirm the total supply reflects the minted amount. As such, we need to implement the function to mint tokens.

### Implementing `mint`

Without `mint`, no tokens can exist since all balances start at zero.

The `mint` function isn't in the [ERC-20 specification](https://eips.ethereum.org/EIPS/eip-20), but it's needed to create tokens and increase the total supply.

Add it to the interface:

```rust
use starknet::ContractAddress;

#[starknet::interface]
pub trait IERC20<TContractState> {
    fn name(self: @TContractState) -> ByteArray;
    fn symbol(self: @TContractState) -> ByteArray;
    fn decimals(self: @TContractState) -> u8;
    fn total_supply(self: @TContractState) -> u256;

    //NEWLY ADDED
    fn mint(ref self: TContractState, recipient: ContractAddress, amount: u256) -> bool;
}
```

We import `ContractAddress` since the `mint` function uses it as a parameter type.

Then implement the mint function in the contract:

```rust
 #[abi(embed_v0)]
    impl ERC20Impl of super::IERC20<ContractState> {
        // ....previous functions.....//

        fn mint(ref self: ContractState, recipient: ContractAddress, amount: u256) -> bool {
            // Get the address of whoever is calling this function
            let caller = get_caller_address();

            // Only the contract owner is allowed to mint new tokens
            assert(caller == self.owner.read(), 'Call not owner');

            // Read current values before updating
            let previous_total_supply = self.total_supply.read();
            let previous_balance = self.balances.entry(recipient).read();

            // Increase total supply by the minted amount
            self.total_supply.write(previous_total_supply + amount);

            // Add the minted tokens to recipient's balance
            self.balances.entry(recipient).write(previous_balance + amount);

            // Emit transfer from zero address
            let zero_address: ContractAddress = 0.try_into().unwrap();
            self.emit(Transfer { from: zero_address, to: recipient, amount });

            true // Return success
        }
    }
```

`mint` takes a recipient address and amount as parameters. Only the contract owner can call this function, which is why `caller == owner` is checked.

When tokens are minted, both the total supply and the recipient's balance increase by the specified amount.

Recall from Solidity that newly minted tokens always appear as transfers from the zero address because they're created from nothing. We follow the same pattern here, emitting a `Transfer` event from the zero address to the recipient.

```rust
 let zero_address: ContractAddress = 0.try_into().unwrap();
 self.emit(Transfer { from: zero_address, to: recipient, amount });
```

We import `StoragePathEntry` because we use `.entry()` to access Map keys, which creates a path to specific mapping entries, and also `get_caller_address` to get the address of the current caller.

Update the imports:

```rust
use starknet::storage::{
    Map, StoragePathEntry, StoragePointerReadAccess, StoragePointerWriteAccess,
};
use starknet::{ContractAddress, get_caller_address};
```

Here's the complete code up to this point:

```rust
use starknet::ContractAddress;

#[starknet::interface]
pub trait IERC20<TContractState> {
    fn name(self: @TContractState) -> ByteArray;
    fn symbol(self: @TContractState) -> ByteArray;
    fn decimals(self: @TContractState) -> u8;

    fn total_supply(self: @TContractState) -> u256;
    fn mint(ref self: TContractState, recipient: ContractAddress, amount: u256) -> bool;
}

#[starknet::contract]
pub mod ERC20 {
    use starknet::storage::{
        Map, StoragePathEntry, StoragePointerReadAccess, StoragePointerWriteAccess,
    };
    use starknet::{ContractAddress, get_caller_address};

    #[storage]
    pub struct Storage {
        balances: Map<ContractAddress, u256>,
        allowances: Map<(ContractAddress, ContractAddress), u256>,
        token_name: ByteArray,
        symbol: ByteArray,
        decimal: u8,
        total_supply: u256,
        owner: ContractAddress,
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    pub enum Event {
        Transfer: Transfer,
        Approval: Approval,
    }

    #[derive(Drop, starknet::Event)]
    pub struct Transfer {
        #[key]
        from: ContractAddress,
        #[key]
        to: ContractAddress,
        amount: u256,
    }

    #[derive(Drop, starknet::Event)]
    pub struct Approval {
        #[key]
        owner: ContractAddress,
        #[key]
        spender: ContractAddress,
        value: u256,
    }

    #[constructor]
    fn constructor(ref self: ContractState, owner: ContractAddress) {
        self.token_name.write("Rare Token");
        self.symbol.write("RST");
        self.decimal.write(18);
        self.owner.write(owner);
    }

    #[abi(embed_v0)]
    impl ERC20Impl of super::IERC20<ContractState> {
        fn name(self: @ContractState) -> ByteArray {
            self.token_name.read()
        }

        fn symbol(self: @ContractState) -> ByteArray {
            self.symbol.read()
        }

        fn decimals(self: @ContractState) -> u8 {
            self.decimal.read()
        }

        fn total_supply(self: @ContractState) -> u256 {
            self.total_supply.read()
        }

        fn mint(ref self: ContractState, recipient: ContractAddress, amount: u256) -> bool {
            let caller = get_caller_address();

            assert(caller == self.owner.read(), 'Call not owner');

            let previous_total_supply = self.total_supply.read();
            let previous_balance = self.balances.entry(recipient).read();

            self.total_supply.write(previous_total_supply + amount);

            self.balances.entry(recipient).write(previous_balance + amount);

            let zero_address: ContractAddress = 0.try_into().unwrap();
            self.emit(Transfer { from: zero_address, to: recipient, amount });

            true
        }
    }
}
```

### Testing `total_supply`

Since only the contract owner can mint tokens, and tests don’t run as the owner by default, the owner address needs to be impersonated.

We'll use `cheat_caller_address` to temporarily change who the contract thinks is calling it, bypassing the access control checks in the contract. Set `CheatSpan::TargetCalls(1)` to apply this cheat only to the next function call (`mint()`).

Import `cheat_caller_address` and `CheatSpan` from `snforge_std`, and add a helper function to generate a test recipient address for receiving the minted tokens, so we end up with:

```rust
use starknet::ContractAddress;
use snforge_std::{
    declare, ContractClassTrait, DeclareResultTrait,
    cheat_caller_address, CheatSpan
};
use erc20::IERC20Dispatcher;
use erc20::IERC20DispatcherTrait;

fn deploy_contract(name: ByteArray, owner: ContractAddress) -> ContractAddress {
    let contract = declare(name).unwrap().contract_class();
    let constructor_args = array![owner.into()];
    let (contract_address, _) = contract.deploy(@constructor_args).unwrap();
    contract_address
}

const OWNER: ContractAddress = 'OWNER'.try_into().unwrap();

// NEWLY ADDED
// Test recipient address constant
const TOKEN_RECIPIENT: ContractAddress = 'RECIPIENT'.try_into().unwrap();
```

Now write the test:

```rust
#[test]
fn test_total_supply() {
    // Deploy the contract
    let contract_address = deploy_contract("ERC20", OWNER);

    // Create dispatcher to interact with the contract
    let erc20_token = IERC20Dispatcher { contract_address };

    // Calculate mint amount: 1000 tokens adjusted for decimals
    let token_decimal = erc20_token.decimals();
    let mint_amount = 1000 * token_decimal.into();

    // Impersonate the owner for the next function call (mint)
    cheat_caller_address(contract_address, OWNER, CheatSpan::TargetCalls(1));
    erc20_token.mint(TOKEN_RECIPIENT, mint_amount);

    // Get the total supply
    let supply = erc20_token.total_supply();

    // Verify total supply matches the minted amount
    assert(supply == mint_amount, 'Incorrect Supply');
}
```

The `test_total_supply` test deploys the contract and calculates the mint amount by multiplying 1000 tokens by the decimal places (18). Before calling `mint`, `cheat_caller_address` sets the caller to be the owner address, allowing the mint to bypass the `assert(caller == owner)` check. After minting to a recipient address, the test retrieves the total supply and verifies that it equals the minted amount.

Add the test to `test_contract.cairo` file, then run `scarb test test_total_supply` to see that it passes.

### Implementing token transfer

The `transfer` function handles moving tokens from the caller to a recipient. First, add the function signature to the interface:

```rust
use starknet::ContractAddress;

#[starknet::interface]
pub trait IERC20<TContractState> {
    fn name(self: @TContractState) -> ByteArray;
    fn symbol(self: @TContractState) -> ByteArray;
    fn decimals(self: @TContractState) -> u8;
    fn total_supply(self: @TContractState) -> u256;
    fn mint(ref self: TContractState, recipient: ContractAddress, amount: u256) -> bool;

    //NEWLY ADDED
    fn transfer(ref self: TContractState, recipient: ContractAddress, amount: u256) -> bool;
}
```

Now, implement the `transfer` function inside the `ERC20Impl` block:

```rust
   #[abi(embed_v0)]
    impl ERC20Impl of super::IERC20<ContractState> {
        //.....previous functions.....//

        fn transfer(ref self: ContractState, recipient: ContractAddress, amount: u256) -> bool {
            // Get the address of whoever called this function
            let sender = get_caller_address();

            // Read current balances for both sender and recipient
            let sender_prev_balance = self.balances.entry(sender).read();
            let recipient_prev_balance = self.balances.entry(recipient).read();

            // Check if sender has enough tokens to transfer
            assert(sender_prev_balance >= amount, 'Insufficient amount');

            // Update balances: subtract from sender, add to recipient
            self.balances.entry(sender).write(sender_prev_balance - amount);
            self.balances.entry(recipient).write(recipient_prev_balance + amount);

            // Verify the transfer worked correctly
            assert(
                self.balances.entry(recipient).read() > recipient_prev_balance,
                'Transaction failed',
            );

            // Emit an event to log this Transfer
            self.emit(Transfer { from: sender, to: recipient, amount });

            true // Return success
        }
    }
}

```

Say Alice has 100 *RareTokens* and wants to send 30 to Bob, who has 50. The function checks if Alice has enough (100 >= 30), updates Alice's balance to 70, updates Bob's balance to 80. It then confirms Bob's balance increased and emits a `Transfer` event with `from: Alice`, `to: Bob`, and `amount: 30` to log this transaction, and returns true to signal successful completion.

To test the transfer function, the contract needs a way to check each account's balance at specific points.

### Implementing `balance_of`

Let's add `balance_of` to query token balances. Add the function signature to the interface:

```rust
use starknet::ContractAddress;

#[starknet::interface]
pub trait IERC20<TContractState> {
    fn name(self: @TContractState) -> ByteArray;
    fn symbol(self: @TContractState) -> ByteArray;
    fn decimals(self: @TContractState) -> u8;
    fn total_supply(self: @TContractState) -> u256;
    fn mint(ref self: TContractState, recipient: ContractAddress, amount: u256) -> bool;
    fn transfer(ref self: TContractState, recipient: ContractAddress, amount: u256) -> bool;

     //NEWLY ADDED
    fn balance_of(self: @TContractState, account: ContractAddress) -> u256;
}
```

Then implement it in the contract:

```rust
#[abi(embed_v0)]
impl ERC20Impl of super::IERC20<ContractState> {
    //.....previous functions.....//

    fn balance_of(self: @ContractState, account: ContractAddress) -> u256 {
        // Use .entry() to access the specific account's balance in the Map
        let balance = self.balances.entry(account).read();
        balance
    }
}
```

To check an account’s *RareToken* balance, `balance_of(account_address)` looks up the address in the balances mapping and returns the corresponding value.

### **Testing `transfer`**

To test the `transfer` function, we need tokens in an account first, then verify that transferring moves tokens correctly from sender to recipient. We'll mint tokens to the owner, then transfer some to a recipient and check both balances.

Since both minting and transferring require the owner's permission, we'll use `start_cheat_caller_address` to impersonate the owner for multiple consecutive calls until explicitly stopped with `stop_cheat_caller_address`.

Import `start_cheat_caller_address` and `stop_cheat_caller_address` from `snforge_std` alongside the other imports:

```rust
use snforge_std::{
    declare, ContractClassTrait, DeclareResultTrait,
    cheat_caller_address, CheatSpan,
    start_cheat_caller_address, stop_cheat_caller_address
};
```

Now here's the test:

```rust
#[test]
fn test_transfer() {
    // Deploy the contract
    let contract_address = deploy_contract("ERC20", OWNER);
    let erc20_token = IERC20Dispatcher { contract_address };

    // Get token decimals for proper amount calculation
    let token_decimal = erc20_token.decimals();

    // Define amounts: 10,000 tokens to mint, 5,000 to transfer
    let amount_to_mint: u256 = 10000 * token_decimal.into();
    let amount_to_transfer: u256 = 5000 * token_decimal.into();

    // Start impersonating the owner for multiple calls
    start_cheat_caller_address(contract_address, OWNER);

    // Mint tokens to the owner
    erc20_token.mint(OWNER, amount_to_mint);

    // Verify the mint was successful
    assert(erc20_token.balance_of(OWNER) == amount_to_mint, 'Incorrect minted amount');

    // Track recipient's balance before transfer
    let receiver_previous_balance = erc20_token.balance_of(TOKEN_RECIPIENT);

    // Transfer tokens from owner to recipient
    erc20_token.transfer(TOKEN_RECIPIENT, amount_to_transfer);

    // Stop impersonating the owner
    stop_cheat_caller_address(contract_address);

    // Verify sender's balance decreased correctly
    assert(erc20_token.balance_of(OWNER) < amount_to_mint, 'Sender balance not reduced');
    assert(erc20_token.balance_of(OWNER) == amount_to_mint - amount_to_transfer, 'Wrong sender balance');

    // Verify recipient's balance increased correctly
    assert(erc20_token.balance_of(TOKEN_RECIPIENT) > receiver_previous_balance, 'Recipient balance unchanged');
    assert(erc20_token.balance_of(TOKEN_RECIPIENT) == amount_to_transfer, 'Wrong recipient amount');
}
```

After deploying the contract in `test_transfer()`, the test calculates the amounts: 10,000 tokens for minting and 5,000 for the transfer. It starts impersonating the owner with `start_cheat_caller_address`, which allows minting tokens to the owner's account. Once the mint succeeds, the test records the recipient's balance before making the transfer.

The test then transfer 5,000 tokens to the recipient and stop the impersonation. The final assertions verify both sides of the transaction: that the owner's balance dropped by exactly 5,000 tokens, and the recipient's balance increased by the same amount. This confirms `transfer` correctly moves tokens between accounts.

Add the test to the `test_contract.cairo` file, then run `scarb test test_transfer` to verify that it passes.

### Testing Insufficient balance for transfer

Let's test that `transfer` properly rejects attempts to transfer more tokens than the sender owns.

Modify the `transfer` call in our test to attempt transferring 11,000 tokens instead of 5000:

```rust
 erc20_token.transfer(TOKEN_RECIPIENT, 11000 * token_decimal.into());
```

When we run `scarb test test_transfer`, the test should fail with this error:

![A test showing transfer failing due to insufficient balance](https://r2media.rareskills.io/CairoERC20/image2.png)

This confirms the contract is working correctly, it's preventing the transfer of more tokens than the sender owns, triggering the `assert(sender_prev_balance >= amount, 'Insufficient amount')` check in the `transfer` function.

> **To keep the test passing, change the amount back to `amount_to_transfer` (5,000 tokens or any amount less than or equal to the owner's balance of 10,000)**
>

**Alternative: Create a dedicated test for the failure case**

Instead of modifying the existing test, add this test using `#[should_panic]` to `test_contract.cairo`:

```rust
#[test]
#[should_panic(expected: ('Insufficient amount',))]
fn test_transfer_insufficient_balance() {
    // Deploy the contract
    let contract_address = deploy_contract("ERC20", OWNER);
    let erc20_token = IERC20Dispatcher { contract_address };

    let token_decimal = erc20_token.decimals();

    // Define amounts: only 5,000 tokens minted, but attempting to transfer 10,000
    let mint_amount: u256 = 5000 * token_decimal.into();
    let transfer_amount: u256 = 10000 * token_decimal.into();

    // Start impersonating the owner
    start_cheat_caller_address(contract_address, OWNER);

    // Mint only 5,000 tokens to the owner
    erc20_token.mint(OWNER, mint_amount);

    // Verify the mint was successful
    assert(erc20_token.balance_of(OWNER) == mint_amount, 'Mint failed');

    // Attempt to transfer more than balance (10,000 tokens when only 5,000 exist)
    // This should panic with 'Insufficient amount'
    erc20_token.transfer(TOKEN_RECIPIENT, transfer_amount);

    // Stop impersonating the owner
    stop_cheat_caller_address(contract_address);
}
```

This test verifies that the transfer fails when attempting to send more tokens than the sender owns. The owner only has 5,000 tokens but tries to transfer 10,000, triggering the `assert(sender_prev_balance >= amount, 'Insufficient amount')` check in the `transfer` function. The `#[should_panic]` attribute tells the test framework that this test is expected to panic with the specific error message `'Insufficient amount'`.

Run `scarb test test_transfer_insufficient_balance` to verify it passes.

### Implementing `allowance`

The `allowance` function checks how much one address is allowed to spend on behalf of another. Let’s add the function signature to the interface:

```rust
use starknet::ContractAddress;

#[starknet::interface]
pub trait IERC20<TContractState> {
    // ... previous functions ...

    //NEWLY ADDED
    fn allowance(self: @TContractState, owner: ContractAddress, spender: ContractAddress) -> u256;
}
```

Then implement it in the contract:

```rust
#[abi(embed_v0)]
impl ERC20Impl of super::IERC20<ContractState> {
   //....previous functions....//

   fn allowance(
       self: @ContractState, owner: ContractAddress, spender: ContractAddress,
    ) -> u256 {
        // Access the allowances Map using a tuple key (owner, spender)
        self.allowances.entry((owner, spender)).read()
    }
```

For example, to see how many *RareTokens* Bob can spend from Alice's account, you'd call `allowance(Alice, Bob)`.

### Implementing `approve`

The `approve` function grants spending permission by setting how much someone (spender) can withdraw from an account's balance (owner).

Add the function signature to the interface:

```rust
use starknet::ContractAddress;

#[starknet::interface]
pub trait IERC20<TContractState> {
    // ... previous functions ...

    //NEWLY ADDED
    fn approve(ref self: TContractState, spender: ContractAddress, amount: u256) -> bool;
}
```

Then implement it in the contract:

```rust
#[abi(embed_v0)]
impl ERC20Impl of super::IERC20<ContractState> {
    //.....previous functions.....//

    fn approve(ref self: ContractState, spender: ContractAddress, amount: u256) -> bool {
        // Get the address of whoever is giving the approval (owner)
        let caller = get_caller_address();

        // Set the allowance: how much the spender can spend on behalf of the caller (owner)
        self.allowances.entry((caller, spender)).write(amount);

        // Emit an event to log this Approval
        self.emit(Approval { owner: caller, spender, value: amount });

        true  // Return success
    }
}
```

In the line `self.allowances.entry((caller, spender)).write(amount)`, `spender` refers to the address being granted the allowance by the `caller`. The `caller` (from `get_caller_address()`) is giving permission to the `spender` to spend a certain amount of tokens from their account.

Therefore, `caller` is the owner of the tokens, and `spender` is someone who has been approved by the owner to spend a certain amount of tokens on their behalf. This creates the entry `allowances[(owner, spender)] = amount` that `transfer_from` will later check and use.

### **Testing `approve`**

Let's test that the `approve` function correctly sets the allowance and that it can be queried back:

```rust
#[test]
fn test_approve() {
    let contract_address = deploy_contract("ERC20", OWNER);
    let erc20_token = IERC20Dispatcher { contract_address };

    let token_decimal = erc20_token.decimals();
    let mint_amount: u256 = 10000 * token_decimal.into();
    let approval_amount: u256 = 5000 * token_decimal.into();

    // Start impersonating the owner
    start_cheat_caller_address(contract_address, OWNER);

    // Mint tokens to the owner first
    erc20_token.mint(OWNER, mint_amount);

    // Verify mint succeeded
    assert(erc20_token.balance_of(OWNER) == mint_amount, 'Mint failed');

    // Owner approves the recipient to spend tokens
    erc20_token.approve(TOKEN_RECIPIENT, approval_amount);

    // Stop impersonating the owner
    stop_cheat_caller_address(contract_address);

    // Verify the allowance was set
    assert(erc20_token.allowance(OWNER, TOKEN_RECIPIENT) > 0, 'Incorrect allowance');
    assert(erc20_token.allowance(OWNER, TOKEN_RECIPIENT) == approval_amount, 'Wrong allowance amount');
}
```

The test deploys the contract and defines two amounts: 10,000 tokens to mint and 5,000 tokens for the approval. Using `start_cheat_caller_address`, the test impersonates the owner for multiple consecutive calls.

First, the test mints 10,000 tokens to the owner and verifies the mint succeeded. Then, while still impersonating the owner, it calls `approve` to grant the recipient permission to spend 5,000 tokens from the owner's balance. After stopping the impersonation, the test verifies two things: first, that an allowance exists (greater than 0), and second, that the allowance amount matches exactly what was approved (5,000 tokens). These assertions confirm that `approve` correctly stores the spending permission in the `allowances` mapping.

Add the test to your `test_contract.cairo` file, then run `scarb test test_approve` to verify that it passes.

### Implementing delegated transfers: `transfer_from`

Now, let’s implement `transfer_from` which moves tokens from one address to another using pre-approved spending permissions.

Update the interface to include the function signature:

```rust
#[starknet::interface]
pub trait IERC20<TContractState> {
    // ... previous functions ...

    fn transfer_from(
        ref self: TContractState, sender: ContractAddress, recipient: ContractAddress, amount: u256,
    ) -> bool;
}
```

`transfer_from` implementation:

```rust
#[abi(embed_v0)]
impl ERC20Impl of super::IERC20<ContractState> {
    //.....previous functions.....//

    fn transfer_from(ref self: ContractState, sender: ContractAddress, recipient: ContractAddress, amount: u256) -> bool {
        // Get the address of whoever is calling this function (the spender)
        let spender = get_caller_address();

        // Read current allowance: how much the spender is allowed to spend from sender's account
        let spender_allowance = self.allowances.entry((sender, spender)).read();

        // Read current balances for both sender and recipient
        let sender_balance = self.balances.entry(sender).read();
        let recipient_balance = self.balances.entry(recipient).read();

        // Check if the transfer amount doesn't exceed the approved allowance
        assert(amount <= spender_allowance, 'amount exceeds allowance');

        // Check if sender has enough tokens to transfer
        assert(amount <= sender_balance, 'amount exceeds balance');

        // Update allowance: reduce by the amount being spent
        self.allowances.entry((sender, spender)).write(spender_allowance - amount);

        // Update balances: subtract from sender, add to recipient
        self.balances.entry(sender).write(sender_balance - amount);
        self.balances.entry(recipient).write(recipient_balance + amount);

        // Emit an event to log this Transfer
        self.emit(Transfer { from: sender, to: recipient, amount });

        true  // Return success
    }
 }
```

In the above code, `spender` (from `get_caller_address()`) executes the transfer, `sender` is the token owner, and `recipient` receives the tokens. The function checks that the `spender` has sufficient allowance by reading `allowances[(sender, spender)]`, then reduces the allowance by the transferred amount.

Without subtracting their spending, the `spender` would have unlimited spending power.

Consider this example that shows how `approve` and `transfer_from` work together:

Alice calls `approve(Bob, 50)` to let Bob spend 50 of her *RareTokens*. Then Bob can use `transfer_from(Alice, Charlie, 30)` to move 30 tokens from Alice's account to Charlie, leaving Bob with 20 remaining allowance.

This approve-then-withdraw pattern is how DeFi protocols, DEXs, and other smart contracts interact with user tokens.

### Testing `transfer_from`

`transfer_from` test requires three parties: an owner with tokens, a spender with approval, and a recipient address.

Since the spender uses their approval to move tokens from the owner's account to the recipient, both the owner and the spender needs to be impersonated at different stages in the test:

```rust
#[test]
fn test_transfer_from() {
    // Deploy the contract
    let contract_address = deploy_contract("ERC20", OWNER);
    let erc20_token = IERC20Dispatcher { contract_address };

    let token_decimal = erc20_token.decimals();

    // Define amounts: 10,000 tokens to mint, 5,000 to approve and transfer
    let mint_amount: u256 = 10000 * token_decimal.into();
    let transfer_amount: u256 = 5000 * token_decimal.into();

    // Start impersonating the owner
    start_cheat_caller_address(contract_address, OWNER);

    // Mint tokens to the owner
    erc20_token.mint(OWNER, mint_amount);

    // Verify mint succeeded
    assert(erc20_token.balance_of(OWNER) == mint_amount, 'Mint failed');

    let spender:ContractAddress = 'SPENDER'.try_into().unwrap();

    // Owner approves SPENDER to spend tokens on their behalf
    erc20_token.approve(spender, transfer_amount);

    // Stop impersonating owner
    stop_cheat_caller_address(contract_address);

    // Verify the allowance was set correctly
    assert(erc20_token.allowance(OWNER, spender) == transfer_amount, 'Approval failed');

    // Track balances before transfer
    let owner_balance_before = erc20_token.balance_of(OWNER);
    let recipient_balance_before = erc20_token.balance_of(TOKEN_RECIPIENT);
    let allowance_before = erc20_token.allowance(OWNER, spender);

    // Now impersonate the SPENDER to call transfer_from
    cheat_caller_address(contract_address, spender, CheatSpan::TargetCalls(1));
    erc20_token.transfer_from(OWNER, TOKEN_RECIPIENT, transfer_amount);

    // Verify owner's balance decreased
    assert(erc20_token.balance_of(OWNER) == owner_balance_before - transfer_amount, 'Owner balance wrong');

    // Verify recipient's balance increased
    assert(erc20_token.balance_of(TOKEN_RECIPIENT) == recipient_balance_before + transfer_amount, 'Recipient balance wrong');

    // Verify allowance decreased
    assert(erc20_token.allowance(OWNER, spender) == allowance_before - transfer_amount, 'Allowance not reduced');
}
```

`test_transfer_from` validates the complete approve-and-spend pattern. The test starts by impersonating the owner to mint 10,000 tokens and approve a spender to use 5,000 of them. After stopping the owner impersonation, it verifies the approval was set correctly.

Next, the test captures the current state: the owner's balance, recipient's balance, and the spender's allowance. Then it impersonate the spender and call `transfer_from` to move 5,000 tokens from the owner to the recipient.

```rust
// Now impersonate the SPENDER to call transfer_from
cheat_caller_address(contract_address, spender, CheatSpan::TargetCalls(1));
erc20_token.transfer_from(OWNER, TOKEN_RECIPIENT, transfer_amount);
```

The final assertions verify three updates: the owner's balance decreased by 5,000, the recipient's balance increased by 5,000, and the spender's allowance was reduced by 5,000. These checks confirm that `transfer_from` correctly handles delegated transfers and properly updates allowances.

Add the test to the `test_contract.cairo` file, then run `scarb test test_transfer_from` to verify that it passes.

### **Testing Insufficient Allowance**

Let's test that `transfer_from` properly rejects attempts to spend more than the approved amount. If a spender tries to transfer more tokens than they were approved for, the transaction should fail.

Modify the `transfer_from` call in our test to attempt transferring 6,000 tokens instead of the approved 5,000:

```rust
// Attempt to transfer more than approved (6,000 instead of 5,000)
cheat_caller_address(contract_address, spender, CheatSpan::TargetCalls(1));
erc20_token.transfer_from(OWNER, TOKEN_RECIPIENT, 6000 * token_decimal.into());
```

When we run `scarb test test_transfer_from`, the test should fail with this error:

![A test showing allowance exceeded for transfer from](https://r2media.rareskills.io/CairoERC20/image1.png)

This error confirms that the contract catches the unauthorized spending attempt. The spender was only approved for 5,000 tokens, so trying to transfer 6,000 triggers the `assert(amount <= spender_allowance, 'amount exceeds allowance')` check in the `transfer_from` function.

Change the amount back to `transfer_amount` (5,000 tokens) to keep the test passing.

**Alternative: Create a dedicated test for the failure case**

Instead of modifying the existing test, we can create a separate test that expects the failure using the `#[should_panic]` attribute:

```rust
#[test]
#[should_panic(expected: ('amount exceeds allowance',))]
fn test_transfer_from_insufficient_allowance() {
    // Deploy the contract
    let contract_address = deploy_contract("ERC20", OWNER);
    let erc20_token = IERC20Dispatcher { contract_address };

    let token_decimal = erc20_token.decimals();

    // Define amounts: 10,000 tokens to mint, 5,000 to approve
    let mint_amount: u256 = 10000 * token_decimal.into();
    let approval_amount: u256 = 5000 * token_decimal.into();

    // Start impersonating the owner
    start_cheat_caller_address(contract_address, OWNER);

    // Mint tokens to the owner
    erc20_token.mint(OWNER, mint_amount);

    let spender: ContractAddress = 'SPENDER'.try_into().unwrap();

    // Owner approves SPENDER to spend 5,000 tokens
    erc20_token.approve(spender, approval_amount);

    // Stop impersonating owner
    stop_cheat_caller_address(contract_address);

    // Attempt to transfer more than approved (6,000 instead of 5,000)
    // This should panic with 'amount exceeds allowance'
    cheat_caller_address(contract_address, spender, CheatSpan::TargetCalls(1));
    erc20_token.transfer_from(OWNER, TOKEN_RECIPIENT, 6000 * token_decimal.into());
}
```

Again, the `#[should_panic]` attribute tells the test framework that this test is expected to fail with the specific error message `'amount exceeds allowance'`. When you add this test to your `test_contract.cairo` file, then run `scarb test test_transfer_from_insufficient_allowance` this test will pass because the panic occurred as expected.

### Testing Insufficient balance

We can also test the case where a spender has sufficient allowance but the owner doesn't have enough tokens. Add this test to `test_contract.cairo`:

```rust
#[test]
#[should_panic(expected: ('amount exceeds balance',))]
fn test_transfer_from_insufficient_balance() {
    // Deploy the contract
    let contract_address = deploy_contract("ERC20", OWNER);
    let erc20_token = IERC20Dispatcher { contract_address };

    let token_decimal = erc20_token.decimals();

    // Define amounts: only 1,000 tokens minted, but 2,000 approved and attempted
    let mint_amount: u256 = 1000 * token_decimal.into();
    let approval_amount: u256 = 2000 * token_decimal.into();
    let transfer_amount: u256 = 2000 * token_decimal.into();

    // Start impersonating the owner
    start_cheat_caller_address(contract_address, OWNER);

    // Mint only 1,000 tokens to the owner
    erc20_token.mint(OWNER, mint_amount);

    let spender: ContractAddress = 'SPENDER'.try_into().unwrap();

    // Owner approves SPENDER to spend 2,000 tokens (more than balance)
    erc20_token.approve(spender, approval_amount);

    // Stop impersonating owner
    stop_cheat_caller_address(contract_address);

    // Spender has sufficient allowance but owner doesn't have enough balance
    // This should panic with 'amount exceeds balance'
    cheat_caller_address(contract_address, spender, CheatSpan::TargetCalls(1));
    erc20_token.transfer_from(OWNER, TOKEN_RECIPIENT, transfer_amount);
}
```

`test_transfer_from_insufficient_balance` test above verifies that even with sufficient allowance, the transfer fails if the token owner doesn't have enough balance. The spender is approved for 2,000 tokens, but the owner only has 1,000, triggering the `assert(amount <= sender_balance, 'amount exceeds balance')` check.

Run `scarb test test_transfer_from_insufficient_balance` to verify it passes.

Here's the complete ERC-20 contract:

```rust
use starknet::ContractAddress;

#[starknet::interface]
pub trait IERC20<TContractState> {
    fn total_supply(self: @TContractState) -> u256;
    fn balance_of(self: @TContractState, account: ContractAddress) -> u256;
    fn allowance(self: @TContractState, owner: ContractAddress, spender: ContractAddress) -> u256;
    fn transfer(ref self: TContractState, recipient: ContractAddress, amount: u256) -> bool;
    fn transfer_from(
        ref self: TContractState, sender: ContractAddress, recipient: ContractAddress, amount: u256,
    ) -> bool;
    fn approve(ref self: TContractState, spender: ContractAddress, amount: u256) -> bool;

    fn name(self: @TContractState) -> ByteArray;
    fn symbol(self: @TContractState) -> ByteArray;
    fn decimals(self: @TContractState) -> u8;

    fn mint(ref self: TContractState, recipient: ContractAddress, amount: u256) -> bool; // For testing purposes
}

#[starknet::contract]
pub mod ERC20 {
    use starknet::{ContractAddress, get_caller_address};
    use starknet::storage::{
        Map, StoragePointerWriteAccess, StoragePointerReadAccess, StoragePathEntry,
    };

    #[storage]
    pub struct Storage {
        balances: Map<ContractAddress, u256>,
        allowances: Map<
            (ContractAddress, ContractAddress), u256,
        >, //  (owner, spender) -> amount, amount>
        token_name: ByteArray,
        symbol: ByteArray,
        decimal: u8,
        total_supply: u256,
        owner: ContractAddress,
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    pub enum Event {
        Transfer: Transfer,
        Approval: Approval,
    }

    #[derive(Drop, starknet::Event)]
    pub struct Transfer {
        #[key]
        from: ContractAddress,
        #[key]
        to: ContractAddress,
        amount: u256,
    }

    #[derive(Drop, starknet::Event)]
    pub struct Approval {
        #[key]
        owner: ContractAddress,
        #[key]
        spender: ContractAddress,
        value: u256,
    }

      #[constructor]
    fn constructor(ref self: ContractState, owner: ContractAddress) {
        self.token_name.write("Rare Token");
        self.symbol.write("RST");
        self.decimal.write(18);
        self.owner.write(owner);
    }

    #[abi(embed_v0)]
    impl ERC20Impl of super::IERC20<ContractState> {
        fn total_supply(self: @ContractState) -> u256 {
            self.total_supply.read()
        }
        fn balance_of(self: @ContractState, account: ContractAddress) -> u256 {
            let balance = self.balances.entry(account).read();
            balance
        }

        fn name(self: @ContractState) -> ByteArray {
            self.token_name.read()
        }

        fn symbol(self: @ContractState) -> ByteArray {
            self.symbol.read()
        }

        fn decimals(self: @ContractState) -> u8 {
            self.decimal.read()
        }

        fn allowance(
            self: @ContractState, owner: ContractAddress, spender: ContractAddress,
        ) -> u256 {
            let allowance = self.allowances.entry((owner, spender)).read();

            allowance
        }

        fn transfer(ref self: ContractState, recipient: ContractAddress, amount: u256) -> bool {
            let sender = get_caller_address();

            let sender_prev_balance = self.balances.entry(sender).read();
            let recipient_prev_balance = self.balances.entry(recipient).read();

            assert(sender_prev_balance >= amount, 'Insufficient amount');

            self.balances.entry(sender).write(sender_prev_balance - amount);
            self.balances.entry(recipient).write(recipient_prev_balance + amount);

            assert(
                self.balances.entry(recipient).read() > recipient_prev_balance,
                'Transaction failed',
            );
            self.emit(Transfer { from: sender, to: recipient, amount });

            true
        }

        fn transfer_from(
            ref self: ContractState,
            sender: ContractAddress,
            recipient: ContractAddress,
            amount: u256,
        ) -> bool {
            let spender = get_caller_address();

            let spender_allowance = self.allowances.entry((sender, spender)).read();
            let sender_balance = self.balances.entry(sender).read();
            let recipient_balance = self.balances.entry(recipient).read();

            assert(amount <= spender_allowance, 'amount exceeds allowance');
            assert(amount <= sender_balance, 'amount exceeds balance');

            self.allowances.entry((sender, spender)).write(spender_allowance - amount);
            self.balances.entry(sender).write(sender_balance - amount);
            self.balances.entry(recipient).write(recipient_balance + amount);

            self.emit(Transfer { from: sender, to: recipient, amount });

            true
        }

        fn approve(ref self: ContractState, spender: ContractAddress, amount: u256) -> bool {
            let caller = get_caller_address();

            self.allowances.entry((caller, spender)).write(amount);

            self.emit(Approval { owner: caller, spender, value: amount });

            true
        }

        fn mint(ref self: ContractState, recipient: ContractAddress, amount: u256) -> bool {
            let caller = get_caller_address();
            assert(caller == self.owner.read(), 'Call not owner');

            let previous_total_supply = self.total_supply.read();
            let previous_balance = self.balances.entry(recipient).read();

            self.total_supply.write(previous_total_supply + amount);
            self.balances.entry(recipient).write(previous_balance + amount);

            let zero_address: ContractAddress = 0.try_into().unwrap();

            self.emit(Transfer { from: zero_address, to: recipient, amount });

            true
        }
    }
}
```

The following is the complete test:

```rust
use erc20::{IERC20Dispatcher, IERC20DispatcherTrait};
use snforge_std::{
    CheatSpan, ContractClassTrait, DeclareResultTrait, cheat_caller_address, declare,
    start_cheat_caller_address, stop_cheat_caller_address,
};
use starknet::ContractAddress;

fn deploy_contract(name: ByteArray, owner: ContractAddress) -> ContractAddress {
    let contract = declare(name).unwrap().contract_class();
    let constructor_args = array![owner.into()];
    let (contract_address, _) = contract.deploy(@constructor_args).unwrap();
    contract_address
}

const OWNER: ContractAddress = 'OWNER'.try_into().unwrap();
const TOKEN_RECIPIENT: ContractAddress = 'RECIPIENT'.try_into().unwrap();

#[test]
fn test_token_constructor() {
    let contract_address = deploy_contract("ERC20", OWNER);

    let erc20_token = IERC20Dispatcher { contract_address };

    let token_name = erc20_token.name();
    let token_symbol = erc20_token.symbol();
    let token_decimal = erc20_token.decimals();

    assert(token_name == "Rare Token", 'Wrong token name');
    assert(token_symbol == "RST", 'Wrong token symbol');
    assert(token_decimal == 18, 'Wrong token decimal');
}

#[test]
fn test_total_supply() {
    let contract_address = deploy_contract("ERC20", OWNER);

    let erc20_token = IERC20Dispatcher { contract_address };

    let token_decimal = erc20_token.decimals();
    let mint_amount = 1000 * token_decimal.into();

    // cheat caller address to be the owner
    cheat_caller_address(contract_address, OWNER, CheatSpan::TargetCalls(1));
    erc20_token.mint(TOKEN_RECIPIENT, mint_amount);

    let supply = erc20_token.total_supply();

    assert(supply == mint_amount, 'Incorrect Supply');
}

#[test]
fn test_transfer() {
    let contract_address = deploy_contract("ERC20", OWNER);
    let erc20_token = IERC20Dispatcher { contract_address };

    let token_decimal = erc20_token.decimals();

    let amount_to_mint: u256 = 10000 * token_decimal.into();
    let amount_to_transfer: u256 = 5000 * token_decimal.into();

    // Start impersonating the owner for multiple calls
    start_cheat_caller_address(contract_address, OWNER);

    erc20_token.mint(OWNER, amount_to_mint);

    assert(erc20_token.balance_of(OWNER) == amount_to_mint, 'Incorrect minted amount');

    let receiver_previous_balance = erc20_token.balance_of(TOKEN_RECIPIENT);
    erc20_token.transfer(TOKEN_RECIPIENT, amount_to_transfer);

    stop_cheat_caller_address(contract_address);

    assert(erc20_token.balance_of(OWNER) < amount_to_mint, 'Sender balance not reduced');
    assert(
        erc20_token.balance_of(OWNER) == amount_to_mint - amount_to_transfer,
        'Wrong sender balance',
    );

    assert(
        erc20_token.balance_of(TOKEN_RECIPIENT) > receiver_previous_balance,
        'Recipient balance unchanged',
    );
    assert(
        erc20_token.balance_of(TOKEN_RECIPIENT) == amount_to_transfer, 'Wrong recipient amount',
    );
}

#[test]
#[should_panic(expected: ('Insufficient amount',))]
fn test_transfer_insufficient_balance() {
    // Deploy the contract
    let contract_address = deploy_contract("ERC20", OWNER);
    let erc20_token = IERC20Dispatcher { contract_address };

    let token_decimal = erc20_token.decimals();

    // Define amounts: only 5,000 tokens minted, but attempting to transfer 10,000
    let mint_amount: u256 = 5000 * token_decimal.into();
    let transfer_amount: u256 = 10000 * token_decimal.into();

    // Start impersonating the owner
    start_cheat_caller_address(contract_address, OWNER);

    // Mint only 5,000 tokens to the owner
    erc20_token.mint(OWNER, mint_amount);

    // Verify the mint was successful
    assert(erc20_token.balance_of(OWNER) == mint_amount, 'Mint failed');

    // Attempt to transfer more than balance (10,000 tokens when only 5,000 exist)
    // This should panic with 'Insufficient amount'
    erc20_token.transfer(TOKEN_RECIPIENT, transfer_amount);

    // Stop impersonating the owner
    stop_cheat_caller_address(contract_address);
}

#[test]
fn test_approve() {
    let contract_address = deploy_contract("ERC20", OWNER);
    let erc20_token = IERC20Dispatcher { contract_address };

    let token_decimal = erc20_token.decimals();
    let mint_amount: u256 = 10000 * token_decimal.into();
    let approval_amount: u256 = 5000 * token_decimal.into();

    // Start impersonating the owner
    start_cheat_caller_address(contract_address, OWNER);

    // Mint tokens to the owner first
    erc20_token.mint(OWNER, mint_amount);

    // Verify mint succeeded
    assert(erc20_token.balance_of(OWNER) == mint_amount, 'Mint failed');

    // Owner approves the recipient to spend tokens
    erc20_token.approve(TOKEN_RECIPIENT, approval_amount);

    // Stop impersonating the owner
    stop_cheat_caller_address(contract_address);

    // Verify the allowance was set
    assert(erc20_token.allowance(OWNER, TOKEN_RECIPIENT) > 0, 'Incorrect allowance');
    assert(erc20_token.allowance(OWNER, TOKEN_RECIPIENT) == approval_amount, 'Wrong allowance amount');
}

#[test]
fn test_transfer_from() {
    let contract_address = deploy_contract("ERC20", OWNER);
    let erc20_token = IERC20Dispatcher { contract_address };

    let token_decimal = erc20_token.decimals();
    let mint_amount: u256 = 10000 * token_decimal.into();
    let transfer_amount: u256 = 5000 * token_decimal.into();

    start_cheat_caller_address(contract_address, OWNER);

    erc20_token.mint(OWNER, mint_amount);

    assert(erc20_token.balance_of(OWNER) == mint_amount, 'Mint failed');

    let spender: ContractAddress = 'SPENDER'.try_into().unwrap();
    erc20_token.approve(spender, transfer_amount);

    stop_cheat_caller_address(contract_address);

    assert(erc20_token.allowance(OWNER, spender) == transfer_amount, 'Approval failed');

    let owner_balance_before = erc20_token.balance_of(OWNER);
    let recipient_balance_before = erc20_token.balance_of(TOKEN_RECIPIENT);
    let allowance_before = erc20_token.allowance(OWNER, spender);

    cheat_caller_address(contract_address, spender, CheatSpan::TargetCalls(1));
    erc20_token.transfer_from(OWNER, TOKEN_RECIPIENT, 5000 * token_decimal.into());

    assert(
        erc20_token.balance_of(OWNER) == owner_balance_before - transfer_amount,
        'Owner balance wrong',
    );

    assert(
        erc20_token.balance_of(TOKEN_RECIPIENT) == recipient_balance_before + transfer_amount,
        'Recipient balance wrong',
    );

    assert(
        erc20_token.allowance(OWNER, spender) == allowance_before - transfer_amount,
        'Allowance not reduced',
    );
}

#[test]
#[should_panic(expected: ('amount exceeds allowance',))]
fn test_transfer_from_insufficient_allowance() {
    // Deploy the contract
    let contract_address = deploy_contract("ERC20", OWNER);
    let erc20_token = IERC20Dispatcher { contract_address };

    let token_decimal = erc20_token.decimals();

    // Define amounts: 10,000 tokens to mint, 5,000 to approve
    let mint_amount: u256 = 10000 * token_decimal.into();
    let approval_amount: u256 = 5000 * token_decimal.into();

    // Start impersonating the owner
    start_cheat_caller_address(contract_address, OWNER);

    // Mint tokens to the owner
    erc20_token.mint(OWNER, mint_amount);

    let spender: ContractAddress = 'SPENDER'.try_into().unwrap();

    // Owner approves SPENDER to spend 5,000 tokens
    erc20_token.approve(spender, approval_amount);

    // Stop impersonating owner
    stop_cheat_caller_address(contract_address);

    // Attempt to transfer more than approved (6,000 instead of 5,000)
    // This should panic with 'amount exceeds allowance'
    cheat_caller_address(contract_address, spender, CheatSpan::TargetCalls(1));
    erc20_token.transfer_from(OWNER, TOKEN_RECIPIENT, 6000 * token_decimal.into());
}

#[test]
#[should_panic(expected: ('amount exceeds balance',))]
fn test_transfer_from_insufficient_balance() {
    // Deploy the contract
    let contract_address = deploy_contract("ERC20", OWNER);
    let erc20_token = IERC20Dispatcher { contract_address };

    let token_decimal = erc20_token.decimals();

    // Define amounts: only 1,000 tokens minted, but 2,000 approved and attempted
    let mint_amount: u256 = 1000 * token_decimal.into();
    let approval_amount: u256 = 2000 * token_decimal.into();
    let transfer_amount: u256 = 2000 * token_decimal.into();

    // Start impersonating the owner
    start_cheat_caller_address(contract_address, OWNER);

    // Mint only 1,000 tokens to the owner
    erc20_token.mint(OWNER, mint_amount);

    let spender: ContractAddress = 'SPENDER'.try_into().unwrap();

    // Owner approves SPENDER to spend 2,000 tokens (more than balance)
    erc20_token.approve(spender, approval_amount);

    // Stop impersonating owner
    stop_cheat_caller_address(contract_address);

    // Spender has sufficient allowance but owner doesn't have enough balance
    // This should panic with 'amount exceeds balance'
    cheat_caller_address(contract_address, spender, CheatSpan::TargetCalls(1));
    erc20_token.transfer_from(OWNER, TOKEN_RECIPIENT, transfer_amount);
}
```

To run all tests, use the command `scarb test` in your terminal. This will execute all the test functions and display the results. You should see output indicating that each test passed:

![A screenshot showing tests passing](https://r2media.rareskills.io/CairoERC20/image3.png)

### Exercise: Testing the `mint` function

Write a test for the `mint` function to practice what you've learned. The test should verify that:

- Only the owner can mint tokens
- The recipient's balance increases by the minted amount
- The total supply increases by the minted amount

Once done, run `scarb test test_mint` to verify it works.

# Conclusion

This tutorial covered building and testing an ERC-20 token contract on Starknet. From here, the contract can be extended with features like pausing, access controls, and so on.

Alternatively, OpenZeppelin's pre-built components for Cairo can be used instead of building everything from scratch. See the "Component 2" chapter to learn how to integrate OpenZeppelin's ERC20, Ownable, and Pausable components into a contract.

*This article is part of a tutorial series on [Cairo Programming on Starknet](https://rareskills.io/cairo-tutorial)*