# Structure of a Basic Contract

This article shows how to build a deployable Cairo contract for Starknet. Starting from a simple sketch, we will gradually add features to build a working contract demonstrating the core building blocks of a Cairo contract.

The contract will have a counter variable that can be increased by any amount and a function to retrieve its value.

## The first version of our contract

```rust
mod Counter {
    fn increase_counter(amount: felt252) {
        // TODO
    }

    fn get_counter() -> felt252 {
        // TODO
    }
}

```

The code above includes the following features:

- A module block, denoted by the `mod` keyword. Each Cairo contract is written inside a module. This is similar to `contract` keyword in Solidity, and the name of the module can be anything.
- Two functions: one to increase the counter and another to retrieve its current value.

## Adding an “interface” by defining a trait for the Counter contract

In Solidity, an interface defines a set of functions that a contract **must** implement. Interfaces are not mandatory for contracts, but their usage is encouraged.

In Cairo, this same idea is represented using a **trait**, which defines list of functions without providing their implementation. In that sense, a Cairo trait plays the same role as an interface in Solidity.

However, it’s important to clarify that a trait on its own is not automatically treated as a contract interface. We need to explicitly mark the trait as an interface for it to be treated as one and that’s done using an annotation which we will see in a later section.

For now, think of it this way:

- the trait describes what functions a contract must have,
- and the annotation (which we will introduce shortly) tells the trait how it should behave, in this case, as a contract interface.

It is not possible to implement the functions that are not part of the defined interface - we will see later another option for implementing extra functions.

The code below extends the Counter contract by defining a trait and providing implementations for the functions declared within it:

```rust
// Define a trait with two functions
pub trait ICounter {
    fn increase_counter(amount: felt252);
    fn get_counter() -> felt252;
}

mod Counter {
		// Implement the functions within the `ICounter` trait
    impl CounterImpl of super::ICounter {
        fn increase_counter(amount: felt252) {
            // TODO
        }

        fn get_counter() -> felt252 {
            // TODO
        }
    }
}

```

This draft adds the following features:

- A public trait, denoted by the `pub` and `trait` keywords.
- An implementation (`impl`) block contains function implementations. This block implements the `ICounter` trait. The name of the trait or implementation block can be anything, though it’s common practice to use descriptive names that reflect the contract’s purpose. By convention, Scarb follows the pattern `IContractName` for interfaces and `ContractNameImpl` for the corresponding implementation that defines the public functions.

## Adding storage

Next, we need a place to store the counter value. Since the counter value needs to persist between transactions, it should be stored as part of the contract's state. In Cairo, persistent data is stored **within a single struct named `Storage` that groups all state variables together.**

```rust
// Storage traits
use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

struct Storage {
    counter: felt252,
}
```

This includes the following features:

- A `use` statement that imports traits required to read from and write to contract storage.
- A new structure for the contract storage, defined with the `struct` keyword. The struct must be named `Storage`.
- The actual `counter` variable inside the storage.

## Adding state and logic

Now that we have storage, our functions need a way to interact with it. In Cairo, the contract state isn't automatically available; it must be passed to functions explicitly as a parameter. 

To facilitate this, Cairo uses a state reference, a specific parameter that represents and allows access to the contract’s storage.

There are two ways to define a state reference in Cairo: one that provides read and write access to storage, and another that provides read-only access. Here’s how to use them:

1. **Read and write access:** use a reference variable with the `ref` keyword.
2. **Read-only access:** use a snapshot variable with the `@` symbol. This is similar to Solidity’s `view` functions, where the function can read from but not modify contract storage.

Notice that the `increase_counter` function uses the `ref` keyword in its parameter to gain read and write access to the contract’s state, while the `get_counter` function uses the `@` symbol to gain read-only access, as shown in the code below:

