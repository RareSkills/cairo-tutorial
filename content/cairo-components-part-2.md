# Components Part 2: OpenZeppelin ERC-20 Tutorial

In Component Part 1, we learned how to create and use a component within a single file. We built a `CounterComponent` from scratch and integrated its storage, events, and implementations into our contract.

Most components used in smart contracts come from external libraries. OpenZeppelin Contracts for Cairo provides components for ownership, access control, token standards, and more that can be imported into contracts, similar to OpenZeppelin Contracts for Solidity.

In this tutorial, you’ll learn how to import and use components from OpenZeppelin, rather than building everything from scratch; understand import paths for external crate components; and use the [OpenZeppelin Wizard](https://docs.openzeppelin.com/contracts-cairo/2.x/wizard) to generate boilerplate code.

# Setting Up Dependencies
Before we can import OpenZeppelin components, we need to add the OpenZeppelin Contracts library as a dependency in our project. Cairo uses [Scarbs.xyz](https://scarbs.xyz/) as its official package registry, similar to npm for JavaScript or crates.io for Rust.

Create a new scarb project and navigate to its directory:

```rust
scarb new erc20_component
cd erc20_component
```

Open `Scarb.toml` file in your project directory and add the following entry under the `[dependencies]` section:

```rust
[dependencies]
starknet = "2.13.1"
openzeppelin = "2.0.0" //ADD THIS LINE
```

![OpenZeppelin in Scarb.toml](https://r2media.rareskills.io/CairoComponentsOpenZeppelin/OZScarb.png)

The syntax `openzeppelin = "2.0.0"` automatically fetches the package from [Scarbs.xyz](https://scarbs.xyz/), Cairo's official package registry. The version "2.0.0" specifies which OpenZeppelin Contracts release to use. We're using `v2.0.0`, which is the latest stable release at the time of writing. Check [Scarbs.xyz for OpenZeppelin](https://scarbs.xyz/packages?q=openzeppelin) or the [OpenZeppelin Contracts for Cairo releases page](https://github.com/OpenZeppelin/cairo-contracts/releases) for the current latest version.

Run `scarb build` to download and compile the dependencies. Once the build succeeds, the dependencies are ready and you can import OpenZeppelin components into your contract.

# Building an ERC20 Token with OpenZeppelin Wizard

We'll build an ERC20 token contract using OpenZeppelin components. Using the OpenZeppelin Wizard, we’ll generate the contract code and then explain how the components are imported and integrated.

## Using the OpenZeppelin Wizard

The OpenZeppelin Wizard is an interactive web-based tool that generates boilerplate code for contracts. Instead of building from scratch, it allows us to select the features we want and produces complete contract code ready to use. It's a faster way to implement functionality like Ownable, ERC20, ERC721, and more.

Our token will use these three components:

- **ERC20Component:** For token functionality
- **OwnableComponent:** For access control
- **PausableComponent:** To pause/unpause token transfers

Now that we understand what the OpenZeppelin Wizard does, let’s use it to generate a contract. The OpenZeppelin Wizard for Cairo is available on the Wizard subdomain of the OpenZeppelin website. Go to [OpenZeppelin Wizard for Cairo](https://wizard.openzeppelin.com/cairo) in your browser and select ‘ERC20’ as the contract type.

In the ‘SETTINGS’ section, change the name to your desired token name and update the symbol. In the ‘FEATURES’ section, check (☑️) Mintable and Pausable; Ownable is automatically checked.

![OpenZeppelin ERC-20 builders](https://r2media.rareskills.io/CairoComponentsOpenZeppelin/image2.png)

Copy the code at the top right, and paste it into `src/lib.cairo` file in your project directory. Your generated code should look similar to the following contract with all necessary imports, component declarations, storage structure, events, constructor, and custom functions (pause, unpause, and mint):

```rust
// SPDX-License-Identifier: MIT
// Compatible with OpenZeppelin Contracts for Cairo ^2.0.0

#[starknet::contract]
mod RareToken {
    use openzeppelin::access::ownable::OwnableComponent;
    use openzeppelin::security::pausable::PausableComponent;
    use openzeppelin::token::erc20::{DefaultConfig as ERC20DefaultConfig, ERC20Component};
    use starknet::ContractAddress;

    component!(path: ERC20Component, storage: erc20, event: ERC20Event);
    component!(path: PausableComponent, storage: pausable, event: PausableEvent);
    component!(path: OwnableComponent, storage: ownable, event: OwnableEvent);

    // External
    #[abi(embed_v0)]
    impl ERC20MixinImpl = ERC20Component::ERC20MixinImpl<ContractState>;
    #[abi(embed_v0)]
    impl PausableImpl = PausableComponent::PausableImpl<ContractState>;
    #[abi(embed_v0)]
    impl OwnableMixinImpl = OwnableComponent::OwnableMixinImpl<ContractState>;

    // Internal
    impl ERC20InternalImpl = ERC20Component::InternalImpl<ContractState>;
    impl PausableInternalImpl = PausableComponent::InternalImpl<ContractState>;
    impl OwnableInternalImpl = OwnableComponent::InternalImpl<ContractState>;

    #[storage]
    struct Storage {
        #[substorage(v0)]
        erc20: ERC20Component::Storage,
        #[substorage(v0)]
        pausable: PausableComponent::Storage,
        #[substorage(v0)]
        ownable: OwnableComponent::Storage,
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        #[flat]
        ERC20Event: ERC20Component::Event,
        #[flat]
        PausableEvent: PausableComponent::Event,
        #[flat]
        OwnableEvent: OwnableComponent::Event,
    }

    #[constructor]
    fn constructor(ref self: ContractState, owner: ContractAddress) {
        self.erc20.initializer("RareToken", "RTK");
        self.ownable.initializer(owner);
    }

    impl ERC20HooksImpl of ERC20Component::ERC20HooksTrait<ContractState> {
        fn before_update(
            ref self: ERC20Component::ComponentState<ContractState>,
            from: ContractAddress,
            recipient: ContractAddress,
            amount: u256,
        ) {
            let contract_state = self.get_contract();
            contract_state.pausable.assert_not_paused();
        }
    }

    #[generate_trait]
    #[abi(per_item)]
    impl ExternalImpl of ExternalTrait {
        #[external(v0)]
        fn pause(ref self: ContractState) {
            self.ownable.assert_only_owner();
            self.pausable.pause();
        }

        #[external(v0)]
        fn unpause(ref self: ContractState) {
            self.ownable.assert_only_owner();
            self.pausable.unpause();
        }

        #[external(v0)]
        fn mint(ref self: ContractState, recipient: ContractAddress, amount: u256) {
            self.ownable.assert_only_owner();
            self.erc20.mint(recipient, amount);
        }
    }
}
```

Without much effort, we have generated a fully functional contract with mintable, pausable and access control features.

With the generated code in hand, let's break down how the OpenZeppelin components are imported and integrated into the contract.

## Understanding the Generated Code

When working with components, there are three steps required:

1. importing the component,
2. linking your contract to it using the `component!` macro, and
3. embedding the component implementations to expose their functions in your contract

Let's see how this works in our generated  `RareToken` contract.

### Step 1: Importing Components

The first step is importing the components. The import statements highlighted in the code below bring the `OwnableComponent`, `PausableComponent`, and `ERC20Component` into the contract's scope, making their functionality available to use:

![OpenZeppelin import in token](https://r2media.rareskills.io/CairoComponentsOpenZeppelin/image3.png)

### Step 2: Linking Components with the `component!` Macro

After importing the required components, the components are set up (linked) in the contract using the `component!` macro:

![component! macro illustration](https://r2media.rareskills.io/CairoComponentsOpenZeppelin/image4.png)

The `component!` macro declares how our contract will connect to each component. It takes three arguments:

1. `path`: The path to the component (what was imported). In this case: `ERC20Component`, `PausableComponent`, and `OwnableComponent`
2. `storage`: The name of the storage variable in the contract that points to the component's storage. To access a component's storage, you need a variable in your contract's storage that references the component's storage

![Storage import for OpenZeppelin](https://r2media.rareskills.io/CairoComponentsOpenZeppelin/image5.png)

In the example above, the storage names `erc20`, `pausable`, and `ownable` were used. These names can be customized, but they must match what’s declared in the contract's storage struct.

As discussed in Component Part 1, each storage field is annotated with `#[substorage(v0)]` to indicate that it references component storage.

**3.** `event`: The name of the event variant in the contract that points to the component's events.

In the screenshot below, notice how the event names highlighted at the top (lines 11-13) correspond to the event variants highlighted at the bottom (lines 42, 44, 46). The `event` parameter in the `component!` macro (e.g., `ERC20Event`) maps to the variant name in the contract's event enum.

![Event import](https://r2media.rareskills.io/CairoComponentsOpenZeppelin/image6.png)

In this case, `ERC20Event`, `PausableEvent`, and `OwnableEvent` was used. Like storage names, these can be anything, but they must match what’s declared in the contract's event enum.

The `#[flat]` attribute applied to each event variant is important here. Recall from the  ***"Using `#[flat]` attribute"*** section of the Events chapter that the `#[flat]` attribute changes how event selectors are computed.

Without `#[flat]`, component events include a component ID as the first key, and all events from a component variant share the same selector computed from the outer enum variant name. For example, both `Transfer` and `Approval` events from `ERC20Component` would use `starknetKeccak("ERC20Event")` as their selector, making it impossible to distinguish between different event types based on the selector alone.

With `#[flat]`, the component ID prefix is removed, and each event uses its own struct name for the selector: `starknetKeccak("Transfer")`, `starknetKeccak("Approval")`. This enables precise event filtering and matches the standard event structure expected by external tools and indexers.

### Step 3: Component Implementations

Now let's look at the component implementations in the generated code. There are two types: external and internal. External implementations can be called from outside the contract, while internal ones can only be used within the contract.

The generated code includes three external implementations that expose the components' functionality:

```rust
// External
#[abi(embed_v0)]
impl ERC20MixinImpl = ERC20Component::ERC20MixinImpl<ContractState>;
#[abi(embed_v0)]
impl PausableImpl = PausableComponent::PausableImpl<ContractState>;
#[abi(embed_v0)]
impl OwnableMixinImpl = OwnableComponent::OwnableMixinImpl<ContractState>;
```

The `#[abi(embed_v0)]` attribute makes these implementations publicly accessible; their functions can be called from outside the contract. Let's break down each implementation.

 [`ERC20MixinImpl`](https://github.com/OpenZeppelin/cairo-contracts/blob/main/packages/token/src/erc20/erc20.cairo#L119-#L241) combines all necessary ERC20 functionality in one package:

- `ERC20Impl` : has the core functions such as `transfer`, `approve`, `balance_of`
- `ERC20MetadataImpl` : has Metadata functions like `name`, `symbol`, `decimals`
- `ERC20CamelImpl` : has the Camel-case function versions for compatibility (e.g., `balanceOf`, `totalSupply`)

Using the ERC20 mixin saves us from embedding each implementation separately.

In addition to the ERC20 mixin, the contract embeds two other external implementations:

- `PausableImpl` provides `pause()` to halt contract operations, `unpause()` to resume them, and `is_paused()` to check the current pause state
- `OwnableMixinImpl` provides `owner()` to view the current owner, `transfer_ownership()` to transfer ownership to a new address, and `renounce_ownership()` to remove the owner entirely

The generated code also includes these internal implementations:

```rust
// Internal
impl ERC20InternalImpl = ERC20Component::InternalImpl<ContractState>;
impl PausableInternalImpl = PausableComponent::InternalImpl<ContractState>;
impl OwnableInternalImpl = OwnableComponent::InternalImpl<ContractState>;
```

Notice that the implementation above don't have `#[abi(embed_v0)]`that’s because they're not publicly callable from outside the contract.

### Constructor

The constructor sets up the token's name and symbol through the [ERC20 component's initialize](https://github.com/OpenZeppelin/cairo-contracts/blob/main/packages/token/src/erc20/erc20.cairo#L427-#L438)r and sets the contract owner through the [Ownable component's initializer](https://github.com/OpenZeppelin/cairo-contracts/blob/main/packages/access/src/ownable/ownable.cairo#L283-#L286).

```rust
#[constructor]
fn constructor(
    ref self: ContractState,
    name: ByteArray,
    symbol: ByteArray,
    fixed_supply: u256,
    recipient: ContractAddress,
    owner: ContractAddress
) {
    self.erc20.initializer(name, symbol);
    self.erc20.mint(recipient, fixed_supply);
    self.ownable.initializer(owner);
}
```

Each initializer can only be called once, locking in these settings after deployment.

### ERC20 Hooks

Hooks are functions that run automatically before or after certain operations. The ERC20 component provides an [`ERC20HooksTrait`](https://github.com/OpenZeppelin/cairo-contracts/blob/main/packages/token/src/erc20/erc20.cairo#L98-#L112) that allows you to add logic that runs during token transfers.

**The `before_update` hook**

The generated code contains a `before_update` hook that checks if the contract is paused before any token operation:

```rust
impl ERC20HooksImpl of ERC20Component::ERC20HooksTrait<ContractState> {
    fn before_update(
        ref self: ERC20Component::ComponentState<ContractState>,
        from: ContractAddress,
        recipient: ContractAddress,
        amount: u256,
    ) {
        let contract_state = self.get_contract();
        contract_state.pausable.assert_not_paused();
    }
}
```

The `before_update` function runs before any token balance change (transfers, mints, or burns). In this implementation:

- `self.get_contract()` retrieves the contract state
- `contract_state.pausable.assert_not_paused()` checks if the contract is paused
- If paused, the transaction reverts; if not, the transfer proceeds

This is how the pausable feature works; by checking the pause state before every token operation, the contract can halt all transfers when paused.

**Before and After Update Hooks**

Without implementing the `before_update` hook in the generated code, the pausable component would exist in the contract but wouldn't actually affect token transfers.

The `ERC20HooksTrait` also includes an `after_update` hook which runs after a token operation completes. While it’s not used in this contract, you could implement it to add custom logic that executes after transfers, mints, or burns.

**Exposing Internal Component Functions**

Some component functions like `pause()` and `mint()` are internal; they exist within the components but aren't publicly accessible. The generated code creates public wrapper functions that expose these operations while adding owner-only access control:

```rust
 #[generate_trait]
    #[abi(per_item)]
    impl ExternalImpl of ExternalTrait {
        #[external(v0)]
        fn pause(ref self: ContractState) {
            self.ownable.assert_only_owner();
            self.pausable.pause();
        }

        #[external(v0)]
        fn unpause(ref self: ContractState) {
            self.ownable.assert_only_owner();
            self.pausable.unpause();
        }

        #[external(v0)]
        fn mint(ref self: ContractState, recipient: ContractAddress, amount: u256) {
            self.ownable.assert_only_owner();
            self.erc20.mint(recipient, amount);
        }
    }
```

The `#[generate_trait]` attribute automatically generates the `ExternalTrait` interface from this implementation, so you don't have to write the trait definition manually.

 `#[abi(per_item)]` attribute marks each function individually for ABI generation, and when combined with `#[external(v0)]` on each function, makes them part of the contract's public interface. The `v0` in `#[external(v0)]` specifies the ABI version.

**How the Wrapper Functions Work**

Each function follows the same pattern: verify ownership, then execute the operation. For example, `pause()` calls `self.ownable.assert_only_owner()` to verify the caller is the owner, then calls `self.pausable.pause()` to pause the contract; if the caller isn't the owner, the transaction reverts.

Similarly, `unpause()` verifies ownership and then unpauses the contract, while `mint()` verifies ownership and then mints new tokens to the specified recipient address using `self.erc20.mint()`.

Without these wrapper functions, internal component functions like `pause()`, `unpause()`, and `mint()` would exist, but the owner/deployer won't be able to interact with them from outside the contract.

# Testing the Contract

Now that we have the token contract set up, let's write some tests. We'll focus on testing the custom features we added: `pause()`, `unpause()`, and `mint()` with their access controls.

## Setting up the test file

Navigate to `tests/test_contract.cairo` in your project directory. Clear the tests generated with the boilerplate, leaving only the basic imports:

```rust
use starknet::ContractAddress;
use snforge_std::{declare, ContractClassTrait, DeclareResultTrait};
```

To interact with the standard ERC-20 functions in our tests, we need to import the ERC-20 interface and its dispatcher trait from OpenZeppelin:

```rust
use starknet::ContractAddress;
use snforge_std::{declare, ContractClassTrait, DeclareResultTrait};

// NEWLY ADDED//
use openzeppelin::token::erc20::interface::{IERC20Dispatcher, IERC20DispatcherTrait};
```

The `IERC20Dispatcher` allows us to call standard ERC-20 functions like `transfer`, `balance_of`, and `total_supply` on our contract.

Recall that the generated contract used the `#[generate_trait]` attribute to automatically create traits for the custom functions (`pause`, `unpause`, `mint`). These traits weren't written explicitly in the contract, so to call these functions in tests, a manual interface definition is needed as shown below:

```rust
use starknet::ContractAddress;
use snforge_std::{declare, ContractClassTrait, DeclareResultTrait};
use openzeppelin::token::erc20::interface::{IERC20Dispatcher, IERC20DispatcherTrait};

// NEWLY ADDED //
// Define the interface for our custom functions
#[starknet::interface]
trait IRareToken<TContractState> {
    fn pause(ref self: TContractState);
    fn unpause(ref self: TContractState);
    fn mint(ref self: TContractState, recipient: ContractAddress, amount: u256);

    fn decimals(self: @TContractState) -> u8;
}
```

The `IRareToken` interface in the code above exposes the custom functions in the test environment. The `#[starknet::interface]` attribute generates the dispatcher (`IRareTokenDispatcher`) and dispatcher trait ( `IRareTokenDispatcherTrait`) that will be used to interact with those functions.

We need consistent addresses for the tests. Define constants to provide test addresses instead of creating new ones each time:

```rust
const OWNER: ContractAddress = 'OWNER'.try_into().unwrap();
const USER: ContractAddress = 'USER'.try_into().unwrap();
const RECIPIENT: ContractAddress = 'RECIPIENT'.try_into().unwrap();
```

These constants convert string literals into contract addresses.

Now we need a helper function to deploy our token contract in the test environment. Add the `deploy_token` function in `test_contract.cairo`:

```rust
use starknet::ContractAddress;
use snforge_std::{declare, ContractClassTrait, DeclareResultTrait};

use openzeppelin::token::erc20::interface::{IERC20Dispatcher, IERC20DispatcherTrait};

#[starknet::interface]
trait IRareToken<TContractState> {
    fn pause(ref self: TContractState);
    fn unpause(ref self: TContractState);
    fn mint(ref self: TContractState, recipient: ContractAddress, amount: u256);

    fn decimals(self: @TContractState) -> u8;
}

const OWNER: ContractAddress = 'OWNER'.try_into().unwrap();
const USER: ContractAddress = 'USER'.try_into().unwrap();
const RECIPIENT: ContractAddress = 'RECIPIENT'.try_into().unwrap();

// NEWLY ADDED //
fn deploy_token() -> (ContractAddress, IERC20Dispatcher, IRareTokenDispatcher) {
    let contract = declare("RareToken").unwrap().contract_class();
    let mut constructor_args = array![OWNER.into()];
    let (contract_address, _) = contract.deploy(@constructor_args).unwrap();
    let token = IERC20Dispatcher { contract_address };
    let rare_token = IRareTokenDispatcher { contract_address };
    (contract_address, token, rare_token)
}
```

`deploy_token` uses `declare("RareToken").unwrap().contract_class()` to declare the RareToken contract and retrieve its contract class, which loads the compiled contract code.

Next, it prepares the constructor arguments with `array![OWNER.into()]`, which creates an array containing the owner address.

![Constructor with owner argument highlighted](https://r2media.rareskills.io/CairoComponentsOpenZeppelin/image7.png)

The constructor expects one parameter (the owner address), so we convert it into a `felt252` using `.into()` in the test. The token name "RareToken" and symbol "RTK" are already hardcoded in the contract's constructor.

Once the arguments are ready, `contract.deploy(@constructor_args).unwrap()` deploys the contract and returns the contract address. With the contract deployed, we create two dispatchers for the same contract address: `IERC20Dispatcher` for standard ERC-20 functions and `IRareTokenDispatcher` for the custom functions like `pause()`, `unpause()`, and `mint()`.

The function returns a tuple containing the contract address and both dispatchers, giving us everything we need to interact with the deployed contract in our tests.

## Testing `pause()` to prevent transfers

The pause function halts all token operations, which is useful during security incidents or maintenance.

Import `start_cheat_caller_address` and `stop_cheat_caller_address` alongside other imports from `snforge_std` to allow us to impersonate different addresses when calling contract functions:

```rust
use snforge_std::{declare, ContractClassTrait, DeclareResultTrait, start_cheat_caller_address,stop_cheat_caller_address};
```

Now let's write a test that verifies transfers are blocked when the contract is paused:

```rust
#[test]
fn test_pause_prevents_transfer() {
    let (contract_address, token, rare_token) = deploy_token();

    // Get token decimals for proper amount calculation
    let token_decimal = rare_token.decimals();
    let amount_to_mint: u256 = 10000 * token_decimal.into();

    // Mint tokens to USER
    start_cheat_caller_address(contract_address, OWNER);
    rare_token.mint(USER, amount_to_mint);

    // Pause the contract
    rare_token.pause();
    stop_cheat_caller_address(contract_address);

    // Try to transfer - should fail when paused
    start_cheat_caller_address(contract_address, USER);
    token.transfer(RECIPIENT, 100 * token_decimal.into());  // This should panic
}
```

The test begins by deploying the contract through `deploy_token()`, which returns the contract address and dispatchers we need to interact with the contract. We then retrieve the token's decimal places using `rare_token.decimals()`. ERC-20 tokens typically use 18 decimals, so multiplying `10000 * 10^18` gives us 10,000 tokens.

Next, we use `start_cheat_caller_address` to impersonate the OWNER and mint tokens to USER. While still acting as OWNER, we call `pause()` to activate the `pause()` function, then use `stop_cheat_caller_address` to reset the caller address back to the default.

With the contract now paused, we impersonate USER using `start_cheat_caller_address` again and attempt to transfer tokens to RECIPIENT. This transfer should fail because the contract is paused, which is exactly what we want to verify.

When you run `scarb test test_pause_prevents_transfer`, you should see this error in your terminal:

![scarb test](https://r2media.rareskills.io/CairoComponentsOpenZeppelin/image8.png)

The contract correctly rejects the transfer because it's paused. The error message comes from OpenZeppelin's Pausable component. If you check the [OpenZeppelin Pausable component source code](https://github.com/OpenZeppelin/cairo-contracts/blob/main/packages/security/src/pausable.cairo#L39-#L42), you'll see this is the exact error thrown when operations are attempted on a paused contract:

```rust
fn assert_not_paused(self: @ComponentState<TContractState>) {
    assert(!self.is_paused(), Errors::PAUSED);
}
```

We can improve the test by using the `#[should_panic]` attribute to explicitly indicate that we expect the test to panic. This makes the test pass when it panics with the expected error:

```rust
#[test]
#[should_panic(expected: ('Pausable: paused',))]
fn test_pause_prevents_transfer() {
    let (contract_address, token, rare_token) = deploy_token();

    // Get token decimals for proper amount calculation
    let token_decimal = rare_token.decimals();
    let amount_to_mint: u256 = 10000 * token_decimal.into();

    // Mint tokens to USER
    start_cheat_caller_address(contract_address, OWNER);
    rare_token.mint(USER, amount_to_mint);

    // Pause the contract
    rare_token.pause();
    stop_cheat_caller_address(contract_address);

    // Try to transfer - should fail when paused
    start_cheat_caller_address(contract_address, USER);
    token.transfer(RECIPIENT, 100 * token_decimal.into());  // This should panic
}
```

The `#[should_panic(expected: ('Pausable: paused',))]` attribute tells the test framework:

- This test should panic
- The panic should contain the error message `'Pausable: paused'`

If the test doesn't panic or panics with a different error, the test will fail. Now when you run `scarb test test_pause_prevents_transfer`, you should see the test pass successfully.

## Testing `unpause()` to allow transfers

After pausing a contract, you need the ability to resume normal operations. This test verifies that after unpausing, token transfers work as expected:

```rust
#[test]
fn test_unpause_allows_transfer() {
    let (contract_address, token, rare_token) = deploy_token();

    // Get token decimals for proper amount calculation
    let token_decimal = rare_token.decimals();
    let amount_to_mint: u256 = 1000 * token_decimal.into();

    // Mint tokens to USER
    start_cheat_caller_address(contract_address, OWNER);
    rare_token.mint(USER, amount_to_mint);

    // Pause then unpause the contract
    rare_token.pause();
    rare_token.unpause();
    stop_cheat_caller_address(contract_address);

    // Transfer should now succeed
    start_cheat_caller_address(contract_address, USER);
    token.transfer(RECIPIENT, 100 * token_decimal.into());

    // Verify the transfer worked*
    assert!(token.balance_of(USER) == 900 * token_decimal.into(), "User balance incorrect");
    assert!(token.balance_of(RECIPIENT) == 100 * token_decimal.into(), "Recipient balance incorrect");
}
```

We start by deploying the contract and getting the token decimals, then mint 1,000 tokens to USER as the OWNER. The key difference in this test is that we pause the contract and immediately unpause it while still acting as OWNER. After calling `stop_cheat_caller_address`, we switch to impersonate USER and attempt a transfer of 100 tokens to RECIPIENT.

Since the contract is no longer paused, the transfer should succeed. We verify this by checking the balances: USER should have 900 tokens remaining (1000 - 100), and RECIPIENT should have received 100 tokens. The `assert!` macro confirms these balances are correct, ensuring the unpause function properly restores normal contract operations.

Run the test with `scarb test test_unpause_allows_transfer` and it should pass, confirming that the pause mechanism can be toggled on and off successfully.

## Testing Access Control for `pause()`

Functions like `pause()` that can halt contract operations need proper access control. Only the contract owner should be able to pause the contract. This test verifies that non-owners cannot pause:

```rust
#[test]
#[should_panic(expected: ('Caller is not the owner',))]
fn test_only_owner_can_pause() {
    let (contract_address, _token, rare_token) = deploy_token();

    // Try to pause as non-owner - should panic
    start_cheat_caller_address(contract_address, USER);
    rare_token.pause();

    // no need to stop cheat since it doesn't reach here
}
```

This test is straightforward but important. We deploy the contract, then immediately try to call `pause()` as USER (not the owner). The `#[should_panic(expected: ('Caller is not the owner',))]` attribute tells the test framework we expect this to fail with a specific error message.

When `rare_token.pause()` is called, it internally triggers `self.ownable.assert_only_owner()` from the Ownable component. Since USER is not the owner, this assertion fails and the transaction reverts with the error "Caller is not the owner" as expected.

Run the test with `scarb test test_only_owner_can_pause` and it should pass, confirming that our access control works correctly.

Here's the test file we've built:

```rust
use starknet::ContractAddress;
use snforge_std::{declare, ContractClassTrait, DeclareResultTrait, start_cheat_caller_address, stop_cheat_caller_address};

use openzeppelin::token::erc20::interface::{IERC20Dispatcher, IERC20DispatcherTrait};

#[starknet::interface]
trait IRareToken<TContractState> {
    fn pause(ref self: TContractState);
    fn unpause(ref self: TContractState);
    fn mint(ref self: TContractState, recipient: ContractAddress, amount: u256);

    fn decimals(self: @TContractState) -> u8;
}

const OWNER: ContractAddress = 'OWNER'.try_into().unwrap();
const USER: ContractAddress = 'USER'.try_into().unwrap();
const RECIPIENT: ContractAddress = 'RECIPIENT'.try_into().unwrap();

fn deploy_token() -> (ContractAddress, IERC20Dispatcher, IRareTokenDispatcher) {
    let contract = declare("RareToken").unwrap().contract_class();
    let mut constructor_args = array![OWNER.into()];
    let (contract_address, _) = contract.deploy(@constructor_args).unwrap();
    let token = IERC20Dispatcher { contract_address };
    let rare_token = IRareTokenDispatcher { contract_address };
    (contract_address, token, rare_token)
}

#[test]
#[should_panic(expected: ('Pausable: paused',))]
fn test_pause_prevents_transfer() {
    let (contract_address, token, rare_token) = deploy_token();

    // Get token decimals for proper amount calculation
    let token_decimal = rare_token.decimals();
    let amount_to_mint: u256 = 10000 * token_decimal.into();

    // Mint tokens to USER
    start_cheat_caller_address(contract_address, OWNER);
    rare_token.mint(USER, amount_to_mint);

    // Pause the contract
    rare_token.pause();
    stop_cheat_caller_address(contract_address);

    // Try to transfer - should fail when paused
    start_cheat_caller_address(contract_address, USER);
    token.transfer(RECIPIENT, 100 * token_decimal.into());  // This should panic
}

#[test]
fn test_unpause_allows_transfer() {
    let (contract_address, token, rare_token) = deploy_token();

    // Get token decimals for proper amount calculation
    let token_decimal = rare_token.decimals();
    let amount_to_mint: u256 = 1000 * token_decimal.into();

    // Mint tokens to USER
    start_cheat_caller_address(contract_address, OWNER);
    rare_token.mint(USER, amount_to_mint);

    // Pause then unpause the contract
    rare_token.pause();
    rare_token.unpause();
    stop_cheat_caller_address(contract_address);

    // Transfer should now succeed
    start_cheat_caller_address(contract_address, USER);
    token.transfer(RECIPIENT, 100 * token_decimal.into());

    // Verify the transfer worked
    assert!(token.balance_of(USER) == 900 * token_decimal.into(), "User balance incorrect");
    assert!(token.balance_of(RECIPIENT) == 100 * token_decimal.into(), "Recipient balance incorrect");
}

#[test]
#[should_panic(expected: ('Caller is not the owner',))]
fn test_only_owner_can_pause() {
    let (contract_address, _token, rare_token) = deploy_token();

    // Try to pause as non-owner - should panic
    start_cheat_caller_address(contract_address, USER);
    rare_token.pause();
}
```

**Homework:** The OpenZeppelin ERC-20 library supports burning, but this function is internal. Your task is to:

- Expose the burn function in the contract by adding a public wrapper function similar to how `mint()` is exposed
- The burn should come from `get_caller_address()`
- Write tests for the burn functionality:
    - Test that a user can burn their own tokens
    - Test that burning decreases the user's balance
    - Test that burning decreases the total supply
    - Test that burning cannot happen when the contract is paused
    - Test that a user cannot burn more tokens than they have

*This article is part of a tutorial series on [Cairo Programming on Starknet](https://rareskills.io/cairo-tutorial)*
