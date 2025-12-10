# Type Casting

Type casting in Cairo is the process of converting values from one data type to another.

This becomes necessary when working with Cairo's strict type system, where explicit type matching is required for function calls, variable assignments, contract interactions, and data operations.

Cairo takes a more cautious approach to handling type conversions than Solidity. It provides clear ways to handle cases where casting might fail, rather than allowing conversions that can silently cause unintended errors like data truncation.

To illustrate the difference in approach between both languages, consider converting a `uint32`to a `uint8` type:

**Solidity example:**

```solidity
contract Example {
    function castingExample() public pure returns (uint8) {
        uint32 largeNumber = 1000;
        uint8 smallNumber = uint8(largeNumber);
        return smallNumber;
    }
}
```

In Solidity, casting silently truncates `largeNumber` (1000) to `smallNumber` (232) using modulo arithmetic (1000 % 2**8 = 232). The `castingExample()` function executes successfully and returns 232 without any warning that data loss occurred.

**Cairo example:**

```rust
fn casting_example() -> u8 {
    let large_number: u32 = 1000;
    let small_number: u8 = large_number.try_into().unwrap();
    println!("Small number is : {}", small_number);
    small_number
}

fn main(){
    casting_example();
}
```

At runtime, `try_into()` checks whether `large_number` (1000) can fit into a `u8` (maximum value 255). When the conversion fails, it returns `None`, and the subsequent `unwrap()` call panics with `"Option::unwrap failed."`, immediately stopping execution instead of allowing data corruption.

The `try_into().unwrap()` syntax is covered in detail in the next section.

## Difference Between Guaranteed Conversions and Conversions that Could Fail

Cairo handles type casting through two main traits: `Into` for safe conversions and `TryInto` for conversions that might fail.

### `Into` Trait: Infallible type casting

The `Into` trait is used for conversions that are guaranteed to succeed. These are "*safe*" conversions where the target type can always represent any value from the source type. This trait is automatically available in every Cairo program without needing an import statement.

Here's an example showing conversions from a smaller to larger integer types, and from integers to Cairo's native `felt252` type:

```rust
fn main() {
    let small_number: u8 = 100;
    let large_number: u32 = small_number.into();

    let num: u16 = 500;
    let as_felt: felt252 = num.into();
}
```

It's clear here that `small_number` of type  `u8` or any type smaller than `u32` can fit into `larger_number` , and `felt252` can contain any value from all `uint` types except `u256`(which is larger than `felt252`).

When we use `.into()`, we're telling the Cairo compiler, "we know this conversion will always work, so just do it." In the examples above, converting `small_number` from `u8` to `u32` and `num` from `u16` to `felt252` is guaranteed to succeed because the target types can hold any value from the source types.

However, if we attempt to use `.into()` to convert from a larger type to a smaller type, the Cairo compiler will throw an error:

```rust
fn main() {
    let large_number: u256 = 100;
    let small_number: u32 = large_number.into(); // ERROR: Trait has no implementation in context: core::traits::Into::<core::integer::u256, core::integer::u32>
}
```

This error occurs because Cairo's `Into` trait is only implemented for safe conversions where the target type can hold all possible values of the source type. Since `u256` has a much larger value range than `u32`, there is no `Into` implementation for this conversion, even though the value 100 would fit in a `u32`.

### `TryInto` Trait: Fallible type casting

The `TryInto` trait is the right method to use for conversions that might fail if the source value range doesn't fit in the target type. Like Into, this trait is also automatically available without needing an import statement.

`TryInto` returns an `Option` enum, which can either be `Some(converted_value)` if the conversion was successful, or `None` if it failed.