```rust
pub trait ICounter<TContractState> {
	// Function that can read and modify the contract's state
    fn increase_counter(ref self: TContractState, amount: felt252);

    // Function that can only read from the contract's state
    fn get_counter(self: @TContractState) -> felt252;
}

mod Counter {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    struct Storage {
        counter: felt252,
    }

    impl CounterImpl of super::ICounter<ContractState> {

		    // Uses `ref self`: gives read and write access to the storage
        fn increase_counter(ref self: ContractState, amount: felt252) {
            self.counter.write(self.counter.read() + amount);
        }

				// Uses `@`: gives read-only access to the storage
        fn get_counter(self: @ContractState) -> felt252 {
            self.counter.read()
        }
    }
}

```

The changes we've made so far to the contract allows us to interact directly with storage through the contract state. The main changes are as follows:

- Added the `TContractState` type parameter to the trait as a placeholder NOT an actual type, so it can work with any contract state layout, instead of being tied to a specific one.
    - `pub trait ICounter {` became `pub trait ICounter<TContractState> {`
- In the impl block, the `TContractState` placeholder is replaced with the actual contract state type (`ContractState`):
    - `impl CounterImpl of super::ICounter {` became `impl CounterImpl of super::ICounter<ContractState> {`
- Added a reference to the state to both of the functions. One has write access and the other has only read access to the storage:
    - `fn increase_counter(amount: felt252) {` became `fn increase_counter(ref self: ContractState, amount: felt252) {`
    - `fn get_counter() -> felt252 {` became `fn get_counter(self: @ContractState) -> felt252 {`
- Added logic to increase the counter and to read it using `self`, which is of type `ContractState`, representing the contract’s state:
    - Added logic `self.counter.write(self.counter.read() + amount);`
    - Added logic `self.counter.read()`

## Finishing the contract with annotations

Cairo uses different annotations (*also called attributes*) to indicate how different parts of the contract should behave. These annotations specify things like:

- which trait defines the interface,
- which module is a deployable contract,
- which struct is the storage struct,
- and which implementation block exposes functions to the outside world.

Each annotation in Cairo starts with a `#[]` and is placed directly above the code it applies to. For example, placing this attribute `#[starknet::interface]` on a part of code tells that it should be treated as the contract’s interface.

Here is the complete contract with its annotations:

```rust
#[starknet::interface]
pub trait ICounter<TContractState> {
    fn increase_counter(ref self: TContractState, amount: felt252);
    fn get_counter(self: @TContractState) -> felt252;
}

#[starknet::contract]
mod Counter {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    struct Storage {
        counter: felt252,
    }

    #[abi(embed_v0)]
    impl CounterImpl of super::ICounter<ContractState> {
        fn increase_counter(ref self: ContractState, amount: felt252) {
            self.counter.write(self.counter.read() + amount);
        }

        fn get_counter(self: @ContractState) -> felt252 {
            self.counter.read()
        }
    }
}

```

The added annotations are:

- `#[starknet::interface]` marks a trait as an interface. You can’t have an `impl` block without an annotated interface.
- `#[starknet::contract]` marks a module as a Starknet smart contract.
- `#[storage]` indicates the structure that defines the contract’s storage layout. A contract must have exactly one storage `struct` with this annotation.
- `#[abi(embed_v0)]` makes the functions inside the `impl` block part of the contract’s public ABI — like `public` or `external` functions in Solidity. Omitting this annotation makes the functions available only inside this contract.
    - The `embed_v0` means embed the ABI using version 0 of the ABI format. In `v0`, function selectors are derived only from the function name. The `impl` name is not considered, which means if a contract has different `impl` blocks (implementing different traits) that both define a function with the same name, a name collision will occur. Future versions (like `v1`) may improve this by including more context in selector derivation without breaking backward compatibility.
    - In Cairo, there are no visibility keywords, like Solidity’s `private` or `internal`, to denote a private function. Another way to create private functions is to simply add them outside the `impl` block, with no annotations. An example is shown later in this article.

With these in place, the contract is ready to be compiled, deployed, and called from other contracts or clients.

## Functions outside the interface

