# Access Control in Cairo

Access control defines who can call specific functions or modify contract behavior. This article explains how Cairo implements access control using the `assert` macro.

## A Review of Access Control in Solidity

In Solidity, `modifiers` are a concise way to wrap behavior around a function. They're commonly used for access control. Consider the following contract defines an `onlyOwner` modifier, which ensures that only the contract owner can call the `callMe` function:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.30;

contract SomeContract {

    address owner;

    constructor() {
        owner = msg.sender;
    }

    // THE `ONLYOWNER` MODIFIER
    modifier onlyOwner() {
        require(msg.sender == owner, "Not the owner");
        _;
    }

    function callMe() public onlyOwner {
        // callMe logic
    }
}

```

Modifiers allow you to keep your main function logic clean by moving preconditions elsewhere like we have in the `onlyOwner` modifier above.

## Cairo has no modifiers — how Cairo does access control

In Cairo, there’s no modifier keyword. Instead, we define a regular function to enforce our checks, let’s say `only_owner` and invoke it inside the `call_me` function.

The code below shows an example of how that might look:

The constructor assigns the caller’s address (`get_caller_address()` is similar to Solidity’s `msg.sender`) to the `owner` variable.

```rust
#[starknet::contract]
mod SomeContract {
    // import the required functions from the starknet core library
    use starknet::ContractAddress;
    use starknet::get_caller_address;
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    struct Storage {
        owner: ContractAddress,
    }

    #[constructor]
    fn constructor(ref self: ContractState) {
        self.owner.write(get_caller_address());
    }

    #[generate_trait]
    impl Internal of InternalTrait {
        fn only_owner(self: @ContractState) {
            let caller = get_caller_address();
            let stored_owner = self.owner.read();

            // ENSURES THE CALLER IS THE OWNER OR REVERT
            assert(caller == stored_owner, 'Not owner');
        }
    }

    #[abi(embed_v0)]
    impl SomeContractImpl of super::ISomeContract<ContractState> {
        // CALL_ME FUNCTION
        fn call_me(ref self: ContractState) {
            self.only_owner();
            // callMe logic
        }
    }
}

```

This Cairo version mirrors the Solidity pattern by restricting access to the `call_me` function. It ensures only the owner can call it by asserting that the caller’s address matches the stored owner in the contract state.

```rust
assert(caller == stored_owner, 'Not owner');

```

**The `assert` function behaves similarly to Solidity’s `require`** which halts execution and reverts the transaction if the condition fails. To make it even better,  Cairo offers another function called `assert!`, which supports formatted error messages, making it more expressive.

## `assert` Vs `assert!`

While the `assert` function and `assert!` macro (*the `!` distinguishes macros from functions*) serve the same purpose, ensuring that a condition is true, they differ in how they report errors.

`assert`:

The first argument, `condition`, is a boolean expression. If it’s `false`, the program panics with the fixed error message in **single quotes**.

```rust
assert(condition, 'static error message');

```

`assert!`:

- The first argument, `condition`, is a boolean expression. If it’s `false`, the program panics with the second argument.
- The second argument is a formatted string in **double quotes**.

```rust
assert!(condition, "Formatted error: {}", variable);

```

### What the `{}` Means in the Formatted String

In a formatted string, `{}` is a placeholder. When the code runs, the value of `variable` is converted to a string and inserted where the `{}` appears.

Think of it like a *fill-in-the-blank*:

```rust
let name = "Alice";
println!("Hello, {}", name);
// Prints: Hello, Alice

```

We can have multiple placeholders:

```rust
println!("x = {}, y = {}", x, y);

```

The order matters: each `{}` is filled by the corresponding argument after the string.

This gives developers more flexibility when debugging or handling errors. Instead of a static string, you can include runtime values in the message, something Solidity’s `require` doesn’t support directly.

> The recommended method is to use `assert!`, even in production.
>

## Supported Types in `assert!`

Not all types can be used inside the `assert!` message. Only types implementing the `core::fmt::Display` trait can be used in `assert!` message formatting. The `Display` trait defines how a type converts to a string representation when using the `{}` format specifier. These are:

- `ByteArray`
- `bool`
- `NonZero<T>` (for any `T` that itself implements `Display`)
- All integer primitives (`felt252`, `u8`, `u16`, `u32`, `u64`, `u128`, `u256`, and signed variants if present)
- `@T` (reference of any of the `Display` type above)

For example, a type like `felt252` is fine, but custom structs or types like `ContractAddress` will raise an error because they don’t implement the  `Display` trait.

If you try to do:

```rust
let caller: ContractAddress = get_caller_address();

// ❌ This will fail to compile
assert!(caller == owner, "Caller was: {}", caller);

```

You’ll see an error like:

```
Trait has no implementation in context: core::fmt::Display::<core::starknet::contract_address::ContractAddress>

```

To work around this, you can convert the address to a `felt252` if you just need the numeric representation:

```rust
let caller: ContractAddress = get_caller_address();
let caller_felt: felt252 = caller.into();

// ✅ This works, assuming the `owner` variable is of type felt252 too
assert!(caller_felt == owner, "Caller was: {}", caller_felt);

```

So while `assert!` gives you expressive error handling, keep in mind the type requirements when formatting your messages.

**Exercise:** Write a Cairo function that takes two numbers, `n` and `d`, and returns their division. If `d` is zero, the function should revert with the message: “**n is not divisible by d**” (including the actual values of `n` and `d` in the error). Hint use `assert!` function. To solve the `safe_divide` exercise, clone this [repo](https://github.com/RareSkills/Cairo-Exercises/tree/main).

*This article is part of a tutorial series on [Cairo Programming on Starknet](https://rareskills.io/cairo-tutorial)*