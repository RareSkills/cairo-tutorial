# Integers in Cairo

Cairo doesn’t offer the full range of integer sizes found in Solidity. While Solidity provides integer types for every multiple of 8 bits up to 256, Cairo supports only the following integer types:

- `u8`
- `u16`
- `u32`
- `u64`
- `u128`
- `u256`

*For readers familiar with Rust, the `usize` type is a u32.*

Here is an example of a Cairo contract that adds two `u256` numbers together:

```rust
#[starknet::interface]
pub trait IAdd<TContractState> {
    fn add(self: @TContractState, a: u256, b: u256) -> u256;
}

#[starknet::contract]
mod Add {
    #[storage]
    struct Storage {}

    #[abi(embed_v0)]
    impl AddImpl of super::IAdd<ContractState> {
        fn add(self: @ContractState, a: u256, b: u256) -> u256 {
            a + b
        }
    }
}

```

Cairo also supports signed integers of the following type:

- `i8`
- `i16`
- `i32`
- `i64`
- `i128`

Note that `i256` is not supported.

In this article, we will cover how integers work in Cairo, highlighting the key differences from Solidity. We will go over concepts such as overflow behavior, casting between integer sizes, and working with signed and unsigned values. We will also touch on `felt252`, Cairo’s native field element that sits at the core of all number operations.

## Integers have overflow and underflow protection

Overflow protection is enabled by default in Cairo for integer (signed and unsigned) types. To see this, create a new scarb project `scarb new integers`, then delete the default contract in `./src/lib.cairo` and replace it with the following code:

```rust
#[starknet::interface]
pub trait IHelloStarknet<TContractState> {
    fn underflow(ref self: TContractState, x: u256, y: u256) -> u256;
}

#[starknet::contract]
mod HelloStarknet {

    #[storage]
    struct Storage {}

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn underflow(ref self: ContractState, x: u256, y: u256) -> u256 {
            x - y
        }
    }
}

```

Remove the automatically created tests and add the following code below. Note that the `#[should_panic]` macro above the function specifies the test passes if the execution panics.

```rust
use snforge_std::{ContractClassTrait, DeclareResultTrait, declare};
use integers::{ IHelloStarknetDispatcher, IHelloStarknetDispatcherTrait};
use starknet::ContractAddress;

fn deploy_contract(name: ByteArray) -> ContractAddress {
    let contract = declare(name).unwrap().contract_class();
    let (contract_address, _) = contract.deploy(@ArrayTrait::new()).unwrap();
    contract_address
}

#[test]
#[should_panic]
fn test_flow_protection() {
    let contract_address = deploy_contract("HelloStarknet");

    let dispatcher = IHelloStarknetDispatcher { contract_address };

    dispatcher.underflow(0, 1);
}

```

Run the tests with `scarb test` and note that the test passes.

## No floating points

Like Solidity, Cairo does not support floating points

## Casting integers

There are two types of casts:

- casts that are guaranteed to succeed and
- ones that might fail.

For example, casting a `u8` to a `u16` will always succeed because a `u16` can hold any number that a `u8` can. However, casting from `u16` to `u8` might fail because some of the most significant bits might be truncated.

Casting from `i16` to `u16` can fail if the `i16` holds a negative value. Casting from `u16` to `i16` can fail if the number held in the `u16` is too large — unsigned integers of the same bit size as signed integers can hold larger positive numbers than the signed integer can.

### Casting that always succeed

Converting to a larger type will always succeed because the target type can hold any value from the source type.

Example of cast that always succeeds:

- `u8` → `u16`, `u32`, `u64`, `u128`, `u256`
- `u16` → `u32`, `u64`, `u128`, `u256`
- `i8` → `i16`, `i32`, `i64`, `i128`
- `i16` → `i32`, `i64`, `i128`

To perform a cast that always succeeds, use `.into()`

```rust
let small: u8 = 7;
let large: u16 = small.into(); // Always succeeds - u16 can hold any u8 value

```

### Casting that may fail

Unlike Solidity which silently truncates the most significant bits when a larger number is casted to a smaller type, Cairo will panic if a cast cannot be performed safely.

Here is the code snippet for a cast that may fail:

```rust
// may fail if value is too large
let large: u16 = 300;
let small: u8 = large.try_into().unwrap();  // Panics! 300 > 255

```

The code above panics because the value `300` does not fit into a `u8`, which can only store values up to `255`. For now, just remember that `unwrap()` method extracts the value out of a “wrapper” type (*discussed in the next sub-section*). If the wrapper contains an error instead of a valid value, `unwrap()` will panic.

