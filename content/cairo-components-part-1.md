# Cairo Components Part 1

Components in Cairo behave like abstract contracts in Solidity. They can define and work with storage, events, and functions, but they can’t be deployed on their own. The intended use of components is to separate logic (*e.g re-usability*) in a similar manner to how abstract contracts in Solidity do.

Consider the following Solidity code:

```solidity
abstract contract C {
    uint256 balance;

    function increase_balance(uint256 amount) public {
        require(amount != 0, "amount cannot be zero");
        balance = balance + amount;
    }

    function get_balance() public view returns (uint256) {
        return x;
    }
}

contract D is C {

}

```

The contract `C` cannot be deployed because it is abstract. However, if `D` is deployed, then `D` will have all the functionality and state of `C`. Specifically, `D` will have public functions `increase_balance()` and `get()` which behave as defined in `C`.

`D` received all of the functions, events, and storage of `C`.

The contract we will build today is the Cairo equivalent of the Solidity code shown above.

## Minimal Component Example

Create an empty directory and run `scarb init` inside.

Paste the code below into `src/lib.cairo`. Run the generated tests with `scarb test`; they should all pass.

Here is what the code does:

- It declares an interface with two functions that increase and returns a stored balance in the contract.
- It creates a component that defines its own storage `x` and implements the `increase` and `get_balance` functions using read/write operations.
- The contract imports the component, registers its storage and events, and exposes the component’s implementation through its ABI.

We’ll explain how they all come together after the code:

```rust
// SAME TRAIT SCARB CREATES BY DEFAULT

#[starknet::interface]
pub trait IHelloStarknet<TContractState> {
    /// Increase contract balance.
    fn increase_balance(ref self: TContractState, amount: felt252);
    /// Retrieve contract balance.
    fn get_balance(self: @TContractState) -> felt252;
}

// COMPONENT IS NEW

#[starknet::component]
pub mod CounterComponent {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    pub struct Storage {
        x: felt252,
    }

    #[event]
    #[derive(Drop, Debug, PartialEq, starknet::Event)]
    pub enum Event {}

    #[embeddable_as(CounterImplMixin)]
    impl CounterImpl<TContractState, +HasComponent<TContractState>> of super::IHelloStarknet<ComponentState<TContractState>> {
        fn get_balance(self: @ComponentState<TContractState>) -> felt252 {
            self.x.read()
        }

        fn increase_balance(ref self: ComponentState<TContractState>, amount: felt252) {
            assert(amount != 0, 'Amount cannot be 0');
            self.x.write(self.x.read() + amount);
        }
    }
}

// THIS CONTRACT HAS NO FUNCTIONALITY, IT ONLY USES THE COMPONENT

#[starknet::contract]
mod HelloStarknet {
    use super::CounterComponent;

    component!(path: CounterComponent, storage: counter, event: CounterEvent);

    #[abi(embed_v0)]
    impl CounterImpl = CounterComponent::CounterImplMixin<ContractState>;

    #[storage]
    struct Storage {
        #[substorage(v0)]
        counter: CounterComponent::Storage,
    }

    #[event]
    #[derive(Drop, Debug, PartialEq, starknet::Event)]
    pub enum Event {
        #[flat]
        CounterEvent: CounterComponent::Event,
    }
}

```

## The Above Contract Breakdown

### `IHelloStarknet` Interface

The trait at the top of the file is unchanged from the one Scarb creates by default.

We didn’t change it because the test files specifically import this interface. Using a different name would cause the tests to not compile:

```rust
#[starknet::interface]
pub trait IHelloStarknet<TContractState> {
    /// Increase contract balance.
    fn increase_balance(ref self: TContractState, amount: felt252);
    /// Retrieve contract balance.
    fn get_balance(self: @TContractState) -> felt252;
}

```

### The Counter Component

The `CounterComponent` (*similar to the “abstract contract” in Solidity we saw earlier*) is nearly identical to contract Scarb creates by default. The differences are explained after the code blocks.

**`CounterComponent`**:

```rust
#[starknet::component]
pub mod CounterComponent {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    pub struct Storage {
        x: felt252,
    }

    #[event]
    #[derive(Drop, Debug, PartialEq, starknet::Event)]
    pub enum Event {}

    #[embeddable_as(CounterImplMixin)]
    impl CounterImpl<TContractState, +HasComponent<TContractState>> of super::IHelloStarknet<ComponentState<TContractState>> {
        fn get_balance(self: @ComponentState<TContractState>) -> felt252 {
            self.x.read()
        }

        fn increase_balance(ref self: ComponentState<TContractState>, amount: felt252) {
            assert(amount != 0, 'Amount cannot be 0');
            self.x.write(self.x.read() + amount);
        }
    }
}

```