In Cairo, it’s also possible to define public functions outside an interface implementation by marking them with annotation `#[external(v0)]`. Just like in `#[abi(embed_v0)]`, the `v0` in `#[external(v0)]` means use the ABI version 0.

It’s possible to use both interfaces and annotated external functions in a contract. However, using interfaces is recommended because it allows external contracts to rely on a shared definition when interacting with your contract.

In the following code, we will add a new function `increase_counter_by_five` that is annotated with `#[external(v0)]`. This function is externally callable and included in the contract’s ABI, even though it isn’t defined through an interface (*it behaves like public functions but without an interface*).

This new function calls another new, private, function `get_five` . This function is callable only inside this contract.

```rust
#[starknet::interface]
pub trait IHelloStarknet<TContractState> {
    fn increase_counter(ref self: TContractState, amount: felt252);
    fn get_counter(self: @TContractState) -> felt252;
}

#[starknet::contract]
mod HelloStarknet {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    struct Storage {
        counter: felt252,
    }

    #[abi(embed_v0)]
    impl CounterImpl of super::IHelloStarknet<ContractState> {
        fn increase_counter(ref self: ContractState, amount: felt252) {
            self.counter.write(self.counter.read() + amount);
        }

        fn get_counter(self: @ContractState) -> felt252 {
            self.counter.read()
        }
    }

	// ********* NEWLY ADDED - START ********* //
    #[external(v0)]
    fn increase_counter_by_five(ref self: ContractState) {
        self.counter.write(self.counter.read() + get_five());
    }

    fn get_five() -> felt252 {
        5
    }
    // ********* NEWLY ADDED - END ********* //
}

```

> *Note that the first parameter of any `public` or `external` function must be a reference to the contract’s state.*
> 

## Compiling the contract