Note that if you try to use the `into()` cast in a situation where the cast can fail (casting from  large to small type), the code will not compile.

### Detecting if a cast will fail

When converting between integer types using `try_into()`, the result may or may not fit into the target type. Because of this, the conversion returns an `Option` (*an example of a “wrapper” type*), which is Cairo’s way of saying *“this might succeed, or it might fail.”*

- If the conversion works, you get `Option::Some(value)`.
- If it doesn’t fit, you get `Option::None`.

This forces us to handle the possible failure safely before using the value. Two common ways to do this are:

- using the `.is_some()` method which returns bool indicating whether the option has some value or not.
- using `if let`.

Using `.is_some()` method:

```rust
// Value 300 cannot fit into u8 (max 255), so try_into returns None
let value: u16 = 300;
let result_option: Option<u8> = value.try_into();

if result_option.is_some() {
	// cast succeeded
} else {
	// cast failed
}

```

Using `if let`:

```rust
if let Some(result) = result_option {
    // cast succeeded, use `result`
} else {
    // cast failed
}

```

## Constants

Constants in Cairo are values that are known at compile time and cannot be changed at runtime. They are declared inside the mod block using the `const` keyword and must have their type explicitly specified, like so:

```rust
const <NAME>: <Type> = <value>;

```

Here's how to declare and use constants in Cairo:

```rust
#[starknet::contract]
mod HelloStarknet {
		// DECLARE CONSTANTS
    const num_one: u256 = 1;
    const num_two: i8 = -2;

    #[storage]
    struct Storage {}

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn get_one(self: @ContractState) -> u256 {
        		// USE CONSTANTS
            num_one
        }

        fn get_two(self: @ContractState) -> i8 {
        		// USE CONSTANTS
            num_two
        }
    }
}

```

### Constants vs Immutables

Unlike Solidity, Cairo doesn't have a separate `immutable` keyword. Values that need to be set once during contract deployment but aren't known at compile time should be stored in contract storage and set in the constructor.

## Max integer sizes

In Solidity, `type(uint256).max` is used to get the max size of an integer. In Cairo, we use `let max_u256: u256 = Bounded::MAX` as shown below:

```rust
#[starknet::contract]
mod HelloStarknet {
    use core::num::traits::{Bounded}; // Bounded is how we get the max

    #[storage]
    struct Storage {} // unusued

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn max_demo(ref self: ContractState) -> u256 {
          let max_u256: u256 = Bounded::MAX;
          max_u256
        }
    }
}
```

In the above code, the contract first imports the Bounded trait, which provides access to numeric constants: `MIN` and `MAX` . The 
`MAX` returns you maximum value allowed for the type. Inside the `max_demo()` function, we use `Bounded::MAX` to retrieve the maximum value of the `u256` type.

Most of the time, Cairo is able to determine types on its own, but when using `Bounded::MAX` the compiler won’t automatically know which integer type you need the max for. Hence, the variable needs an explicit type annotation, which is the `u256` after `:` i.e. `let max_u256: u256 = Bounded::MAX;`.

## Min integer sizes

Just as we can get the maximum value for integer types, Cairo also provides access to  their minimum values through the `Bounded` trait with `Bounded::MIN` as shown below:

```rust
#[starknet::contract]
mod HelloStarknet {
    use core::num::traits::{Bounded}; // Bounded provides both MIN and MAX

    #[storage]
    struct Storage {} // unused

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn min_demo(ref self: ContractState) -> (u256, i128) {
		        // This will be 0 for unsigned types
            let min_u256: u256 = Bounded::MIN;

						// This will be the most negative value
            let min_i128: i128 = Bounded::MIN;

            (min_u256, min_i128)
        }
    }
}

```

### Understanding Min sizes

For unsigned integer types (u8, u16, u32, u64, u128, u256), the minimum value is always `0`:

```rust
let min_u8: u8 = Bounded::MIN;     // 0
let min_u16: u16 = Bounded::MIN;   // 0
let min_u32: u32 = Bounded::MIN;   // 0
let min_u64: u64 = Bounded::MIN;   // 0
let min_u128: u128 = Bounded::MIN; // 0
let min_u256: u256 = Bounded::MIN; // 0

```

For signed integer types (i8, i16, i32, i64, i128), the minimum value is the most negative number that can be represented:

```rust
let min_i8: i8 = Bounded::MIN;     // -128
let min_i16: i16 = Bounded::MIN;   // -32,768
let min_i32: i32 = Bounded::MIN;   // -2,147,483,648
let min_i64: i64 = Bounded::MIN;   // -9,223,372,036,854,775,808
let min_i128: i128 = Bounded::MIN; // a very large negative value

```

