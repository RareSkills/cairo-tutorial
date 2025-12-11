# Cairo Storage Variable Types

In Starknet, **Contract Storage** is the persistent memory where your smart contract's state lives. Unlike variables declared inside a function which disappear after execution, data in storage remains on the blockchain permanently.

However, simply declaring a variable isn't enough. To interact with contract storage effectively in Cairo, the compiler needs two pieces of logic:

1. **Data Representation:** How to serialize and deserialize the data type for storage. This is handled by the `starknet::Store` trait.
2. **Access Logic:** How to actually read from or write to the specific storage slot. This is handled by a set of **Access Traits**.

For types such as integers, `bool`, `felt252`, `ByteArray`, and so on, Cairo already provides an implementation of the `starknet::Store` trait. As a result, these types can be used directly in the contract’s storage without any extra work. For example, in the contract below, both `felt252` and `u256` are valid storage members because they already implement the trait.

```rust
#[storage]
struct Storage {
    num1: felt252,
    num2: u256,
}
```

However, when dealing with complex types in storage such as mappings, arrays, or user-defined structs, we must either derive the trait or use a special type provided by Cairo to represent our type in storage. These cases will be discussed in detail in later sections.

This article will cover the different types that can be used in storage and the traits each type requires to be used in storage.

## Storage Access Traits

Access traits determine how values are read from or written to storage based on the type being accessed. Cairo uses different access traits internally to resolve these operations depending on whether we are interacting with the types that already implements the `starknet::Store` trait or the special types.

Here's a quick breakdown of these access traits, we will get into details later:

- `StoragePointerReadAccess` and `StoragePointerWriteAccess`: Used for reading and writing values to simple types or custom structs that implement `starknet::Store`.
- `StorageMapReadAccess` and `StorageMapWriteAccess`: Handle reading from and writing to mapping (key-value) types in storage.
- `StoragePathEntry`: Helps resolve access to nested mapping.
- `VecTrait` and `MutableVecTrait`: Provides access to dynamics array in a storage.

Now that we know that any type used in Cairo’s storage must implement the `starknet::Store` trait and that reading from or writing to it requires importing the appropriate access trait, let’s see which types already implement the `starknet::Store` trait before we move on to how reads and writes work at the contract level.

## Types that Implement `starknet::Store` Trait

The following Cairo types implement the `starknet::Store` trait:

- felt252
- unsigned and signed integers
- bool
- bytes31
- ByteArray
- ContractAddress
- Tuple

Since the above types already implement the `starknet::Store` trait, reading from and writing to storage only requires importing the necessary access traits to make the `read()` and `write()` methods available.

Consider a contract that declares several state variables using types listed above. The snippet below shows how each type can be declared in storage:

```rust
#[starknet::contract]
mod HelloStarknet {
    use starknet::ContractAddress;

    #[storage]
    struct Storage {
        // felt252: Field element
        user_id: felt252,

        // u256: 256-bit unsigned integer
        total_supply: u256,

        // bool: Boolean value for true/false conditions
        is_paused: bool,

        // bytes31: Fixed-size byte array (31 bytes), for storing short strings/data
        contract_name: bytes31,

        // ByteArray: for storing long strings
        contract_description: ByteArray,

        // ContractAddress: Starknet contract address type
        owner_address: ContractAddress,

        // Tuple: Groups multiple values together
        version_info: (u8, i8) // (unsigned integer, signed integer)
    }
}

```

Now that we have seen how different data types can be declared in Cairo’s storage, the next step is understanding how to work with them, that is, how to actually write values into storage and later read them back.

### Write Operation

Before we can write to any of the state variable declared above, we have to first import the `StoragePointerWriteAccess` trait. This trait enables the `.write(value)` method on storage variables, allowing us to assign values directly through their storage pointers.

It is available in the `starknet::storage` module:

```rust
// Import `StoragePointerWriteAccess` trait
use starknet::storage::StoragePointerWriteAccess;

```

Here's how to perform write operations on different simple types (*the newly added code are annotated with `/* NEWLY ADDED */` comment*):