Default contract created by Scarb:

```rust
#[starknet::contract]
mod HelloStarknet {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    struct Storage {
        balance: felt252,
    }

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn increase_balance(ref self: ContractState, amount: felt252) {
            assert(amount != 0, 'Amount cannot be 0');
            self.balance.write(self.balance.read() + amount);
        }

        fn get_balance(self: @ContractState) -> felt252 {
            self.balance.read()
        }
    }
}

```

Here are the differences between the `CounterComponent` and the default contract Scarb generates:

- The component is annotated with the attribute `#[starknet::component]`
    - The contract is annotated with the attribute `#[starknet::contract]`
- The `impl` in the component has the attribute `#[embeddable_as(CounterImplMixin)]`
    - The contract has attribute `#[abi(embed_v0)]`
- In the component, the `CounterImpl` has the trait `impl CounterImpl<TContractState, +HasComponent<TContractState>> of super::IHelloStarknet<ComponentState<TContractState>>`
    - The contract has trait`impl HelloStarknetImpl of super::IHelloStarknet<ContractState>`
- The component declares an empty event block even though it does not use events
    - Contracts can omit the event block, but a component cannot. In practice, most real-world components will have events. We keep the event empty for now to maximize simplicity.

Next is a detailed explanation of the differences listed above.

### `#[starknet::component]` vs `#[starknet::contract]`

If we intend to build a component instead of a contract, the compiler needs to know the type of module. Annotating the `mod` block with `#[starknet::component]` tells the compiler we are building a component, while `[starknet::contract]` tells the compiler we are building a contract.

### `#[embeddable_as(CounterImplMixin)]`

This attribute allows a contract to “bring in” an `impl` from the component.

```rust
// Counter Mixin
#[abi(embed_v0)]
impl CounterImpl = CounterComponent::CounterImplMixin<ContractState>;

```

In the contract: `CounterComponent` refers to the module `CounterComponent` and `CounterImplMixin` refers to the `Impl` it is bringing in (”mixing in”).

The name `CounterImplMixin` is arbitrary.

We could have written `#[embeddable_as(FooBar)]` in the component and put the following code in the contract:

```rust
#[abi(embed_v0)]
impl CounterImpl = CounterComponent::FooBar<ContractState>;

```

> *We can define multiple `embeddable_as` implementations within a component if we want to expose different impl blocks for different purposes (we will show an example for this in a follow up article).*
> 

A “Mixin” is not a language construct or a term the compiler recognizes. It’s idiomatic terminology in Cairo for an `impl` that is included in a contract from a component, *and* that `impl` will expose new “public” functions in the contract. A contract could include an `impl` which does not expose any external functions, but this would not be considered a “mixin.”

The `#[abi(embed_v0)]` in the contract exposes the functions from the counter `impl`. If we did not include `#[abi(embed_v0)]` as follows:

```rust
// #[abi(embed_v0)] commented out
impl CounterImpl = CounterComponent::Counter<ContractState>;

```

Our code would still compile, but there would be no public functions, so the tests wouldn’t pass.

### Understanding the `impl` definition in Component

The `impl` definition in the above component looks like this:

```rust
impl CounterImpl<TContractState, +HasComponent<TContractState>> of super::IHelloStarknet<ComponentState<TContractState>>

```

This looks intimidating at first, especially if you don’t have a Rust background. The good news is that it’s mostly boilerplate, you will reuse this pattern across all components, you won’t have to rewrite it. But we should know what it means.

In a component, every `impl` follows this structure:

```rust
impl {ImplName}<TContractState, +HasComponent<TContractState>> of {PathToTrait}::{TraitName}<ComponentState<TContractState>>

```

Let’s break that down:

- `{ImplName}` is the name you give to the implementation block. It can be anything you choose.
- `TContractState` represents the contract’s state type.
- `+HasComponent<TContractState>` tells the compiler that the contract using this component includes its state.
- `of {PathToTrait}::{TraitName}` links the implementation to the trait that defines the component’s interface.
- `ComponentState<TContractState>` means the trait operates on the component’s part of the contract state.

In our example:

- `{PathToTrait}` is `super` because the trait is declared in the same file.
- `{TraitName}` is `IHelloStarknet`, since the tests expect this specific trait name.

Once you understand this pattern, you can reuse it whenever you declare an implementation for a component.

## How the contract uses the component

Here is the contract code again:

```rust
#[starknet::contract]
mod HelloStarknet {
    use super::CounterComponent;

    component!(path: CounterComponent, storage: counter, event: CounterEvent);

    #[abi(embed_v0)]
    impl CounterImpl = CounterComponent::CounterImplMixin<ContractState>;

    #[storage]
    struct Storage {
        #[substorage(v0)]
        counter: CounterComponent::Storage,
    }

    #[event]
    #[derive(Drop, Debug, PartialEq, starknet::Event)]
    pub enum Event {
        #[flat]
        CounterEvent: CounterComponent::Event,
    }
}

```

In Solidity, functions, storage variables, and events are automatically “pulled in” when a contract inherits from an abstract contract. This is not the case in Cairo.

To “bring in” a component, we need to follow this checklist:

1. Import the component with `use`. In this case, it is `use super::CounterComponent`. This merely makes the code *available* and does not *integrate* it into the component
2. We must declare the `component!` macro. This will be explained more shortly
3. Public functions must be mixed in
4. Storage must be embedded as `#[substorage(v0)]`
5. Events must be embedded with `#[flat]`

None of these steps are optional. Below is a detailed explanation of each item in the checklist.

### Importing the Component

Naming our `CounterComponent` ”CounterComponent” is optional. It could be called “SparklingWaterIsTasty” and that would be fine. However, the component name we import must be the same name used for:

- `path` in `component!`
- It must be the source of the Mixin, Storage, and Event as highlighted below

![CounterComponent keyword higlighted](https://r2media.rareskills.io/CairoComponents1/image4.png)

### Importing the impl

To include the external functions of the component in our contract, we must do the following:

- Declare an `impl` and make it external with the `#[abi(embed_v0)]` attribute (in orange below)
- Bring in the `CounterImplMixin` from the component. The name `CounterImplMixin` must match the name declared in the component’s `#[embeddable_as(CounterImplMixin)]`. **Matching the impl name in the component does not guarantee the import will work, you must use the name declared in the `embeddable_as` macro.**
- Finally, we give the `CounterImplMixin` mixin “access” to the contract storage by “passing” the `ContractState` (as shown in the white box below).

![Code with abi embed highlighted](https://r2media.rareskills.io/CairoComponents1/image3.png)

### Importing the storage

Unlike contract inheritance which automatically imports storage, this must be done manually in Cairo.

All storage of a contract exists in the struct labelled `#[storage]` in the contract. Although there is also a `#[storage]` struct in the component, this doesn’t “count” since that is in a component.

Fortunately, we do not have to import every single storage variable separately. We import the storage “in one go” with the `#[substorage(v0)]` attribute.

Now let’s show how to import storage:

- All of the storage imported from a component must have a key inside the contract’s storage struct. The name of this key must match the name declared in the `component!` macro (the green box and arrow below). This could be named anything, but the they must be consistent between the value declared in the `component!` and the key in the struct. The name `counter` itself is arbitrary. This link is also how the compiler knows that the storage inside of `counter` was defined elsewhere.
- To bring in the storage struct of the component (not the contract), we put it as the value in the struct as `CounterComponent::Storage` (Yellow and purple box below). Note that `Storage` here is the name of the struct in the component.

![Code highlighted with storage imports highlighted](https://r2media.rareskills.io/CairoComponents1/image1.png)

### Importing the events

Importing the events follows the same pattern as importing the storage:

- The `CounterEvent` declared in the `component!` macro must match the corresponding item in the contract’s `Event` enum. This one-to-one match is how the compiler knows the event is defined outside the contract. The name `CounterEvent` is arbitrary, but whatever name we choose must appear exactly the same in both the `component!` macro and the enum variant.
- The `#[flat]` (as shown in the orange box below) attribute above the entry is a required boilerplate that tells the compiler to flatten the component's events into your contract's event structure rather than nesting them.
- The `CounterComponent` is how we bring in the `Event`. The `Event` in magenta is the `Event` enum declared in the component.

![Code highlighted with events imported](https://r2media.rareskills.io/CairoComponents1/image2.png)

## Summary

A component creates its own functions, storage, and events but cannot be deployed as a contract.

A component can be imported into a contract using an import and declaring references with `component!`

The functions, storage, and events must be imported separately.

To import the functions, create a new `impl` declared with `#[abi(embed_v0)]` and set the implementation to the mixin name specified in `#[embeddable_as(mixin_name)]`.

To import storage, create a new key in the contract’s storage with `#[substorage(v0)]`. Set the key for the storage to be the same name declared for `storage:` in the `component!` macro. Then set the value to be the path to the storage struct in the component.

To import events, create a new entry in the event enum and apply the `#[flat]` attribute to it. Set the entry to be the same name declared for `event:` in the `component!` macro. Then set the type to be the path to the enum in the component.

*This article is part of a tutorial series on [Cairo Programming on Starknet](https://rareskills.io/cairo-tutorial)*