### Type annotation requirement

Just like with `Bounded::MAX`, the compiler cannot automatically guess the type when using `Bounded::MIN`, so explicit type annotations are required:

```rust
// This won't compile - ambiguous type ❌
let min_val = Bounded::MIN;

// This will compile - explicit type annotation ✅
let min_val: u64 = Bounded::MIN;

```

## Shorthand for specifying types on integer literals

If we assign a fixed value to an integer, we can specify the type of the integer explicitly or allow the compiler to infer the type.

Specifying the type:

```rust
// first way
let x: i32 = 10;

// second way
let y = 10_i32;

```

Allowing the compiler to infer the type:

If we do not specify the type, the compiler will try to infer it from the context. For example, the following function returns a `u32` so the type of 10 is `u32`:

```rust
fn hello_world() -> u32 {
		let x = 10;
		x
}

```

## Signed integer division overflow

In Solidity, there is a specific edge case with signed integer division that can cause unexpected behavior. Consider this Solidity contract:

```solidity
contract D {
    function div(int8 a, int8 b) public pure returns (int8 c) {
        c = a / b;
    }
}

```

The issue occurs when you divide the most negative value by -1. For `int8`, the range is -128 to 127. When you perform `-128 / -1`, mathematically the result should be `128`, but `128` cannot fit in an `int8` (which has a maximum value of 127). This causes an overflow.

In Solidity, this operation would either:

- Wrap around to an unexpected value
- Revert (in newer versions with overflow protection)

### How Cairo handles integer division overflow

Just like in Solidity version ≥ 0.8, Cairo provides built-in overflow protection. If an operation would result in an overflow, the program will panic at runtime, preventing unintended behavior.

```rust
#[starknet::contract]
mod Div {
    #[storage]
    struct Storage {}

    #[abi(embed_v0)]
    impl DivImpl of super::IDiv<ContractState> {
        fn div(self: @ContractState, a: i8, b: i8) -> i8 {
            // This will panic if `a` is -128 and `b` is -1
            a / b
        }
    }
}

```

To prevent a panic from signed division overflow, we need to manually check conditions before performing the operation, like so:

```rust
#[starknet::contract]
mod Div {
    use core::num::traits::Bounded;

    #[storage]
    struct Storage {}

    #[abi(embed_v0)]
    impl DivImpl of super::IDiv<ContractState> {
        fn div(self: @ContractState, a: i8, b: i8) -> i8 {
            if b == 0 {
                // Division by zero
            } else if a == Bounded::<i8>::MIN && b == -1 {
                // Overflow case
            } else {
                a / b
            }
        }
    }
}

```

## Casting Up Failure

This Solidity function looks safe but can produce unexpected results:

```solidity
function mul(uint8 a, uint8 b) public pure returns (uint256 c) {
    c = a * b;
}

```

The issue is that the multiplication `a * b` happens in `uint8` arithmetic first, then the result is cast to `uint256`. If `a * b` overflows the `uint8` range (0-255), the multiplication wraps around before being cast up.

For example:

- `mul(200, 200)` should mathematically return `40000`
- But `200 * 200 = 40000` overflows `uint8` (max 255)
- The wrapped result in `uint8` would be `40000 % 256 = 64`
- Then `64` gets cast to `uint256`, returning `64` instead of `40000` , of course this happens in Solidity versions less than 0.8

### Overflow in Cairo

Cairo handles casting up overflow issue through its built-in overflow protection. That is, if an operation produces a value that goes beyond the allowed range, Cairo will throw an error and halt execution instead of allowing unintended behavior.

For example, the code below will panic if `a * b > 255`:

```rust
// This will panic if the multiplication overflows u8
fn mul(self: @ContractState, a: u8, b: u8) -> u256 {
    let result_u8 = a * b; // Panic if a * b > 255
		result_u8.into()       // This line never executes if overflow occurs
}

```

### Safe Casting Up

A safe approach to avoid our Cairo contract from panicking due to overflow is to cast up before arithmetic operations when we need the result in a larger type. For example, we can cast from u8 to u256:

```rust
// cast up before multiplication
fn safe_mul(self: @ContractState, a: u8, b: u8) -> u256 {
    let a_wide: u256 = a.into();
    let b_wide: u256 = b.into();
    a_wide * b_wide // No overflow possible
}

```

## Exponents

In Solidity, the syntax for exponents is `b ** e` where `b` is the base and `e` is the exponent.

In Cairo, you must import `Pow` with `use core::num::traits::Pow;`. Then, you can raise an integer to a power using `b.pow(e)`.