```rust
#[starknet::contract]
mod HelloStarknet {
    use starknet::ContractAddress;

    // Import `StoragePointerWriteAccess` trait
    use starknet::storage::StoragePointerWriteAccess; /* NEWLY ADDED */

    #[storage]
    struct Storage {
        user_id: felt252,
        total_supply: u256,
        is_paused: bool,
        contract_name: bytes31,
        contract_description: ByteArray,
        owner_address: ContractAddress,
        version_info: (u8, i8)
    }

    /* NEWLY ADDED */
    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn write_vars(ref self: ContractState) {
             // Writing to felt252
            self.user_id.write(12345);

            // Writing to u256
            self.total_supply.write(1000000_u256);

            // Writing to bool
            self.is_paused.write(false);

            // Writing to bytes31 (short string)
            self.contract_name.write('HelloContract'.try_into().unwrap());

            // Writing to ByteArray (long string)
            self.contract_description.write("This is a very very very long textttt");

            // Writing to ContractAddress
            self.owner_address.write(0x1234.try_into().unwrap());

            // Writing to tuple
            self.version_info.write((1_u8, -2_i8));
        }
    }
}

```

### Read Operations

For read operations, we need to import the `StoragePointerReadAccess` trait, which let us use the `.read()` method on the declared types above:

```rust
use starknet::storage::StoragePointerReadAccess;

```

Extending the contract from the previous section, the code below imports the `StoragePointerReadAccess` trait and read the value from the state variables (*the newly added code are annotated with `/* NEWLY ADDED */` comment*):

```rust
#[starknet::contract]
mod HelloStarknet {
    use starknet::ContractAddress;

    // Import `StoragePointerWriteAccess` trait
        use starknet::storage::{
            StoragePointerWriteAccess,

            /* NEWLY ADDED */
            StoragePointerReadAccess,
        };

    #[storage]
    struct Storage {
        user_id: felt252,
        total_supply: u256,
        is_paused: bool,
        contract_name: bytes31,
        contract_description: ByteArray,
        owner_address: ContractAddress,
        version_info: (u8, i8)
    }

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn write_vars(ref self: ContractState) {
            self.user_id.write(12345);
            self.total_supply.write(1000000_u256);
            self.is_paused.write(false);
            self.contract_name.write('HelloContract'.try_into().unwrap());
            self.contract_description.write("This is a very very very long textttt");
            self.owner_address.write(0x1234.try_into().unwrap());
            self.version_info.write((1_u8, -2_i8));
        }

        /* NEWLY ADDED */
        fn read_vars(self: @ContractState) {
            // felt252: Reading user ID returns a field element (0 to P-1 range)
            let _ = self.user_id.read();

            // u256: Reading large integer, useful for token balances and big numbers
            let _ = self.total_supply.read();

            // bool: Reading boolean state, returns true or false
            let _ = self.is_paused.read();

            // bytes31: Reading fixed-size byte array, used for short strings
            let _ = self.contract_name.read();

            // ByteArray: Reading dynamic-size byte array, used for long strings
            let _ = self.contract_description.read();

            // ContractAddress: Reading Starknet address, type-safe contract/user address
            let _ = self.owner_address.read();

            // Tuple: Reading compound type returns both values as (u8, i8) pair
            let _ = self.version_info.read();
        }
    }
}

```

## Mapping and Vec

Collection types like dictionaries and arrays cannot be stored directly in Cairo contract storage. This is because they use dynamic memory layouts that the storage system doesn’t support by default. Instead, Cairo provides specialized types for working with collections in storage: `Map` and `Vec`.

These special types are available in the `starknet::storage` module, and are used to declare mappings and vectors in contract storage. Before we can use either of them in our `Storage` struct, we will need to explicitly import these special types from the core library, as shown below:

```rust
use starknet::storage::{ Map, Vec };

```

> Note that `Map` and `Vec` don’t need to be imported together, you can import only the one you need, depending on your use case. For instance, if your contract only requires mappings, it's fine to import just the Map type.
> 

Once we have imported `Map` and `Vec`, we can then use them inside the storage struct, like so:

```rust
use starknet::storage::{ Map, Vec };

#[storage]
struct Storage {
    // mapping(address => uint256) my_map;
    my_map: Map<ContractAddress, u256>,

    // uint64[] my_vec;
    my_vec: Vec<u64>,
}

```

The equivalent of how we declare both Map and Vec types in Solidity is commented above each declaration in the above code. The `Map` type takes two generic parameters: the `KeyType` and `ValueType`. In our example, `ContractAddress` is the key and `u256` is the value, meaning this map stores a `u256` amount for each address. The `Vec` type, on the other hand, takes a single type and represents an array of elements of that type. In our example, it is an array of 64-bit unsigned integers.