To ensure our code is valid and ready to run, we should compile it.
A popular tool for working with Cairo code is [**Scarb**](https://docs.swmansion.com/scarb/) — a Cairo package manager and build system. If you haven’t already, install it by following the instructions in [Cairo for Solidity developers](https://www.notion.so/Cairo-for-Solidity-Developers-2335396265d780b49bcbca88c8dba6cf?pvs=21) article.

Once installed, you can create and compile your contract project with the following steps:

1. Initialize a new project by running `scarb new counter`, and proceed with the default test runner when prompted.
2. Navigate into the project folder: `cd counter`.
3. Replace the contents of `src/lib.cairo` with our contract.
4. Compile the contract: `scarb build`.

If you get compilation errors similar to `Type annotations needed`, make sure your `Scarb.toml` has `starknet = "2.12.0"` added under section `[dependencies]`.

## Testing the contract

Scarb also generates a test contract after initializing a new project. The tests are written directly in Cairo and executed locally to test the actual contract logic before deploying on-chain.

To see the test, navigate to `./tests/test_contract.cairo`. Below is a break down of what’s happening in the generated test.

### Imports

![imports at the top of a cairo contract file](https://r2media.rareskills.io/CairoContract/image1.png)

1. `use starknet::ContractAddress;`

    This imports `ContractAddress` from `starknet` module.

    - Imports the `ContractAddress` type.
    - This is Starknet’s representation of a contract address and is required whenever interacting with or referencing deployed contracts.
2. `use snforge_std::{declare, ContractClassTrait, DeclareResultTrait};`

    This imports the tools needed to declare and deploy contracts during testing from starknet foundry standard library `snforge_std`.

    - `declare`: Used to declare a contract in the test environment before deployment. It’s like submitting the contract code to the network.
    - `ContractClassTrait`: Provides helper methods for interacting with declared contract classes (*such as deploying them*).
    - `DeclareResultTrait`: Exposes a function on the declaration result that retrieves the contract class (equivalent to the contract’s bytecode in Solidity).
3. `use counter::IHelloStarknetSafeDispatcher;` and `use counter::IHelloStarknetSafeDispatcherTrait;`

    This imports the safe version of the contract interface from the project name, in our case, `counter`.

    - `IHelloStarknetSafeDispatcher`: The safe dispatcher is responsible for calling the contract’s functions. But unlike Solidity where a function call just returns the value directly, here every call returns a wrapper that either contains the returned value (if successful) or an error (if it failed).

        Importantly, even if a contract call fails, execution continues within the test function. This allows the safe dispatcher to handle the error gracefully instead of reverting the entire transaction.

    - `IHelloStarknetSafeDispatcherTrait`: Exposes the callable functions in the contract for the dispatcher. Every function’s return value is wrapped, indicating it could succeed or fail.
4. `use counter::IHelloStarknetDispatcher;` and `use counter::IHelloStarknetDispatcherTrait;`

    This imports the contract interface (not the safe version) from the project name, in our case, `counter`.

    - `IHelloStarknetDispatcher`: The dispatcher also calls the contract’s functions. However, unlike the safe version, it directly returns the function’s value without any wrapper. If the target contract fails, the call immediately panics, causing execution to stop in the test function and preventing any form of graceful error handling.
    - `IHelloStarknetDispatcherTrait`: Exposes the callable functions in the contract for the dispatcher. Every functions return the raw return types of the interface

### Deploy function

![code to deploy a contract on starknet](https://r2media.rareskills.io/CairoContract/image2.png)

This function takes the contract name (in our case, `HelloStarknet`) as an argument, deploys the contract, and returns its contract address.

**Note:** the **contract name** is the identifier that comes after the `mod` keyword inside the `lib.cairo` file (`mod HelloStarknet`), whereas the **project name** (such as `counter`) is simply the folder name created when initializing the project with Scarb.

Below is a breakdown of what’s happening in the function:

- **`declare(name)`**
    - This takes the contract’s name (usually provided as a byte array) and declares it to the Starknet network.
- **`.contract_class()`**
    - Extracts the contract class from the declared contract.
- **`.deploy(@ArrayTrait::new())`**
    - Deploys the contract class.
    - `ArrayTrait::new()` is used to pass constructor arguments (here it's an empty array because the constructor takes no parameters).
    - It returns a tuple where the first element is the contract address.
- Return Value
    - The function returns the newly deployed contract’s address.

### Test cases

![a test running in the terminal](https://r2media.rareskills.io/CairoContract/image3.png)

In the screenshot above, there are two test cases:

1. `test_increase_balance`: Uses the regular dispatcher to call functions in the contract.
2. `test_cannot_increase_balance_with_zero_value`: Uses the safe dispatcher to call functions in the contract.

### Test command

Run the following command to test:

```bash
scarb test

```

## Summary of key differences and similarities

In this article, we have listed multiple similarities between Cairo and Solidity, but also various differences. For clarity, the comparisons are:

- Cairo’s `mod` keyword serves a similar role to Solidity’s `contract` keyword.
- Cairo’s interfaces are defined using a trait annotated with `#[starknet::interface]`, like Solidity’s `interface`.
- To create read-only functions in Cairo, like `view` in Solidity, pass the state as a snapshot using the `@` symbol.
- To create Solidity’s `pure`like functions in Cairo, define the function like we did with `get_five` function.
- To make a function externally callable, like with `public` and `external` in Solidity, use `#[external(v0)]` or implement it in an `impl` block with `#[abi(embed_v0)]`.

## Conclusion

Solidity and Cairo contracts serve very similar purposes. While Cairo’s syntax is different, many of the core concepts will feel familiar to Solidity developers.

The structure discussed in this article is one possible approach, but it’s not the only architectural choice provided by Starknet. In upcoming articles of this series, we’ll explore alternative designs to help you better understand the flexibility Cairo and Starknet offer for building scalable, composable smart contracts.

## Next steps

To continue learning about Cairo contracts, you are encouraged to try and play with the exercises in our [GitHub repo](https://github.com/RareSkills/Cairo-Exercises).

*This article is part of a tutorial series on [Cairo Programming on Starknet](https://rareskills.io/cairo-tutorial)*