Building on the earlier example where 1000 (`u32`) couldn't safely convert to `u8`, the `try_convert_to_u8` function below shows Cairo's approach to potentially unsafe type conversions. This function takes a `u32` value and attempts to convert it to `u8`. The code shows both successful conversions (when values fit) and failed conversions (when values are too large for `u8`'s 0-255 range):

```rust
fn try_convert_to_u8(num: u32) {
    // Attempt to convert u32 to u8 (returns Option<u8>)
    let result: Option<u8> = num.try_into();

    // Use 'match' to handle both success and failure cases
    // 'match' is Cairo's pattern matching (like a switch statement that checks what's inside Option)

    match result {
        Option::Some(val) => {
           // Conversion succeeded (val contains the converted u8 value)
           println!("Successfully converted {} to u8: {}", num, val);
        },
        Option::None => {
           // Conversion failed (number was too large for u8 (which holds 0-255))
           println!("Conversion failed! {} is too big for u8 (max: 255)", num);
        }
    }
}

fn main() {
    try_convert_to_u8(1000);  // Will fail (too large for u8)
    try_convert_to_u8(100);   // Will succeed (fits in u8)
    try_convert_to_u8(255);   // Will succeed (maximum u8 value)
    try_convert_to_u8(256);   // Will fail (exceeds u8 maximum by 1)
}
```

Inside the `try_convert_to_u8` function, we call `try_into()` on the input `num` value, which returns an `Option` type that we store in `result`

The `match` statement is Cairo's pattern matching feature, similar to a switch statement in other languages. It checks what's inside the `Option` and executes a specific block of code depending on whether the result has a value or is empty. If the conversion succeeded, we get `Some(val)` containing the converted value, and we print a success message. If the conversion failed because the number was too large, we get `None`, and we print an error message explaining the failure.

In the `main()` function, we test four different scenarios to demonstrate how the conversion behaves with various input values.

When we call `try_convert_to_u8(1000)`, the diagram below shows how the conversion returns `None` in `Option<u8>` because 1000 exceeds the maximum value `u8` (255):

![Visual diagram of an Option containing a None](https://r2media.rareskills.io/CairoTypeCasting/image2.png)

Since the conversion returned `None`, the `match` statement detects that it’s empty and executes the `Option::None` branch, printing the error message **"Conversion failed! 1000 is too big for u8 (max: 255).”**

Subsequently, when `try_convert_to_u8(100)` runs, the diagram below shows how the conversion returns `Some(100)` in `Option<u8>` because 100 fits within the valid range of 0-255:

![Visual diagram of an Option containing a Some](https://r2media.rareskills.io/CairoTypeCasting/image1.png)

Since the conversion succeeded, the `match` statement executes the `Option::Some(val)` branch, printing **"Successfully converted 100 to u8: 100."**

### When to use `into()` or `try_into()`

**Use `into()` for:**

- Converting smaller types to larger ones (e.g., `u32` to `u64`)
- Any conversion where data loss is impossible

> `into()` explicitly disallows casting from larger to smaller types, even if the specific value would fit in the target range.
>

**Use `try_into()` for:**

- Converting larger types to smaller ones (`u32` to `u8`)
- Any conversion where the value might not fit
- When you want to handle conversion failures gracefully instead of panicking, use `try_into()` with `match`

With an understanding of Cairo's casting mechanisms, we can now look into how these traits apply to specific conversion scenarios.

## felt252 to uints

Converting `felt252` to unsigned integer types is a common operation in Cairo since `felt252` is the native type. The conversion approach depends on whether we’re converting to a larger or smaller integer type.

Here's an example that converts a `felt252` value (felt_value) to `u256` (as_u256):

```rust
fn main() {
    let felt_value: felt252 = 42615;
    let as_u256: u256 = felt_value.into();
}
```

This conversion is guaranteed to succeed since `u256` can hold any `felt252` value, which is why we can use `.into()` instead of `.try_into()`.

Converting `felt252` to smaller integer types requires `try_into()` since the `felt252` value might exceed the target type's range.

This function attempts to convert `felt252` to `u8`, but will panic if the value is larger than 255:

```rust
fn convert_felt_to_small_uint(felt_value: felt252) -> u8 {
    felt_value.try_into().unwrap()
}
```

`convert_felt_to_small_uint` function takes a `felt252` value, tries to convert it to `u8`, panics if it fails. It uses `.unwrap()` to extract the `u8` value from the `Option` returned by `try_into()`. If the `felt252` value exceeds 255, the conversion returns `None` and `.unwrap()` will cause the program to panic.

A safer way is to handle potential conversion failures explicitly without panicking. The following code shows proper error handling by creating a function that returns `Option<u8>` instead of panicking, then testing both a successful conversion (100) and a failed conversion (1000), with `match` statements to handle each outcome appropriately:

```rust
fn safe_convert_felt_to_u8(felt_value: felt252) -> Option<u8> {
    felt_value.try_into()
}

fn main() {
    let small_felt: felt252 = 100;
    let large_felt: felt252 = 1000;

    let small_as_u8 = safe_convert_felt_to_u8(small_felt); // Returns Some(100)
    println!("Small conversion result: {:?}", small_as_u8);

    let large_as_u8 = safe_convert_felt_to_u8(large_felt); // Returns None

    // handle the successful conversion
    match small_as_u8 {
        Option::Some(val) => println!("Successfully converted 100 to u8: {}", val),
        Option::None => println!("Small conversion failed"),
    }

    // handle the failed conversion
    match large_as_u8 {
        Option::Some(val) => println!("Converted: {}", val),
        Option::None => println!("Conversion failed: 1000 is too large for u8"),
    }
}
```

`safe_convert_felt_to_u8` takes a `felt252` value and returns `Option<u8>`. Notice it doesn't use `.unwrap()`; it returns the `Option` directly from `try_into()`, letting the caller decide how to handle potential failures.

In the main function, we test two scenarios:

- **Converting 100 to `u8`**: This succeeds because 100 fits within `u8`'s range (0-255), so `small_as_u8` contains `Some(100)`
- **Converting 1000 to `u8`**: This fails because 1000 exceeds `u8`'s maximum value of 255, so `large_as_u8` contains `None`

The first `match` statement handles the successful conversion. Since `small_as_u8` contains `Some(100)`, it matches the `Option::Some(val)` branch and prints the success message with the converted value.

The second `match` statement handles the failed conversion. Since `large_as_u8` contains `None`, it matches the `Option::None` branch and prints an error message explaining why the conversion failed.

This shows how to handle both successful and failed conversions gracefully without panicking, giving us full control over error handling in our program.

## uints to Address Type

Cairo doesn't allow direct conversion from integer types to addresses. Instead, conversions must go through `felt252` as an intermediate type. Converting `uints` to addresses becomes necessary when working with user IDs, numeric identifiers, or when deriving addresses from integer calculations in smart contracts.

The conversion process involves two steps: first converting the integer to `felt252`, then converting the `felt252` to `ContractAddress`:

```rust
use starknet::ContractAddress;

fn user_address(user_id: u64) -> ContractAddress {
    let address_felt: felt252 = user_id.into();
    address_felt.try_into().unwrap()
}
```

The `user_address` function accepts a `u64` parameter user_id. It first converts the `u64` value to `felt252` using `.into()` which always succeeds because `felt252` can hold any `u64` value and store it in address_felt. Then it converts `felt252` address_felt to `ContractAddress` using `.try_into().unwrap()`.

We use `.try_into()` because it's the only available conversion method from `felt252` to `ContractAddress`. The `.unwrap()` extracts the result but will panic if the conversion fails.

In the code example above, the `try_into()` method returns `Option<ContractAddress>`; either `Some(address)` if conversion succeeds or `None` if it fails. The `.unwrap()` method extracts the actual `ContractAddress` value from the `Some()`, but will panic if the conversion failed and returned `None`.

Converting `u256` requires extra caution since `u256` values can exceed the `felt252` range, and `ContractAddress` has an even smaller valid range of `[0, 2**251)`. This means both conversion steps can potentially fail.

For cases where we are confident the value will fit, say when value is within the `felt252` range, we can use the direct approach:

```rust
fn convert_u256_to_address(value: u256) -> ContractAddress {
    // First step: convert u256 to felt252 (will panic if value exceeds felt252 range)
    let address_felt: felt252 = value.try_into().unwrap();
    // Second step: convert felt252 to ContractAddress (will panic if outside valid address range)
    address_felt.try_into().unwrap()
}
```

When dealing with arbitrary `u256` values that might be outside the valid range, it's better to handle potential failures explicitly:

```rust
fn safe_convert_u256_to_address(value: u256) -> Option<ContractAddress> {
    // First step: try to convert u256 to felt252 (might fail if value is too large)
    match value.try_into() {
        Option::Some(felt_val) => {
            // u256 to felt252 conversion succeeded
            let address_felt: felt252 = felt_val;
            // Second step: try to convert felt252 to ContractAddress
            // This can also fail if the felt252 value is outside valid address range
            address_felt.try_into()
        }
        Option::None => {
            // u256 to felt252 conversion failed (value too large for felt252)
            Option::None
        }
    }
}
```

Instead of panicking on conversion failure, `safe_convert_u256_to_address` returns `None` for inputs that exceed `ContractAddress` range, allowing the caller to handle oversized inputs gracefully.

## ByteArray and String Casting

Converting between string representations and numeric types in Cairo involves working with Cairo's string types: short strings (which are `felt252`) and `ByteArray` for longer strings.

### Short strings

```rust
fn main() {
    let string_as_felt: felt252 = 'Hello';
    let hex_representation: felt252 = 0x48656c6c6f;

    println!("String: {}", string_as_felt);
    println!("Hex form: {}", hex_representation);
}
```

When this code runs, you'll see that short strings in Cairo are directly stored as `felt252` values.

![Screenshot of terminal output showing equivalency between underlying string and felt values](https://r2media.rareskills.io/CairoTypeCasting/image3.png)

There's no conversion happening; `'Hello'` and its hexadecimal equivalent `0x48656c6c6f` are the same `felt252` value.

### Short strings to uints

Here's an example that converts a character (stored as `felt252`) to its ASCII value as a `u8`:

```rust
fn main() {
    let char_a: felt252 = 'A';
    let char_as_u8: u8 = char_a.try_into().unwrap();

    println!("Character 'A' as u8: {}", char_as_u8);
}
```

When this code runs, it displays **"Character 'A' as `u8`: 65"** in the terminal because the character value fits into the target type. If the value doesn't fit, the program will panic, so handling cases when the conversion success is uncertain is important.

### ByteArray Operations

To convert `ByteArray` data, individual bytes are accessed through indexing and each byte is converted separately.

*The `ByteArray` type is primarily for handling strings longer than 31 bytes, not for numeric type conversions.*

**Note that accessing individual bytes in `ByteArray` returns `u8`**. When working with individual characters from a `ByteArray` as numeric values is needed, they can be extracted by index and converted to other numeric types like `u32` or `felt252`. Consider the following example:

```rust
fn main() {
    let text: ByteArray = "Cairo";

    let first_byte = text[0];
    let third_byte = text[2];

    let byte_as_u32: u32 = first_byte.into();
    let byte_as_felt: felt252 = third_byte.into();

    println!("First byte 'C' as u32: {}", byte_as_u32);
    println!("Third byte 'i' as felt252: {}", byte_as_felt);
}
```

The `ByteArray` "Cairo" is accessed where the first character, 'C,' is accessed using `text[0]`, which returns a `u8` value. It is then converted to `u32` using `.into()`, which is guaranteed to succeed since `u32` can hold any `u8` value. Similarly, the third character 'i' at `text[2]` is converted to `felt252`.

This way, we work with `ByteArray` content at the byte level and convert individual characters to the numeric types you need.

Most of the time, if we have a `ByteArray`, we probably want to keep it as string data since `ByteArray` is optimized for storing and manipulating text rather than numeric computation.

## bool casting

Cairo keeps boolean values (`true`/`false`) separate from numeric types by design but allows converting boolean values to `felt252` using `.into()`, where `true` becomes 1 and `false` becomes 0. However, booleans cannot be directly converted to integer types like `u32`, `u64`, etc. using automatic casting methods:

```rust
fn main() {
    let flag: bool = true;

    // This works, bool to felt252:
    let as_felt: felt252 = flag.into(); // Works: true becomes 1, false becomes 0

    // This will cause a compilation error:
    // let as_u32: u32 = flag.into(); // ERROR: Trait has no implementation in context: core::traits::Into::<core::bool, core::integer::u32>.

    // Manual conversion for integers:
    let as_u32: u32 = if flag {
        1
    } else {
        0
    };
}
```

Numbers cannot be cast back to booleans automatically. Explicit comparisons are required as shown below:

```rust
fn main() {
    let number: u32 = 1;
    let felt_num: felt252 = 1;

    // These will cause compilation errors:
    // let back_to_bool: bool = number.into();   // ERROR: Trait has no implementation in context: core::traits::Into::<core::integer::u32, core::bool>.
    // let felt_to_bool: bool = felt_num.into(); // ERROR: Trait has no implementation in context: core::traits::Into::<core::felt252, core::bool>.

    // Manual conversion:
    let back_to_bool: bool = number != 0;
    let felt_to_bool: bool = felt_num != 0;
}
```

Booleans also cannot be used directly as array indices or in arithmetic operations:

```rust
fn main() {
    let my_array = array![10, 20];
    let index: bool = true;

    // These will cause compilation errors:
    // let value = my_array[index];
    // let result = index + 1;

    // Conversions work:
    let numeric_index: u32 = if index {
        1
    } else {
        0
    };
    let value = my_array[numeric_index];
}
```

So the key point is: `bool` to `felt252` works automatically, but `bool` to integer types require manual conversion, and no automatic casting works in the reverse direction.

In most cases, booleans should remain as booleans for better type safety and code clarity.

## Conclusion

Type casting in Cairo emphasizes safety and explicitness over automatic conversion. It uses `Into` for guaranteed conversions and `TryInto` for conversions that might fail, forcing one to handle potential errors explicitly.

This approach prevents the silent data loss common in other languages and catches bugs before they reach production. Cairo's explicit conversions mean your smart contracts behave predictably, even when handling unexpected data.

Unlike Solidity, which requires manual safety checks for type conversion, Cairo builds error handling directly into its casting system through `Option` types.

While Cairo's conversion requirements need more code than implicit casting, this explicitness builds more reliable smart contracts.

*This article is part of a tutorial series on [Cairo Programming on Starknet](https://rareskills.io/cairo-tutorial)*