> Note that there isn't a traditional "fixed array" storage type in Cairo like we have in Solidity and other languages.
> 

With these state variables set up, let’s look at how to interact with them using read and write operations.

### Read and Write Operations on the Map Type

In our example, `my_map` represents a mapping from an address to a `u256` value, similar to how we would define `mapping(address => uint256)` in Solidity.

```rust
my_map: Map<ContractAddress, u256>
```

Before performing these operations, we first need to import the necessary access traits that enable reading from and writing to storage. They are also available in the `starknet::storage` module.

```rust
#[starknet::contract]
mod HelloStarknet {

    // IMPORT MAP TYPE AND NECESSARY ACCESS TRAITS
    use starknet::storage::{
        Map,
        StorageMapWriteAccess, // Enables .write(key, value) operations
        StorageMapReadAccess,  // Enables .read(key) operations
    };

}

```

These traits enable the methods like `.write(key, value)` and `.read(key)`, which we will use in the upcoming examples. Without importing them, we wont be able to perform any of these operations on our storage collection.

With that in place, we can now implement the write and read operations for the `Map` type.

**Write Operation**

We use the `.write(key, value)` method provided by the `StorageMapWriteAccess`. The syntax is straightforward:

```rust
self.my_map.write(key, value);

```

Here’s what each part does:

- `self.my_map` refers to the `Map` declared in the storage struct.
- `.write(...)` is the method that updates the map.
- `key` is the identifier (in our case, a `ContractAddress`) under which the value will be stored.
- `value` is the actual data (in our case, a `u256`) that gets saved.

Each call to `write()` will either insert a new key-value pair or overwrite the existing value if the key already exists.

The below is an example of how to use `.write()` in a function:

```rust
#[starknet::contract]
mod HelloStarknet {
    //...

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {

        fn write_to_mapping(
            ref self: ContractState,
            user: ContractAddress,
            amount: u256
        ) {
            self.my_map.write(user, amount); // write operation
        }
    }
}

```

This stores the `amount` for the given `user` address. Under the hood, Cairo handles writing this to the appropriate storage slot based on the key.

**Read Operation**

We use the `.read(key)` method provided by the `StorageMapReadAccess`. The syntax looks like this:

```rust
self.my_map.read(key);

```

Here's a breakdown of what it does:

- `self.my_map` refers to the map instance in the storage struct.
- `.read(...)` accesses the stored value.
- `key` is the identifier we want to look up, in out case, a `ContractAddress` since we used that as the map’s key.

The `read()` method returns the value associated with the key. If the key hasn’t been written to before, it returns the default value for the map’s valueType (e.g., `0` for `u256`).

Example:

```rust
#[starknet::contract]
mod HelloStarknet {
    // ...

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn get_value(self: @ContractState, user: ContractAddress) -> u256 {
            self.my_map.read(user) // read operation
        }
    }
}

```

This function reads the value stored for a given `user` address and returns it.

### Nested Maps Operations

Reading or writing to a nested mapping in storage requires an additional access trait called `StoragePathEntry`. This trait enables the `.entry(key)` method, which provides access to inner maps stored under a given key.

In other words, when we are dealing with a nested mapping where the value itself is a `Map`, we can’t access it directly with `.read()` or `.write()`. Instead, we must first call `.entry(key)` to reach the inner layer, then perform operations on it.

**Declare a Nested Mapping**

Let’s declare our nested mapping in storage. This will be a two-level map where the outer map uses a `ContractAddress` (a user’s address) as the key, the inner map also uses a `ContractAddress` (a token address) as the key and the value stored is a `u256` representing a token balance:

```rust
#[storage]
struct Storage {
    // user_address => token_address => balance
    two_level_mapping: Map<ContractAddress, Map<ContractAddress, u256>>,
}

```

**Importing Required Trait**

```rust
use starknet::storage::StoragePathEntry;

```

`StoragePathEntry` enables the `.entry(key)` method to get the storage path to the next key in the sequence.

While `.entry()` gives us access to the nested layer, it isn’t enough on its own to perform read or write operations. We still need to import the traits that enable those specific methods. The exact traits we will need depend on how we are performing these operations.

We will look at two ways to read and write to nested mapping in storage.

**Write and Read Operation**