```rust

#[starknet::contract]
mod HelloStarknet {

	  use core::num::traits::Pow; // THIS IMPORT IS REQUIRED

    #[storage]
    struct Storage {}

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn upcast_demo(ref self: ContractState, x: u256, y: u32) -> u256 {
			      x.pow(y) // compute exponent
        }
    }
}

```

The `.pow()` method returns a value of the same type as the base, so `x.pow(y)` here produces a value of type (`u256`).

**Important:** The exponent must be of type `u32` (or `usize` which is a `u32` under the hood in Cairo). The code will not compile if another integer type is used.

## Underscores in literals

Like Solidity, large numbers in Cairo can be broken up with underscores to make them easier to read:

```rust
// valid Cairo
let basis_points = 10_000;
```

## Scientific Notation Shorthand

In Solidity, you can write numbers using scientific notation such as `10e18`, which represents $10 \times 10^{18}$. Cairo does not support this. To write `10e18` in Cairo, use the Pow trait as shown below:

```rust
use core::num::traits::Pow;

// ...

let num = 10_u256.pow(18_u32);
```

Remember, the exponent must be of type `u32`.

## Bitwise operations, Shifting Operations, and Comparisons

### Bitwise Operations

Cairo supports standard bitwise operations on integer types:

**Bitwise AND (`&`):**

```rust
let a: u8 = 0b1100; // 12 in decimal
let b: u8 = 0b1010; // 10 in decimal
let result = a & b; // 0b1100 & 0b1010 = 0b1000 => 8
```

**Bitwise OR (`|`):**

```rust
let a: u8 = 0b1100; // 12 in decimal
let b: u8 = 0b1010; // 10 in decimal
let result = a | b; // 0b1110 = 14
```

**Bitwise XOR (`^`):**

```rust
let a: u8 = 0b1100; // 12 in decimal
let b: u8 = 0b1010; // 10 in decimal
let result = a ^ b; // 0b0110 = 6
```

**Bitwise NOT (`~`):**

```rust
let a: u8 = 0b1100; // 12 in decimal
let result = ~a;    // 0b11110011 = 243 (inverts all bits)

```

### Shifting Operations

Cairo provides left and right bit shifting operations:

### Left Shift (`<<`)

Shifts bits to the left, filling with zeros:

```rust
let a: u8 = 0b0001; // 1 in decimal
let result = a << 3; // 0b1000 = 8 (multiplies by 2**3)

```

### Right Shift (`>>`)

Shifts bits to the right:

```rust
let a: u8 = 0b1100; // 12 in decimal
let result = a >> 2; // 0b0011 = 3 (divides by 2**2)

```

### Comparison Operations

Cairo supports all standard comparison operators:

### Equality (`==` and `!=`)

```rust
let a: u32 = 10;
let b: u32 = 20;
let equal = a == b;     // false
let not_equal = a != b; // true

```

### Ordering (`<`, `<=`, `>`, `>=`)

```rust
let a: u32 = 10;
let b: u32 = 20;
let less_than = a < b;           // true
let less_or_equal = a <= b;      // true
let greater_than = a > b;        // false
let greater_or_equal = a >= b;   // false

```

## A note about felt252

If you read older production Cairo code, you will see the datatype `felt252` used frequently. Similar to how the EVM has a default word size of 256 bits, the CairoVM has a default word size of a little under 252 bits, or to be precise: `3618502788666131213697322783095070105623107215331596699973092056135872020481` or 2²⁵¹+17⋅2¹⁹²+1. The number is slightly smaller than 2²⁵².

Cairo refers to number types that fall in the range of [0..2²⁵¹+17⋅2¹⁹²+1] as `felt252`.

This large number is a prime number that is optimized for zero-knowledge proof math on the Cairo virtual machine.

The name `felt252` comes from the term “field element that fits in 252 bits.” A “field element” is a number that lives in a number system where all addition and multiplication are done modulo some prime number.

Using `felt252` in Cairo code is not recommended because at a later date, the CairoVM may change its default word size to a smaller value to improve the speed at which it can prove transactions.

The Cairo compiler will seamlessly handle the translation of integers (`u8`... `u256`) to `felt252` behind the scenes for you. It is worth noting that a `u256` doesn’t fit in 252 bits, so behind the scenes, a `u256` is really two `felt252` elements. Thus, it is preferable for gas-efficiency reasons to use `u128` or smaller integers where possible. **The only other time using `felt252` makes sense is where extreme optimizations are necessary.** We will revisit gas costs on Starknet in a later tutorial. For now, we recommend that you not use the `felt252` type and simply use integers.

However, because you will see `felt252` frequently in code, it’s worth explaining how it works.

### felt252 has no overflow and underflow protection

