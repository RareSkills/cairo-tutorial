# Function Visibility in Cairo

Cairo does not have "internal" and "pure" modifiers (or any other modifiers for that matter) like Solidity does.

Recall that marking an `impl` block with `#[abi(embed_v0)]` tells Cairo to include its functions in the contract's ABI (Application Binary Interface), making them callable from outside the contract. Functions in this `impl` block must be defined in a trait that it implements (similar to interfaces in Solidity); this ensures external callers know exactly what functions are available and how to call them.

But what about functions that should not be externally callable? Cairo is capable of restricting what functions can and cannot do as well as their function visibility, just as Solidity does.

In this article, we'll show how to accomplish the equivalent of internal, private, and pure functions in Cairo.

## Internal function demonstration

Our first demonstration is an internal view function, one that can view the contract state but is not callable outside the contract.

To get started with the demo, create an empty folder called `internal_demo` and run `scarb init` inside it to initialize a new Cairo project.

Next, add a function `get_balance_2x()` inside `src/lib.cairo`, as shown below:

```rust
// IHelloStarknet INTERFACE
#[starknet::interface]
pub trait IHelloStarknet<TContractState> {
    fn increase_balance(ref self: TContractState, amount: felt252);
    fn get_balance(self: @TContractState) -> felt252;
}

#[starknet::contract]
mod HelloStarknet {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    struct Storage {
        balance: felt252,
    }

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        // ... existing functions (increase_balance, get_balance) ...

				// NEWLY ADDED FUNCTION
				// Note: This function will throw
        fn get_balance_2x(self: @ContractState) -> felt252 {
            self.balance.read() * 2
        }
    }
}
```

We get a compilation error because `get_balance_2x` is not part of the `IHelloStarknet`.