**Method 1**: Always chains `.entry()` for N layers (all the way to the value)

This approach chains multiple `.entry()` calls through each map layer, until it reaches the storage slot. Then, it uses `.write(value)` or `.read()` to interact with the stored value directly.

```rust
#[starknet::contract]
mod HelloStarknet {
    use starknet::ContractAddress;

    // IMPORT TRAITS
    use starknet::storage::{
        Map,
        StoragePathEntry,           // Enables .entry(key)

        /* ADDITIONAL TRAITS */
        StoragePointerWriteAccess,  // Enables .write(value)
        StoragePointerReadAccess,   // Enables .read()
    };

    #[storage]
    struct Storage {
        // user_address => token_address => balance
        two_level_mapping: Map<ContractAddress, Map<ContractAddress, u256>>,
    }

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {

        fn write_nested_map(
            ref self: ContractState,
            key1: ContractAddress,
            key2: ContractAddress,
            value: u256,
        ) {
                // WRITE OPERATION
            self.two_level_mapping.entry(key1).entry(key2).write(value);
        }

        fn read_nested_map(
            ref self: ContractState,
            key1: ContractAddress,
            key2: ContractAddress,
        ) -> u256 {
                // READ OPERATION
            self.two_level_mapping.entry(key1).entry(key2).read()
        }

    }
}

```

For the read and write operations, both functions use the `.entry()` method twice:

- The first `.entry(key1)` accesses the outer map
- The second `.entry(key2)` drills into the inner map to reach a specific storage slot

Once we are at that exact storage location, we use:

- `.write(value)` enabled by the `StoragePointerWriteAccess` trait to write a value directly into that slot.
- `.read()` enabled by the `StoragePointerReadAccess` trait to read the value from that slot.

**Method 2**: Chains `.entry()` for N-1 layers (stopping at the innermost map)

This approach drills down through each map layer using the `.entry()` method till it gets to the inner mapping, then treats it as a whole and interacts with it directly using `.write(key, value)` and `.read(key)`.

```rust
#[starknet::contract]
mod HelloStarknet {
    use starknet::ContractAddress;

    // IMPORT TRAITS
    use starknet::storage::{
        Map,
        StoragePathEntry, // Enables .entry(key) method

        /* ADDITIONAL TRAITS */
        StorageMapWriteAccess, // Enables .write(key, value) method
        StorageMapReadAccess, // Enables .read(key) method
    };

    #[storage]
    struct Storage {
        // user_address => token_address => balance
        two_level_mapping: Map<ContractAddress, Map<ContractAddress, u256>>,
    }

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {

        fn write_nested_map(
            ref self: ContractState,
            key1: ContractAddress,
            key2: ContractAddress,
            value: u256,
        ) {
                // WRITE OPERATION
            self.two_level_mapping.entry(key1).write(key2, value);
        }

        fn read_nested_map(
            ref self: ContractState,
            key1: ContractAddress,
            key2: ContractAddress,
        ) -> u256 {
            // READ OPERATION
            self.two_level_mapping.entry(key1).read(key2)
        }

    }
}

```

For the read and write operations, each function in the code above uses the `.entry(key1)` method once:

- This `.entry(key1)` gives access to the inner map.

Once we have a reference to that inner map, we treat it like a regular `Map`:

- `.write(key2, value)` enabled by the `StorageMapWriteAccess` trait stores the value under the `key2`.
- `.read(key2)` enabled by the `StorageMapReadAccess` trait retrieves the value stored under `key2`.

This method treats the inner value as an entire map, not a single storage slot.

Both methods are valid, the developer simply needs to import the appropriate traits based on how they plan to interact with the nested mapping.

Next, we'll explore how to perform similar operations on the `Vec` type, including pushing new elements and reading from specific indices.

### Read and Write Operations on the Vec Type

The `Vec` type in Cairo is used to represent a growable array in contract storage, similar to dynamic arrays in Solidity like `uint64[]`. It supports common operations such as appending new elements, accessing items by index, and removing elements from the end of the array.

To continue with our example, we will be interacting with the `Vec` type in our storage declaration:

```rust
#[storage]
struct Storage {
    // Solidity equivalent: uint64[] my_vec;
    my_vec: Vec<u64>,
}

```

But before that, we have two traits associated with Vec type: `VecTrait` and `MutableVecTrait`.