Unlike Solidity 0.8.0 or higher, Cairo does not built in overflow and underflow protection for `felt252`. To demonstrate this, create a new project `scarb new numbers`. Then replace the generated code in `lib.cairo` with the following code:

```rust
#[starknet::interface]
pub trait IHelloStarknet<TContractState> {
    fn math_demo(self: @TContractState, x: felt252, y: felt252) -> felt252;
}

#[starknet::contract]
mod HelloStarknet {

    #[storage]
    struct Storage {}

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn math_demo(self: @ContractState, x: felt252, y: felt252) -> felt252 {
            x - y
        }
    }
}

```

Replace the test as follows:

```rust
use starknet::ContractAddress;

use snforge_std::{declare, ContractClassTrait, DeclareResultTrait};
use numbers::IHelloStarknetDispatcher;
use numbers::IHelloStarknetDispatcherTrait;

fn deploy_contract(name: ByteArray) -> ContractAddress {
    let contract = declare(name).unwrap().contract_class();
    let (contract_address, _) = contract.deploy(@ArrayTrait::new()).unwrap();
    contract_address
}

#[test]
fn test_math_demo() {
    let contract_address = deploy_contract("HelloStarknet");
    let dispatcher = IHelloStarknetDispatcher { contract_address };
    let result = dispatcher.math_demo(0, 1); // 0 - 1
    println!("result: {}", result);
}

```

The console will print:

```rust
result: 3618502788666131213697322783095070105623107215331596699973092056135872020480

```

In Solidity lower than 0.8.0, assuming unsigned arithmetic, `0 - 1` results in an underflow. The value then wraps around to the maximum possible `uint256` value. A similar thing happens with `felt252` in Cairo, since it has no overflow or underflow protection. All arithmetic is performed modulo the field prime (`2²⁵¹ + 17 × 2¹⁹² + 1`), so subtracting 1 from 0 returns the largest valid `felt252` value, which looks like a large number.

## felt252_div

If you try to divide a `felt252` by another `felt252` you will get a compilation error. The following code will not compile:

```rust
fn math_demo(self: @ContractState, x: felt252, y: felt252) -> felt252 {
    x / y
}

```

To *fully* understand why Cairo doesn’t allow for division like this, watch our video on [modular arithmetic](https://www.youtube.com/watch?v=0lEwgX_zKqE).

### The Correct Way to Divide `felt252`

To perform division with `felt252` values, we must use the `felt252_div` which is a built-in function in Cairo's core library:

```rust
#[starknet::contract]
mod HelloStarknet {

		// THIS IS NEW
		use core::felt252_div;

    #[storage]
    struct Storage {}

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn math_demo(self: @ContractState) -> felt252 {
            felt252_div(4, 2)
        }
    }
}

```

The `felt252_div` function doesn't perform regular division. Instead, it:

1. Finds the modular inverse of the divisor `y` modulo the field prime
2. Multiplies `x` by this inverse
3. Returns the result modulo the field prime

Mathematically:

```rust
felt252_div(x, y) = x * y^(-1) mod P

```

Where `y^(-1)` is the modular inverse of `y` in the finite field.

### Division by Zero

Attempting `felt252_div(x, 0)` will cause a runtime panic:

```rust
*// This will panic!*
let result = felt252_div(42, 0);

```

Always validate your divisor before performing division. A way `felt252_div` ensures `felt252` value cannot be zero is by using `NonZero<felt252>`.

### NonZero<felt252>

The `felt252_div` function requires its second argument (the divisor) to be of type `NonZero<felt252>` rather than plain `felt252`. This prevents division by zero at compile time.

```rust
// BE SURE TO CHANGE THE TRAIT DEFINITION ALSO

#[starknet::contract]
mod HelloStarknet {

    use core::felt252_div;

    #[storage]
    struct Storage {}

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {

		    // NOTE THE TYPE OF `y`
        fn underflow_demo(self: @ContractState, x: felt252, y: NonZero<felt252>) -> felt252 {
            felt252_div(x, y)
        }
    }
}

```

## Summary

Cairo integers are safe. All `u*` and `i*` types have overflow protection and panic on invalid operations.

Casting is strict: `.into()` is safe (*always succeed when converting to a larger type*), `.try_into()` checks for errors (*may panic if the target type cannot hold the value*).

Exponentiation uses `.pow()` via a trait import.

Bitwise and comparison operators work normally across integer types.

`felt252` is Cairo’s native field element with no overflow checks like integers. Division on felt requires a function (`felt252_div`) with zero-checking.

*This article is part of a tutorial series on [Cairo Programming on Starknet](https://rareskills.io/cairo-tutorial)*