![Screenshot 2025-10-14 at 12.14.23.png](https://r2media.rareskills.io/CairoFunctionVisibility/image4.png)

As stated earlier, when implementing a trait in Cairo, the `impl` block can only contain the functions defined in that trait. Functions that are not part of the trait must be defined in separate `impl` blocks. This differs from Solidity, where contracts can freely add functions beyond those in the interface they implement.

However, we specifically don’t want to include `get_balance_2x` in the `IHelloStarknet` trait because that would make the function public.

The solution to the compilation error caused by including `get_balance_2x` inside the `HelloStarknetImpl` block (without adding it to the trait) is to:

- put `get_balance_2x` in a separate `impl`  block
- have that `impl` block use a separate trait.

Add the following code inside the `HelloStarknet` contract module, after the `HelloStarknetImpl` implementation:

```rust
// NEWLY ADDED //
    #[generate_trait]
    impl InternalFunction of IInternal {
        fn get_balance_2x(self: @ContractState) -> felt252 {
            self.balance.read() * 2
        }
    }
}
```

The complete contract is shown below with the newly added InternalFunction `impl` block highlighted in red:

![Cairo code with an internal function highlighted](https://r2media.rareskills.io/CairoFunctionVisibility/image2.png)

- The name `InternalFunction` is completely arbitrary; it can be any name that makes sense for the contract.
- Since each `impl` block needs an associated trait, we name it `IInternal` (also arbitrary).
- We don't need to explicitly create the trait for the internal `impl`. The compiler generates it automatically with the `#[generate_trait]` attribute.

Now if we try to access `get_balance_2x` from the tests (`tests/test_contract.cairo`),

```rust
#[test]
fn test_balance_2x() {
    let contract_address = deploy_contract("HelloStarknet");

    let dispatcher = IHelloStarknetDispatcher { contract_address };

    let balance_before = dispatcher.get_balance();
    assert(balance_before == 0, 'Invalid balance');

    dispatcher.increase_balance(42);

    let balance_after = dispatcher.get_balance_2x();
    assert(balance_after == 42, 'Invalid balance');
}
```

 the test will not compile, as that function is not visible publicly:

![Cairo compilation issue due to visibility mismatch](https://r2media.rareskills.io/CairoFunctionVisibility/image1.png)

To test that the internal function works as expected, we will add another function, `extern_wrap_get_balance_2x`, which will be public, then access our internal function via the `self` variable as shown below.

Don’t forget that we also need to add this function to the interface (as seen in the red box below) since we want it to be accessible outside the contract:

![Code to make a Cairo function publicly visible](https://r2media.rareskills.io/CairoFunctionVisibility/image3.png)

The `extern_wrap_balance_2x` function (blue box) calls the internal function (green box) that returns twice the current balance. Here’s the full code:

```rust
#[starknet::interface]
pub trait IHelloStarknet<TContractState> {
    fn increase_balance(ref self: TContractState, amount: felt252);
    fn get_balance(self: @TContractState) -> felt252;

    /// Retrieve 2x the balance
    fn extern_wrap_get_balance_2x(self: @TContractState) -> felt252;
}

/// Simple contract for managing balance.
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

        fn extern_wrap_get_balance_2x(self: @ContractState) -> felt252 {
            self.get_balance_2x()
        }
    }

    #[generate_trait]
    impl InternalFunction of IInternal {
        fn get_balance_2x(self: @ContractState) -> felt252 {
            self.balance.read() * 2
        }
    }
}
```

Add the following test to `tests/test_contract.cairo` below the existing tests:

```rust
#[test]
fn test_balance_2x() {
    // Deploy the HelloStarknet contract
    // Note: deploy_contract is a helper function from the test setup
    let contract_address = deploy_contract("HelloStarknet");

    // Create a dispatcher to interact with the contract
    let dispatcher = IHelloStarknetDispatcher { contract_address };

    // check initial balance is 0
    let balance_before = dispatcher.get_balance();
    assert(balance_before == 0, 'Invalid balance');

    // Increase balance by 1
    dispatcher.increase_balance(1);

    // Call the wrapper function that uses our internal function
    // Should return 1 * 2 = 2
    let balance_after_2x = dispatcher.extern_wrap_get_balance_2x();
    assert(balance_after_2x == 2, 'Invalid balance');
}
```

Run the test with `scarb test test_balance_2x`; you should see that it passes.

To sum up: internal functions in Cairo are created by defining them in a separate `impl` block without `#[abi(embed_v0)]` and using `#[generate_trait]` to auto-generate the trait that the `impl` block implements. This keeps functions callable within your contract but hidden from external callers.

## Private view and pure functions in Cairo

In Solidity, the difference between a “private” and “internal” function is that child contracts can see an “internal” function, but a “private” function can only be seen by the contract containing the function.

Cairo doesn’t have inheritance, so we have to be careful when we refer to a “private” function in Cairo.

However, a natural question arises: is it possible to “modularize” the visibility of functions? For example, in Solidity, suppose we have the following setup:

```solidity
contract A {
	function private_magic_number() private returns (uint256) {
		return 6;
	}

	function internal_mul_by_magic_number(uint256 x) internal returns (uint256) {
		return x * private_magic_number()
	}
}

contract B is A {
	function external_fun() external returns (uint256) {
		return internal_mul_by_magic_number();
	}
}
```

Contract `B` can “see” the function `internal_mul_by_magic_number()` because it inherits `A`; `B` cannot see `private_magic_number()`.

However, `external_fun()` in `B` uses `private_magic_number()` ”behind the scenes” when it calls `internal_mul_by_magic_number()`.

Let’s create an identical construct in Cairo to show how a function can be off-limits to other parts of the code, like how a private function works in Solidity.

### Using nested modules for private functions

So far, we’ve only seen `mod` (”module”) as a “container” for contract functions. However, Cairo allows us to use nested modules for further modularization. We can use this pattern to achieve similar functionality to private functions in Solidity.

Below is the default contract structure when you generate a new Scarb (snfoundry) project, but with an inner `mod` that contains the functions `internal_mul_by_magic_number()` and `private_magic_number()`.

This inner module is declared at the end of the contract, so you can scroll directly there to see the key changes:

```rust
/// Interface representing `HelloContract`.
/// This interface allows modification and retrieval of the contract balance.
#[starknet::interface]
pub trait IHelloStarknet<TContractState> {
    /// Increase contract balance.
    fn increase_balance(ref self: TContractState, amount: felt252);
    /// Retrieve contract balance.
    fn get_balance(self: @TContractState) -> felt252;
    fn get_balance_6x(self: @TContractState) -> felt252;
}

/// Simple contract for managing balance.
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

        fn get_balance_6x(self: @ContractState) -> felt252 {
            rare_library::internal_mul_by_magic_number(self.balance.read())
        }
    }

    // ~~~~~~~~~~~~~~~~~~~~~
    // ~ MOD INSERTED HERE ~
    // ~~~~~~~~~~~~~~~~~~~~~
    mod rare_library {
        pub fn internal_mul_by_magic_number(a: felt252) -> felt252 {
            a * private_magic_number()
        }

        fn private_magic_number() -> felt252 {
            6
        }
    }
}
```

**Notice that neither of the functions, `internal_mul_by_magic_number` nor `private_magic_number` access the state via `@self ContractState`, and therefore, from the Solidity perspective, are considered pure functions.**

Also note that `internal_mul_by_magic_number()` is marked `pub` but `private_magic_number()` does not have `pub`. This means that functions within `rare_library` can call `private_magic_number()`, but functions outside the module cannot. Since `internal_mul_by_magic_number` is marked with `pub`, it can be called outside the `mod`.

**Exercise**: Try calling `private_magic_number()` from the `get_balance()` function. You should get a compilation error confirming that the function is inaccessible outside its module.

Because `private_magic_number()` cannot be called by anything outside of the module `rare_library`, we can consider it a private function.

## Moving the `mod` to a separate file

Inline `mod` blocks work well for small modules, but they can clutter your contract file as the module grows. When you need multiple modules, each with their own functions, keeping everything in the main contract file makes it harder to locate specific logic.

Let's refactor our code to move the `rare_library` module into a separate file. This keeps the contract file focused on contract logic while isolating the library's module implementations. We'll continue working with the `internal_demo` project from the previous section.

**Create a separate module file**

Inside the `src/` directory, create a new file called `rare_lib.cairo` and add the following functions:

```rust
pub fn internal_mul_by_magic_number(a: felt252) -> felt252 {
    a * private_magic_number()
}

fn private_magic_number() -> felt252 {
    6
}
```

Note that since we're in a separate file, there's no need to wrap the functions with `mod` anymore; the file itself acts as the module.

**Update `src/lib.cairo`**

Now we need to update `src/lib.cairo` to use our new external module. Make the following changes:

- Declare the module at the top of  `lib.cairo`

```rust
mod rare_lib;
```

- Import the “internal” function we want to use:

```rust
use crate::rare_lib::{internal_mul_by_magic_number};
```

- Add a new function `get_balance_6x()` to the implementation:

```rust
fn get_balance_6x(self: @ContractState) -> felt252 {
       internal_mul_by_magic_number(self.balance.read())
   }
```

- Add the function to the interface trait (otherwise it won't compile):

```rust
#[starknet::interface]
   pub trait IHelloStarknet<TContractState> {
       fn increase_balance(ref self: TContractState, amount: felt252);
       fn get_balance(self: @TContractState) -> felt252;
       fn get_balance_6x(self: @TContractState) -> felt252;  // ADD THIS LINE
   }
```

- Remove the inline `rare_library` mod that we had in the previous section (since we've moved it to its own file).

Here's what the `src/lib.cairo` should look like:

```rust
mod rare_lib;
/// Interface representing 'HelloContract'.
/// This interface allows modification and retrieval of the contract balance.
#[starknet::interface]
pub trait IHelloStarknet<TContractState> {
    /// Increase contract balance.
    fn increase_balance(ref self: TContractState, amount: felt252);
    /// Retrieve contract balance.
    fn get_balance(self: @TContractState) -> felt252;
    fn get_balance_6x(self: @TContractState) -> felt252;
}

/// Simple contract for managing balance.
#[starknet::contract]
mod HelloStarknet {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};
    use crate::rare_lib::{internal_mul_by_magic_number};

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

        fn get_balance_6x(self: @ContractState) -> felt252 {
            internal_mul_by_magic_number(self.balance.read())
        }
    }
}
```

The changes are highlighted below:

![Importing a module from another file in Cairo](https://r2media.rareskills.io/CairoFunctionVisibility/image5.png)

Now, add the following test case to the test file to see that `get_balance_6x` is multiplying the balance by the magic number, even though the functions are in a separate file:

```rust
use starknet::ContractAddress;

use snforge_std::{declare, ContractClassTrait, DeclareResultTrait};

use internal_demo::IHelloStarknetDispatcher;
use internal_demo::IHelloStarknetDispatcherTrait;

fn deploy_contract(name: ByteArray) -> ContractAddress {
    let contract = declare(name).unwrap().contract_class();
    let (contract_address, _) = contract.deploy(@ArrayTrait::new()).unwrap();
    contract_address
}

#[test]
fn test_increase_balance() {
    let contract_address = deploy_contract("HelloStarknet");

    let dispatcher = IHelloStarknetDispatcher { contract_address };

    let balance_before = dispatcher.get_balance();
    assert(balance_before == 0, 'Invalid balance');

    dispatcher.increase_balance(42);

    let balance_after = dispatcher.get_balance();
    assert(balance_after == 42, 'Invalid balance');
}

// NEWLY ADDED //
#[test]
fn test_balance_x6() {
    // Deploy the HelloStarknet contract
    let contract_address = deploy_contract("HelloStarknet");

    // Create a dispatcher to interact with the contract
    let dispatcher = IHelloStarknetDispatcher { contract_address };

    // Verify initial balance is 0
    let balance_before = dispatcher.get_balance();
    assert(balance_before == 0, 'Invalid balance');

    // Increase balance by 1
    dispatcher.increase_balance(1);

    // Call get_balance_6x which uses the internal function
    // Should return 1 * 6 = 6 (multiplied by the magic number)
    let balance_after_6x = dispatcher.get_balance_6x();
    assert(balance_after_6x == 6, 'Invalid balance');
}

```

Run the test:

```rust
scarb test test_balance_6x
```

You should see that it passes, confirming that our refactored modular structure works correctly.

## Wrapping up

In this article we created:

- An internal view function: `get_balance_2x()` (can read contract state)
- An internal pure function: `internal_mul_by_magic_number()` (cannot access state)
- A private pure function: `private_magic_number()` (cannot access state)

Functions are pure when they don't take `self: @ContractState` as a parameter, meaning they cannot read or write contract storage.

**Note:** We didn't create a private function that can view the state. While technically possible by passing `self: @ContractState` to functions in nested modules, it's not a common pattern. In practice, state-viewing functions are typically kept as internal functions (in separate `impl` blocks) rather than private functions (in nested modules), since internal functions already provide sufficient encapsulation for most use cases.

## Summary

- To create internal functions, define a separate `impl` block (without `#[abi(embed_v0)]`) and add the `#[generate_trait]` attribute. This generates a trait automatically, keeping these functions internal to the contract.
- To create a pure function (one that cannot access the state), declare a `mod` inside the contract. Then create a `pub fn` inside the inner `mod`. This function will be accessible to the outer `mod`, but not anything else.
- A `mod` can be put in another file and imported. Only the `pub` functions will be visible on the outside.