**`VecTrait`** provides read-only methods for interacting with vectors in storage. This includes:

- `.len()` – returns the current number of elements in the vector. It return type is `u64`.
- `.get(index)` – safely returns a pointer to the element at the given index. Returns `None` if the index is out of bounds.
- `.at(index)` – returns a pointer to the element at the given index, but **panics** if the index is invalid.

**`MutableVecTrait`** extends `VecTrait` by adding mutating methods that allow you to modify the contents of the vector in storage. These include:

- `.push(value)` – appends a new element to the end of the vector.
- `.pop()` – removes and returns the last element, or returns `None` if the vector is empty.
- `.allocate()` – reserves space for a new element at the end of the vector and returns a writable pointer, useful for complex or nested types.

Depending on the vector operation, we may also need to import access traits like `StoragePointerWriteAccess` or `StoragePointerReadAccess`.

In the next subsections, we will go through examples of common operations like appending, reading, updating, and removing elements using these traits.

**Pushing a New Value to the `my_vec` Vector**

The `push` method used in the `push_number` function increments the vector’s length first, then writes the value to a new storage slot at the end of the vector.

```rust
// IMPORT TRAITS
use starknet::storage::{Vec, MutableVecTrait};

fn push_number(ref self: ContractState, value: u64) {
    // PUSH OPERATION
    self.my_vec.push(value);
}

```

**Reading from an Existing Index**

If we want to retrieve a value at an existing index, we can use `.get()` or `.at()` to get the pointer, then use `.read()` to read its value:

```rust
// IMPORT TRAITS
use starknet::storage::{Vec, MutableVecTrait, StoragePointerReadAccess};

fn read_my_vec(self: @ContractState, index: u64) -> u64 {
    // VEC READ OPERATION
    self.my_vec.at(index).read() // Will panic if index is out of bounds
}

```

Exercise: Why did we add `StoragePointerReadAccess` trait?

**Updating a Value at an Existing Index**

To update a value at an existing index, we can use `.get()` or `.at()` to get the pointer, then use `.write(value)` to modify its value:

```rust
// IMPORT TRAITS
use starknet::storage::{Vec, MutableVecTrait, StoragePointerWriteAccess};

fn write_my_vec(ref self: ContractState, index: u64, val: u64) -> u64 {
    // VEC WRITE OPERATION
    self.my_vec.at(index).write(val) // Will panic if index is out of bounds
}

```

**Getting the Vector’s Length**

`.len()` returns the current number of elements in the vector as a `u64`.

```rust
// IMPORT TRAITS
use starknet::storage::{Vec, MutableVecTrait};

fn get_vec_len(self: @ContractState) -> u64 {
    // RETURN VEC LENGTH
    self.my_vec.len()
}

```

**Popping the Last Element**

```rust
use starknet::storage::{Vec, MutableVecTrait};

fn pop_last(ref self: ContractState) {
    // POP OPERATION
    let _ = self.my_vec.pop();
}
```

`.pop()` retrieves the value stored at the last position in the vector,  decrements the vector's length then returns the retrieved value or `None` if the vector is empty.

## Struct and Enum Type in Storage

Unlike the types that implement the `starknet::Store` trait by default (`u8`, `bool`, `felt252`, etc.), structs require you to explicitly derive the trait, otherwise, any attempt to use that struct in storage will fail at compile time.

To read or write a struct to storage, it must implement the necessary read and write functions which is what the `starknet::Store` trait does automatically.

To make it possible to store a struct in the Cairo storage, we will need to derive the trait by adding this attribute `#[derive(starknet::Store)]` above its definition:

```rust
#[derive(starknet::Store)]
struct User {
    id: u32,
    name: bytes31,
    is_admin: bool,
}

```

Once this is done, the struct can be used in storage-related operations, including inside `#[storage]` struct as type in mappings and arrays.

Below is an example contract that demonstrates how to declare a custom struct in storage, import the necessary traits, and perform read and write operations on that struct.

```rust
#[starknet::contract]
mod HelloStarknet {
    // IMPORT TRAITS
    use starknet::storage::{
        StoragePointerReadAccess, // Enables .read()
        StoragePointerWriteAccess // Eabales .write(value)
    };

        // CUSTOM STRUCT DEFINITION
    #[derive(starknet::Store)]
    struct UserData {
        id: u32,
        name: bytes31,
        is_admin: bool,
    }

    #[storage]
    struct Storage {
        // CUSTOM STRUCT DECLARATION
        user: UserData,
    }

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {

        // WRITE OPERATION
        fn write_struct(ref self: ContractState, _id: u32, _name: bytes31, _is_admin: bool) {

            self.user.id.write(_id); // Write to field 1

            self.user.name.write(_name); // Write to field 2

            self.user.is_admin.write(_is_admin); // Write to field 3
        }

                // READ OPERATION
        fn read_struct(ref self: ContractState) -> (u32, bytes31, bool) {
            let id = self.user.id.read();  // Read from field 1

            let name = self.user.name.read();  // Read from field 2

            let is_admin = self.user.is_admin.read();  // Read from field 3

            (id, name, is_admin)
        }
    }
}

```

**Observe that we imported the following traits:**

```rust
use starknet::storage::{
    StoragePointerReadAccess, // Enables .read() on storage paths
    StoragePointerWriteAccess // Enables .write(value) on storage paths
};

```

We import these traits because the struct fields are simple types, and without them, calls like `.read()` and `.write(value)` would not compile.

**Write Operation**

Inside `write_struct` function:

```rust
// WRITE OPERATION
fn write_struct(
        ref self: ContractState,
        _id: u32,
        _name: bytes31,
        _is_admin: bool
    ) {

    self.user.id.write(_id); // Write to field 1
    self.user.name.write(_name); // Write to field 2
    self.user.is_admin.write(_is_admin); // Write to field 3

}

```

Each call writes a new value to a specific field in the stored struct. This shows that even though the struct is stored as one object, its fields can be accessed and updated independently.

**Read Operation**

This `read_struct` function reads each field individually and returns them as a tuple.:

```rust
fn read_struct(ref self: ContractState) -> (u32, bytes31, bool) {

    let id = self.user.id.read();  // Read from field 1
    let name = self.user.name.read();  // Read from field 2
    let is_admin = self.user.is_admin.read();  // Read from field 3

    (id, name, is_admin)

}

```

### Enum Type

Enums follow a similar pattern to structs, we must explicitly derive `starknet::Store` for them to be stored. Each variant type must also implement the `starknet::Store` trait. In addition, because enums may contain data that requires proper cleanup when values are replaced or dropped, we also need to derive the `Drop` trait.

Here’s a basic example of defining an enum and using it in storage:

```rust
#[starknet::contract]
mod HelloStarknet {

    // DEFINE ENUM
    #[derive(starknet::Store, Drop)]
    enum UserRole {
        Admin,
        Mod,
        #[default]
        User,
    }

    #[storage]
    struct Storage {
        // DECLARE ENUM
        my_role: UserRole,
    }
}

```

In our enum definition, we include the `#[default]` attribute, which is required for any enum that will be used in storage. This attribute marks one of the variants as the default value (in our case, `User` variant) that gets assigned when the storage value has not been set.

**Write and Read Operations on Enum**

The following code imports the access traits required to read and write an enum in storage, followed by two functions that perform these operations:

```rust
// IMPORT TRAITS
use starknet::storage::{
    StoragePointerReadAccess,  // Enables .read() on enum stored in storage
    StoragePointerWriteAccess, // Enables .write(value) on enum stored in storage
};

// WRITE OPERATION
fn write_enum(ref self: ContractState) {
    // Write the Admin variant to storage
    self.my_role.write(UserRole::Admin);
}

// READ OPERATION
fn read_enum(self: @ContractState) {
    // Read the current value of the enum from storage
    let _ = self.my_role.read();
}

```

In Cairo, collection types like `Vec` or `Map` cannot be included as fields in struct or as variants in enum because they rely on dynamic memory layouts that the storage system does not support by default.

```rust
// STRUCT: This will NOT work ❌ - Vec has dynamic size
#[derive(starknet::Store)]
struct InvalidUser {
    name: felt252,
    balance: u256,
    friends: Vec<ContractAddress>,  // ERROR: Cannot store Vec in struct
    tokenBal: Map<ContractAddress, u256>, // ERROR: Cannot store Map in struct
}

// ENUM: This will also NOT work ❌ - Map has dynamic size
#[derive(starknet::Store)]
enum InvalidUserRole {
    Admin: Map<felt252, bool>,  // ERROR: Cannot store Map in enum variant
    #[default]
    User,
}

```

If we need to store collections inside a struct, we have to use a special kind of struct called storage node.

## Storage Nodes

Storage nodes are still structs, but with one key difference: they can contain dynamic collection types like `Vec` and `Map`. Unlike regular user-defined structs we discussed earlier, which do not support collections, storage nodes are specifically designed to handle them, making them ideal for managing nested or dynamic data in storage.

### Defining Storage Nodes

To define a storage node, we use the `#[starknet::storage_node]` attribute instead of the regular struct derives:

```rust
// Storage node - CAN contain collections
#[starknet::storage_node]
struct UserStorageNode {
    name: felt252,
    balance: u256,
    friends: Vec<ContractAddress>,          // ✅ Now allowed!
    tokenBal: Map<ContractAddress, u256>,   // ✅ Also allowed!
}

```

The `#[starknet::storage_node]` attribute allows collection type support and automatically handles the necessary storage logic.

### Using Storage Nodes

Once defined, storage nodes can be declared like any other type inside the `#[storage]` struct. For example, we will declare a storage variable `user_data` with the storage node type we defined above (`UserStorageNode`), like so:

```rust
#[storage]
struct Storage {
    user_data: UserStorageNode,
}

```

Next, we will show how to initialize the `user_data` storage variable and also read from it.

### Storage Node Operations

**Writing to Storage Nodes**

To write to storage nodes, we access their fields directly through the storage variable (in our case, `user_data`), then use the appropriate storage methods like `.write(key, value)`, `.push()`, or `.entry(key).write(value)` depending on the field type as shown below:

```rust
#[abi(embed_v0)]
impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
    fn write_nodes(ref self: ContractState) {
        // Write to simple fields (felt252 and u256)
        self.user_data.name.write(3);
        self.user_data.balance.write(1000_u256);

        // Push a new address to the friends vector
        self.user_data.friends.push(get_caller_address());

        // Write to nested map using either of the two valid approaches
        // Approach 1
        self.user_data.tokenBal.entry(get_caller_address()).write(23);
        // Approach 2
        self.user_data.tokenBal.write(get_caller_address(), 23);
    }
}

```

**Reading from Storage Nodes**

Reading from storage nodes follows similar patterns: we access each field directly and call `.read()` for simple values, `.at(index)` for a specific vector element, or `.read(key)` for a mapping.

```rust
#[abi(embed_v0)]
impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
    fn read_nodes(self: @ContractState) {
        // Read simple values
        let _ = self.user_data.name.read();
        let _ = self.user_data.balance.read();

        // Read a value from the vector at index 0
        let _ = self.user_data.friends.at(0);

        // Read token balance from the nested map
        let _ = self.user_data.tokenBal.read(get_caller_address());
    }
}

```

**Exercise:** By looking at the read and write operations in storage nodes above, list out the traits needed to perform the following operations:

1. `.write(value)`
2. `.push(value)`
3. `.entry(key)`
4. `.write(key, value)`
5. `.read()`
6. `.at(index)`
7. `.read(key)`

## Conclusion

To wrap up, here’s a summary of the most commonly used storage access traits in Cairo. Each trait enables specific methods that allow us to interact with storage types like simple types, `Map`, `Vec`, structs, and enums. Depending on how we want to read from or write to storage, we will need to import the appropriate traits listed below:

| **Trait** | **Enables Method(s)** | **Purpose** |
| --- | --- | --- |
| `StoragePointerReadAccess` | `.read()` | Read a value from a storage path (simple types or struct fields). |
| `StoragePointerWriteAccess` | `.write(value)` | Write a value to a storage path (simple types or struct fields). |
| `StorageMapReadAccess` | `.read(key)` | Read a value from a `Map` by key. |
| `StorageMapWriteAccess` | `.write(key, value)` | Write a value to a `Map` by key. |
| `StoragePathEntry` | `.entry(key)` | Navigate deeper into nested storage (e.g., nested `Map` or storage node). |
| `VecTrait` | `.len()`, `.get(index)`, `.at(index)` | Read-only access to `Vec`: check length, get element optionally or directly. |
| `MutableVecTrait` | `.push(value)`, `.pop()`, `.allocate()` | Mutate a `Vec`: add, remove, or prepare space for an element. |

*This article is part of a tutorial series on [Cairo Programming on Starknet](https://rareskills.io/cairo-tutorial)*