# Events in Starknet

Events emit data from contract execution into the transaction receipt. The receipt holds metadata about what happened during the execution, which can be queried or indexed by external applications. Cairo’s event syntax is more verbose than Solidity’s but serves the same purpose.

In this article, you will understand how Events work in Starknet.

# Events Structure in Cairo

Events in Cairo must be listed in an **Event** enum marked with the `#[event]` attribute. Unlike Solidity's individual event declarations, Cairo requires all events to be organised within a central enum structure.

Here’s an example that lists two events, one for user registration and the other for user login:

```rust
// Event emitted when a new user registers
#[derive(Drop, starknet::Event)]
pub struct UserRegistered {
    pub user_id: u32,
    pub username: ByteArray
}

// Event emitted when a user logs in
#[derive(Drop, starknet::Event)]
pub struct UserLoggedIn {
    pub user_id: u32,
    pub timestamp: u64
}

// Main event enum that holds all possible events this contract can emit
#[event]
#[derive(Drop, starknet::Event)]
pub enum Event {
    NewUser: UserRegistered,   // references UserRegistered struct
    UserLogin: UserLoggedIn    // references UserLoggedIn struct
}
```

*Note: The `Drop` trait allows Cairo to automatically clean up structs and enums from memory when they're no longer needed. You'll see `#[derive(Drop)]` on most Cairo structs and enums.*

The Solidity representation of the two events would be:

```solidity
event NewUser(uint32 userID, string username);
event UserLogin(uint32 userID, uint64 timestamp);
```

In the Cairo code above, we define two event structs (`UserRegistered` and `UserLoggedIn`) that specify the data structure for each event type. Both structs implement the `starknet::Event` trait through the derive attribute.

These separate structs are then unified (listed) under a single `Event` enum, where each variant references its corresponding struct. When events are emitted, the enum variant names (`NewUser`, `UserLogin`) serve as the searchable event identifiers.

Although you'll typically see the same name used for **both the enum variant (like `NewUser`) and its associated struct (like `UserRegistered`), they don't have to match**. A different name is used here to highlight the distinction.

Starknet SDKs, like Starknet.js, can filter and query events using these identifiers. For example, to find all user registrations, you would query for events named `"NewUser"`.

When working with events, you'll often want to filter events by the specific data they contain, such as finding all events for a particular user ID or within a certain value range. This is where indexed parameters come in, just like in Solidity.

# Indexed Events (Key Fields)

Event fields can be marked for indexing using the `#[key]` attribute (similar to `indexed` keyword in Solidity). For example, if we want to make `user_id` searchable in user registrations and `timestamp` searchable for user logins, we would do:

```rust
#[derive(Drop, starknet::Event)]
pub struct UserRegistered {
    #[key]
    pub user_id: u32,          // user_id IS MARKED is AS INDEXED (searchable key)
    pub username: ByteArray    // username IS STORED AS EVENT DATA (not indexed)
}

#[derive(Drop, starknet::Event)]
pub struct UserLoggedIn {
    pub user_id: u32,          // user_id IS STORED AS EVENT DATA (not indexed)
    #[key]
    pub timestamp: u64         // timestamp IS MARKED AS INDEXED
}

// Main event enum that holds all possible events this contract can emit
#[event]
#[derive(Drop, starknet::Event)]
pub enum Event {
    UserRegistered: UserRegistered,
    UserLoggedIn: UserLoggedIn
}
```

Its Solidity equivalent would be:

```solidity
event UserRegistered(uint32 indexed userID, string username);
event UserLoggedIn(uint32 userID, uint64 indexed timestamp);
```

The placement of `#[key]` depends on which specific field you want to make searchable in the event logs. A **field** is a data element within a struct, for example, `user_id` and `username` are fields in the `UserRegistered` struct.

In `UserRegistered`, we're indexing `user_id` for filtering by user, while in `UserLoggedIn`, we're indexing `timestamp` for filtering by time.

You should only add `#[key]` to fields you'll actually query or filter by, each field **must** be annotated individually based on your specific filtering needs.

Key fields are stored separately from regular data fields in the transaction receipts to enable Starknet SDKs to quickly filter events without processing all event data.

## Events Data Structure in Transaction Receipts

A transaction receipt is a record that contains detailed information about a successful transaction. It includes block details (`block_hash`, `block_number`), transaction hash, execution status (`execution_status`, `finality_status`), gas consumption (`execution_resources`), transaction fees (`actual_fee`), etc., and any events that were emitted during execution.

Each transaction receipt contains an `events` array with keys and data for all emitted events where:

- `data`: represent an array containing serialized non-indexed field values
- `from_address`: is the contract address that emitted the event
- `keys`: is an array that **always** contains the event selector hash at `keys[0]`, plus any indexed field values at `keys[1]`, `keys[2]`, etc.

*The `keys` array is present in every event, regardless of whether you use `#[key]` annotations in your contract. At minimum, it contains the event selector hash that identifies the event type (i.e Transfer, e.t.c).*

The following shows an example transaction receipt with the **events** array structure highlighted in the pink box:

![A Cairo transaction with the event fields highlighted](https://r2media.rareskills.io/StarknetEvents/image5.png)

Here is the Typescript code that generated this transaction receipt based on the transaction hash provided using the `getTransactionReceipt` method:

```tsx
import { RpcProvider } from "starknet";
import * as dotenv from "dotenv";

dotenv.config();

async function getTxnReceipt() {
    // Initialize RPC provider with Sepolia testnet endpoint
    const alchemyApiKey = process.env.ALCHEMY_API_KEY;

    // initialize provider for Sepolia testnet with Alchemy
    const provider = new RpcProvider({
      nodeUrl: `https://starknet-sepolia.g.alchemy.com/starknet/version/rpc/v0_8/${alchemyApiKey}`,
    });
    // Transaction hash to query (replace with actual hash)
    const transactionHash =
      "0x5df0e42012440f59eb9cdd7994a3001b72cebc781bd8527fb3a5343cdb9d6f7";

    try {
        // Fetch transaction receipt from the network
        const receipt: any = await provider.getTransactionReceipt(transactionHash);

        // Display formatted receipt data
        console.log(JSON.stringify(receipt, null, 2));
    } catch (error) {
        // Handle network or transaction errors
        console.error("Error getting transaction receipt:", error);
    }
}

// Execute the function
getTxnReceipt();
```

*This example only shows how events appear in transaction receipts. Different querying techniques with `Starknet.js` are covered in the later section of the article.*

### Understanding the Keys Array

In the transaction receipt above, the event has only one key (`keys[0]`) containing the event selector hash `0x99cd8bde557814842a3121e8ddfd433a539b8c9f14bf31ebf108d12e6196e9`.

![Screenshot 2025-09-02 at 11.16.37.png](https://r2media.rareskills.io/StarknetEvents/image12.png)

This event selector hash (`keys[0]`) represents a `Transfer` event with **no indexed** parameters, with all its non-indexed fields stored in the `data` array:

![Screenshot 2025-09-02 at 11.20.43.png](https://r2media.rareskills.io/StarknetEvents/image13.png)

The event selector hash is computed using:

```jsx
const nameHash = num.toHex(hash.starknetKeccak('EventName'));
```

### Cairo Event Structure Comparison with Solidity

In Solidity, event data is structured using topics and data as seen in this [example](https://sepolia.etherscan.io/address/0x3cb5c3961dea77a01d357a8734dfe943e199c355#events):

![A Solidity event with topics and data fields highlighted](https://r2media.rareskills.io/StarknetEvents/image7.png)

- **topic0**: **Always** contains the event signature hash. In the above example, the event `NewUser(uint32,string)` has the hash `keccak256("NewUser(uint32,string)")`, which equals `0x37ecc4388271ab7af2220881c1f2f70fbea71e6b1635107f9daffa0fab84d5b3`.
- **topic1**: Contains the indexed `user_id` parameter.
- **Data field**: Contains all non-indexed parameters (in Hex).

Cairo follows a similar pattern with the keys array. For comparison, view this [transaction](https://sepolia.voyager.online/event/1002359_7_9) on Starknet Sepolia that shows an event with multiple keys:

![A Starknet event with indexed parameters and data highlighted](https://r2media.rareskills.io/StarknetEvents/image8.png)

- **`keys[0]`** (highlighted by the white arrow) holds the event selector hash (`Event8Indexed`).
- Subsequent elements in the array (e.g., `keys[1]`, `keys[2]`, …`keys[10]`), shown with the green arrow, represent the indexed (`#[key]`) fields of the event.
- Non-indexed parameters are stored separately in the `data` field, as illustrated in the purple box.

Based on the image above, the `KEYS` section contains a total of ten (10) indexed parameters. This highlights a key advantage of Cairo over Solidity: while Solidity caps indexed parameters at three (`topic1-topic3`), with `topic0` always reserved for the event signature, Cairo allows up to fifty (50) indexed parameters. This eliminates the need for workarounds like anonymous events in Solidity (which can have up to 4 indexed parameters but lose the event signature).

As in Solidity, non-indexed fields in Cairo require manual searching through the `data` array, while indexed fields (marked with `#[key]`) can be efficiently filtered using Starknet SDKs.

# How Events Work Internally

Starknet's event system in Cairo primarily revolves around two traits:

- `Event`: handles event serialization & deserialization.
- `EventEmitter`: provides the emission capability using `self.emit(...)` inside contract functions.

## `Event` Trait

The Event trait provides methods that serializes events, deserialize events, and generates an internal event type identifier used for event filtering and indexing. Any struct or enum marked with `#[derive(starknet::Event)]` automatically gets implementations for these key methods:

| **Method** | **Purpose** |
| --- | --- |
| `append_keys_and_data` | Serializes the event by splitting indexed (`#[key]`) and non-indexed fields into separate `keys` and `data` arrays |
| `deserialize(ref keys, ref data) -> Option<T>` | Reconstructs the original event from emitted transaction receipt data; returns `None` if invalid |
| `event_type_name` | Internal identifier used to compute event selector for filtering and indexing |

Note that you don’t implement these methods manually. They're generated automatically when you use `#[derive(starknet::Event)]`, which sets up the Events in the contract with everything needed to serialize and reconstruct the event data.

### Understanding Event Serialization

The `Event` trait handles event serialization automatically, but when events contain complex field types such as arrays, nested structs, it relies on the `Serde` trait for additional serialization help.

The `Serde` trait converts complex Cairo types into a sequence of `felt252` values compatible with the Cairo VM. In Cairo, `felt252` is the only primitive type understood by the Cairo VM, so any value larger than 252 bits must be broken down into a list of `felt252` values.

**Types handled automatically by the `Event` trait are:**

- Simple types: `u8`, `u16`, `u32`, `u64`, `u128`, `bool`, `felt252`, `ContractAddress`
- `ByteArray`: Automatically serialized without manual `Serde` derivation

*As covered in earlier chapters, `ByteArray` is a Cairo type that represents strings. It's a struct with three fields:*

- `data: Array<felt252>`: contains 31-byte chunks of the string data*
- `pending_word: felt252`:  remaining bytes after filling the `data` array with complete 31-byte chunks (up to 30 bytes)
- `pending_word_len: u32`: number of bytes in `pending_word`


*Cairo first packs complete 31-byte chunks into the `data` array, then puts any leftover bytes into `pending_word`. For example, "serah" gets serialized as:*

- `data`: `[]` (empty array - no 31-byte chunks needed for a 5-byte string)
- `pending_word`: `0x7365726168` (contains the actual string bytes in hex format)
- `pending_word_len`: `0x5` ( 5 bytes total)

*Since `ByteArray` is commonly used and part of Cairo's standard library, the `Event` trait includes automatic serialization support for it.*

*This automatic serialization is what allows us to use `ByteArray` in events without deriving `Serde` manually.  We'll see the detailed `ByteArray` serialization breakdown when we examine an actual transaction data later in the article.*

**Complex types requiring manual `Serde` derivation are:**

- Custom structs: User-defined structures like nested data
- Arrays: `Array<u32>`, `Array<ByteArray>`, etc.
- User-defined enums with data

When an event contains fields of these complex types, **those field types** must derive `Serde` so the `Event` trait can serialize them during emission. Without this, the compiler cannot process the events correctly. A practical example of this will be shown in the ***“Handling Complex Event Field Types”*** section of the article.

## `EventEmitter` Trait

`EventEmitter` trait enables events to be emitted via:

```rust
self.emit(EventStruct { ... });
```

During contract execution, it serializes the event using the `Event` trait and stores the result in the transaction receipt. For events containing custom struct fields, those structs must derive `Serde` separately so the `Event` trait can serialize them properly.

The following diagram visualizes the event serialization workflow, showing which event types can be serialized automatically and which require additional `Serde` support before emission:

![A diagram showing the dependencies of the event trait](https://r2media.rareskills.io/StarknetEvents/image6.png)

With the basics covered, the following section explores how event structures function when nested or include complex types.

# Handling Complex Event Field Types

Here's the updated `UserRegistered` event with additional field types. `UserMetadata` is a custom struct that holds user environment data (device type and location information). Being a nested struct within `UserRegistered` event, it requires proper serialization:

```rust
#[derive(Drop, starknet::Event)]
pub struct UserRegistered {
    #[key]
    pub user_id: u32,
    pub username: ByteArray,
    pub metadata: UserMetadata,
    pub tag_count: u32,
    pub timestamp: u64,
}

#[derive(Drop, Serde)]
pub struct UserMetadata {
    pub device_type: ByteArray,
    pub ip_region: ByteArray,
}
```

Since `UserMetadata` is a complex type, it must derive `Serde` so that it can be correctly serialized into `felt252` values.

This ties back to our earlier note on Cairo VM constraints: complex types **must be serialized** properly for successful event emission.

Without `Serde` in the `UserMetadata` struct, the code fails to compile, as shown below:

![Screenshot 2025-07-16 at 17.28.06.png](https://r2media.rareskills.io/StarknetEvents/image11.png)

*`UserMetadata` needs `Serde` because it's a custom struct. The `UserRegistered` event struct only needs `starknet::Event` (the Event trait handles basic types automatically but delegates to `Serde` for complex field types.)*

Also, note that complex type indexed fields (`#[key]`) are stored as hashed values in the event's `keys` array and cannot be directly recovered from transaction logs.

**Current `UserRegistered` event (recommended approach):**

This design uses `user_id` (a `u32` primitive type) as the indexed field, which remains readable in transaction logs for efficient querying:

```rust
#[derive(Drop, starknet::Event)]
pub struct UserRegistered {
    #[key]
    pub user_id: u32,           // Primitive type (stays readable)
    pub username: ByteArray,    // Non-indexed (in data array)
    pub metadata: UserMetadata, // Non-indexed (in data array)
    pub tag_count: u32,         // Non-indexed (in data array)
    pub timestamp: u64,         // Non-indexed (in data array)
}
```

**Transaction receipt:**

```json
{
  "keys": [
    "0x...event_selector",    // key[0] is always the event selector.
    "0x7b"                    // user_id = 123 (readable as hex)
  ],
  "data": [
    "username_serialized",
    "metadata_serialized",
    "tag_count_serialized",
    "timestamp_serialized"
  ]
}
```

With this approach, you can easily query for events where `user_id = 123` because the value `0x7b` is directly readable in the keys array (`keys[1]`).

If we used the complex `UserMetadata` struct as an indexed field, it would create querying challenges:

```rust
#[derive(Drop, starknet::Event)]
pub struct UserRegistered {
    #[key]
    pub metadata: UserMetadata,  // Complex struct as indexed field - BAD!
    pub user_id: u32,
    pub username: ByteArray,
}
```

What you'd see in the receipt:

```json
{
  "keys": [
    "0x...event_selector",
    "0xa1b2c3d4e5f67890..."    // Hashed UserMetadata - unreadable!
  ],
  "data": ["0x7b", "username_serialized"]
}

```

The entire `UserMetadata` becomes an unreadable hash in the keys array. We can't query for users by `device_type` or `ip_region` because these values are hidden within the hash. This is why primitive types like `u32` work better for indexed fields when we need to filter events efficiently. So if you plan to query events by indexed fields, it's best to use primitive types like `u32`, `felt252`, or `ContractAddress`.

## Using `#[flat]` attribute

The `#[flat]` attribute alters how event variants are named and identified in transaction logs. It is used to flatten nested event enums to make querying and filtering of specific event easier.

This attribute addresses nested enum structures, not complex field types. This is a separate concept from the complex indexing we just discussed.

> `#[flat]` attribute flattens the event **naming hierarchy**, not the data structure itself.
>

When used on an event variant in the **`Event`** enum, it changes the event selector hash computation to use the inner variant name instead of the outer enum name.

**Outer Enums, Inner Enum and Inner Variants**

To understand how `#[flat]` works, we need to distinguish between these three levels of enum structure:

```rust
// OUTER enum (the main Event enum)
pub enum Event {
    UserRegistered: UserRegistered,
    #[flat]
    UserDataUpdated: UserDataUpdated,  // <- This references the INNER enum
}

// INNER enum (nested inside the outer enum structure)
pub enum UserDataUpdated {
    DeviceType: UpdatedDeviceType,     // <- These are the inner variants
    IpRegion: UpdatedIpRegion,         // <- These are the inner variants
}
```

- **Outer enum**: The main `Event` enum that contains all possible events for the contract
- **Inner enum**: The `UserDataUpdated` enum that contains specific variants (`DeviceType` and `IpRegion`)
- **Inner variant:** The individual enum variants (`DeviceType` and `IpRegion`) within the `UserDataUpdated` enum, each referencing its own event struct

**Complete Example**

The following example shows a contract with multiple event types: a simple struct event (`UserRegistered`) and a nested enum event (`UserDataUpdated`) that contains two variants:

```rust
// Main event enum that holds all possible events this contract can emit
#[event]
#[derive(Drop, starknet::Event)]
pub enum Event {
    UserRegistered: UserRegistered,
    #[flat]                              // NEWLY ADDED (flattens nested event enum)
    UserDataUpdated: UserDataUpdated,
}

// event for user registration
#[derive(Drop, starknet::Event)]
pub struct UserRegistered {
    #[key]
    pub user_id: u32,                      // Indexed user ID for filtering
    pub username: ByteArray,               // Username as event data
    pub metadata: UserMetadata,            // User metadata struct
}

// Nested event enum containing different types of user data updates
#[derive(Drop, starknet::Event)]
pub enum UserDataUpdated {
    DeviceType: UpdatedDeviceType,           // Device type change event
    IpRegion: UpdatedIpRegion,               // IP region change event
}

// Event for device type updates
#[derive(Drop, starknet::Event)]
pub struct UpdatedDeviceType {
    #[key]
    pub user_id: u32,                         // Indexed user ID
    pub new_device_type: ByteArray,           // New device type value
}

// Event for IP region updates
#[derive(Drop, starknet::Event)]
pub struct UpdatedIpRegion {
    #[key]
    pub user_id: u32,                         // Indexed user ID
    pub new_ip_region: ByteArray,             // New IP region value
}

// User metadata structure containing device and location info
#[derive(Drop, Serde)]
pub struct UserMetadata {
    pub device_type: ByteArray,                // User's device type
    pub ip_region: ByteArray,                  // User's IP region
}
```

Notice how the `#[flat]` attribute is applied to the `UserDataUpdated` (nested enum) variant in the main Event enum, this is what changes how the inner variants (`DeviceType` and `IpRegion`) appear in transaction logs.

Recall, that the event selector hash(stored in `keys[0]` of the transaction receipt) is computed using `starknetKeccak("EventName")`.

Without the `#[flat]` attribute, the event selector hash is derived from the outer enum name: `starknetKeccak("UserDataUpdated")`. This means all enum variants (`DeviceType` and `IpRegion`) share the same event selector, so you can’t query for specific variants, you can only query for `"UserDataUpdated"` events in general.

```json
{
  "keys": ["0x...hash_of_UserDataUpdated"],  // Same selector for all variants*
  "data": [...],
  "from_address": "0x..."
}
```

But when we use `#[flat]`, the event selector hash is computed from the inner variant name: `starknetKeccak("DeviceType")` / `starknetKeccak("IpRegion")`, so `DeviceType` and `IpRegion` each get their own selector hash for precise filtering and querying.

```json
// DeviceType event
{
  "keys": ["0x...hash_of_DeviceType"],  // Unique selector
  "data": [...],
  "from_address": "0x..."
}

// IpRegion event
{
  "keys": ["0x...hash_of_IpRegion"],   // Different unique selector
  "data": [...],
  "from_address": "0x..."
}
```

The `#[flat]` attribute only affects event naming and selector computation, the actual data structure, fields, and serialization remain unchanged. This makes event filtering and log inspection much easier when working with nested event enums.

*`#[flat]` attribute is commonly used in OpenZeppelin component libraries to ensure component events match the standard event structure*

*For example, when using ERC20 and Ownable components, `#[flat]` removes the component ID prefix from events, so ERC20's `Transfer` and `Approval` events, along with Ownable's `OwnershipTransferred` event, appear with their own selector hash as the first key, just like they would in standalone contracts. (Components are explained in detail in Chapter 13 - for now, think of them as reusable contract modules.)*

Note that structs used as enum variants in event enums must derive `starknet::Event` because they become event types themselves when used in the enum structure.

## Testing Event Logs

Scaffold a new Scarb project `scarb new testinglog` and select ‘Starknet Foundry (default)’ as your test runner:

![Initializing a Starknet Foundry project](https://r2media.rareskills.io/StarknetEvents/image4.png)

To test for event logs, consider this `UserManager` contract below that allows users to register themselves and tracks how many users have registered.

The contract uses a counter to assign unique IDs to users and stores their information in a Map. When a user registers, the contract emits a `UserRegistered` event that external applications can query. Pay attention to the `#[key]` attribute and how the `UserMetadata` struct is stored.

Copy and paste the complete code into your `src/lib.cairo` file:

```rust
// Interface defining the functions our UserManager contract will implement
#[starknet::interface]
pub trait IUserManager<TContractState> {
    fn register_user(ref self: TContractState, username: ByteArray);
    fn get_user_count(self: @TContractState) -> u32;
}

// Struct to store user information (derives Store to enable storage in contract)
#[derive(Drop, Serde, starknet::Store)]
pub struct UserMetadata {
    pub user_id: u32,
    pub username: ByteArray
}

// Event emitted when a new user registers (user_id is marked as key for indexing)
#[derive(Drop, starknet::Event)]
pub struct UserRegistered {
    #[key]
    pub user_id: u32,
    pub username: ByteArray,
    pub timestamp: u64,
}

#[starknet::contract]
pub mod UserManager {
    use super::{UserRegistered, UserMetadata, IUserManager};
    use starknet::{
        get_block_timestamp, ContractAddress, get_caller_address,
        storage::{Map, StoragePathEntry, StoragePointerReadAccess, StoragePointerWriteAccess}
    };

    #[storage]
    struct Storage {
        user_counter: u32,  // Tracks total number of registered users
        users: Map<ContractAddress, UserMetadata> // Maps user addresses to their metadata
    }

    // Main event enum that holds all possible events this contract can emit
    #[event]
    #[derive(Drop, starknet::Event)]
    pub enum Event {
        UserRegistered: UserRegistered,
    }

    #[abi(embed_v0)]
    impl UserManagerImpl of IUserManager<ContractState> {
        fn register_user(ref self: ContractState, username: ByteArray) {
            // Get current user count and increment for new user ID
            let current_counter = self.user_counter.read();
            let user_id = current_counter + 1;

            // Create user metadata with new ID and provided username
            let metadata = UserMetadata {
                user_id,
                username: username.clone()
            };

            // Update counter and store user data mapped to caller's address
            self.user_counter.write(user_id);
            self.users.entry(get_caller_address()).write(metadata);

            // Emit event with user details and current timestamp
            self.emit(UserRegistered {
                user_id,
                username,
                timestamp: get_block_timestamp(),
            });
        }

        fn get_user_count(self: @ContractState) -> u32 {
            // Return the current number of registered users
            self.user_counter.read()
        }
    }
}
```

`IUserManager` trait defines two functions; `register_user` for registration and `get_user_count` to check the total number of registered users.

- `UserMetadata` struct stores user information (ID and username) and can be saved to contract storage. It derives `starknet::Store` because it's stored in contract storage within the `Map<ContractAddress, UserMetadata>`.

    Any custom struct that needs to be read from or written to contract storage must implement the `Store` trait, which `#[derive(starknet::Store)]` automatically generates.

- The `UserRegistered` event struct logs registration details. The `user_id` field is marked with `#[key]`, making it indexed for efficient filtering in queries.

When `register_user` is called, the contract:

- Increments the user counter to generate a new user ID
- Creates and stores the user's metadata
- Emits a `UserRegistered` event with the user ID, username, and current block timestamp

Navigate to your project directory `cd testinglog` and run `scarb build` to build your project:

![Running scarb build in the terminal](https://r2media.rareskills.io/StarknetEvents/image3.png)

There are various ways to test for events using Starknet Foundry. You can test to assert if an event was emitted using the `assert_emitted` method, or use `assert_not_emitted` to test for lack of event emission. You can also test manually by inspecting events directly.

For manual event testing, you may want to filter events from a specific contract rather than examining all emitted events. The `emitted_by` method on the Events structure allows you to narrow down events to those from a particular address.

Both the `assert_emitted` method and the manual method for testing events will be discussed.

### Test 1: Using the built-in **`assert_emitted`**

The test below verifies that registering a user emits the correct `UserRegistered` event with the expected data. Navigate to the `tests/test_contract.cairo`, copy and paste this test into it:

```rust
use snforge_std::{declare, ContractClassTrait, DeclareResultTrait, spy_events, EventSpyTrait, IsEmitted, Event, EventSpyAssertionsTrait};
use testinglog::{IUserManagerDispatcher, UserManager, UserRegistered, IUserManagerDispatcherTrait};
use starknet::{ContractAddress, get_block_timestamp};

fn deploy_contract(name: ByteArray) -> ContractAddress {
    let contract = declare(name).unwrap().contract_class();
    let (contract_address, _) = contract.deploy(@ArrayTrait::new()).unwrap();
    contract_address
}

#[test]
fn test_registration_event_emission() {
    // Deploy the UserManager contract
    let contract_address = deploy_contract("UserManager");

    // Create a dispatcher to interact with the deployed contract
    let dispatcher = IUserManagerDispatcher { contract_address };

    // Start spying on events before the function call
    let mut spy = spy_events();

    // Register a user - this should emit a UserRegistered event
    dispatcher.register_user("serah");

    // Verify that the expected event was emitted with correct data
    spy.assert_emitted(
        @array![
            (
                contract_address,   // Event should come from our contract
                UserManager::Event::UserRegistered(
                    UserRegistered {
                        user_id: 1,   // First user gets ID 1
                        username: "serah", // Username matches what we passed
                        timestamp: get_block_timestamp() // Timestamp should be current block time
                    }
                )
            )
        ]
    );
}
```

**Imports** bring in required testing tools from Starknet Foundry and our contract definitions.

From `snforge_std`, we import `declare` to load contract classes, along with related traits like `ContractClassTrait` and `DeclareResultTrait` for contract deployment.

```rust
use snforge_std::{declare, ContractClassTrait, DeclareResultTrait, spy_events, EventSpyTrait,IsEmitted, Event, EventSpyAssertionsTrait};
```

Event testing functionality comes from `spy_events` which creates our event spy, `EventSpyTrait` for spy interactions, and `EventSpyAssertionsTrait` which adds assertion methods like `assert_emitted`. We also import `IsEmitted` and `Event` types for event handling operations.

From our testinglog module, we import the auto-generated `IUserManagerDispatcher` and its trait for calling contract functions, the `UserManager` contract module containing our event definitions, and the specific `UserRegistered` event struct we're testing.

```rust
use testinglog::{IUserManagerDispatcher, UserManager, UserRegistered, IUserManagerDispatcherTrait};
```

From `starknet` core, we import `ContractAddress` for handling contract addresses and `get_block_timestamp` to retrieve the current block timestamp.

```rust
use starknet::{ContractAddress, get_block_timestamp};
```

`test_registration_event_emission()` uses the simplified approach with `spy.assert_emitted()`

- `deploy_contract("UserManager")` is a helper function that declares and deploys the `UserManager` contract, returning its address
- `IUserManagerDispatcher { contract_address }` creates a dispatcher to interact with the deployed contract
- `spy_events()` initializes event spying before we trigger the action

After calling `register_user("serah")` through the dispatcher, `spy.assert_emitted()` checks to verify that the expected `UserRegistered` event was emitted with the correct data (*user_id: 1, username: "serah", and current timestamp*). The assertion checks both the contract address that emitted the event and the event data structure.

Run `scarb test` you should see the test pass, confirming that our event testing works correctly.

### Test 2: Using manual method

`test_event_structure()` tests to ensure that the `register_user` function works correctly and emits the expected `UserRegistered` event.

```rust
#[test]
fn test_event_structure() {
    // Deploy the UserManager contract
    let contract_address = deploy_contract("UserManager");

    // Create a dispatcher to interact with the deployed contract
    let dispatcher = IUserManagerDispatcher { contract_address };

    // Start event spy to capture all emitted events
    let mut spy = spy_events();

    // Register a user which should emit a UserRegistered event
    dispatcher.register_user("serah");

    // Retrieve all captured events for analysis
    let events = spy.get_events();
    assert(events.events.len() == 1, 'There should be one event');

    // Create the expected event structure for comparison
    let expected_event = UserManager::Event::UserRegistered(
        UserRegistered {
            user_id: 1,
            username: "serah",
            timestamp: get_block_timestamp()
        }
    );

    // Check if the expected event was actually emitted
    assert!(events.is_emitted(contract_address, @expected_event));

    // Create array of expected events for exact comparison
    let expected_events: Array<(ContractAddress, Event)> = array![
        (contract_address, expected_event.into()),
    ];
    assert!(events.events == expected_events);

    // Extract and examine the raw event data
    let (from, event) = events.events.at(0);
    assert(from == @contract_address, 'Emitted from wrong address');

    // Verify event keys structure (event selector + indexed fields)
    assert(event.keys.len() == 2, 'There should be two keys');
    assert(event.keys.at(0) == @selector!("UserRegistered"), 'Wrong event name');
}
```

When we call `register_user()`, it retrieves all captured events using `spy.get_events()` and performs checks:

- confirming the expected event was emitted using `events.is_emitted()`, and
- examining the raw event structure including the contract address, key count (keys should contain exactly 2 elements: event selector + indexed `user_id`), and event selector.

`register_user()` emits a `UserRegistered` event containing the user data.

This manual method allows testing of specific event properties that automated assertions might not cover.

Paste this second test into the same file `tests/test_contract.cairo`, so that the file contains both the first and second tests. Then proceed to test the project using `scarb test`.

Your terminal output should show the tests passes

![Tests passing in Scarb](https://r2media.rareskills.io/StarknetEvents/image2.png)

## Viewing Raw Event Data

To view the raw event data, you can inspect [this deployed UserManager contract on Voyager](https://www.notion.so/your-link-here). Click on the "Events" tab to see the emitted event logs from calling `register_user`, as shown below:

![Viewing raw event data in the block explorer](https://r2media.rareskills.io/StarknetEvents/image10.png)

As mentioned earlier, a serialized `ByteArray` is a struct consisting of `[data, pending_word, pending_word_len]`, each stored as a `felt252`. That is why "serah" occupied data[0-2] as seen in the image above.

- `data` (data[0]): Empty array `[0x0]` because "serah" (5 bytes) doesn't need any 31-byte chunks
- `pending_word`(data[1]): `0x7365726168` contains the actual string bytes
- `pending_word_len`(data[2]): `0x5` (5 bytes total)
- data[3]:`0x68c6c625` indicates the timestamp in hex.

This raw view shows exactly how Cairo serializes data; everything gets converted into a sequence of `felt252` values (displayed here as hexadecimal) while making it possible to reconstruct the original data structure.

The following sections show three basic approaches to event data retrieval and processing in Starknet.

# Querying and Monitoring Events On-chain and Off-chain

Understanding event structure is just one part. In practice, you may want immediate transaction feedback, real-time monitoring, or historical analysis. Each requires a different approach.

## Parsing Event Logs

Consider this minimal TypeScript example that illustrates how to parse events from Starknet smart contract transactions when you need immediate feedback from your own transactions.

- Clone this [repository](https://github.com/Sayrarh/starknet-event-parsing.git), and cd into the starknet-event-parsing directory:

```bash
git clone https://github.com/Sayrarh/starknet-event-parsing.git
cd starknet-event-parsing
```

- If you don't have yarn installed, install it first using `npm install -g yarn`
- Run `yarn install` to install dependencies, then install dotenv using `yarn add dotenv`
- Create a `.env` file in the root directory:

```rust
ACCOUNT_ADDRESS=0x...
PK=0x...
ALCHEMY_API_KEY=your_alchemy_api_key_here
```

- Replace with your actual account address, private key, and API key from [Alchemy](https://dashboard.alchemy.com/).
- Edit the main script (`src/event.ts`) to specify the address of the ERC-20 contract you want to parse **`Transfer`** events for, or any other contract address (STRK token on Sepolia is used here), and also the recipient address:

```jsx
const contractAddress = "0x04718f5a0fc34cc1af16a1cdee98ffb20c31f5cd61d6ab07201858f4287c938d";
const recipientAddress = "0x0207d7324a20d6A080C7EF6237D289fD57F4fb11187A64f597d4099a720FE6C5";
```

Ensure your account is active on-chain and has STRK tokens for transaction fees.

- Run the script using `yarn dev` . The script will:
    - Connect to your Starknet account on Sepolia
    - Execute a transfer of 1 STRK token to the specified recipient address
    - Wait for transaction confirmation
    - Extracts and displays all events emitted during that transaction

![A screen shot showing results after running yarn dev](https://r2media.rareskills.io/StarknetEvents/image1.png)

This helps in parsing `Transfer` events from any ERC20 token contract on Starknet.

You can also customize the script for different contracts and scenarios:

```tsx
await eventLogic(
  "0x... your contract address",
  "your_function_name",
  [arg1, arg2,...]
);
```

## Listening to Events

This comes in handy when you need real-time monitoring of contract activity. Replace `src/event.ts` with the following code example that triggers a callback every time an ERC-20 token emits a **`Transfer`** event:

```tsx
// Import necessary Starknet.js components for RPC interaction
import { RpcProvider} from "starknet";
import * as dotenv from "dotenv";

dotenv.config();

async function listenToTransfers() {
  const alchemyApiKey = process.env.ALCHEMY_API_KEY;

  // initialize provider for Sepolia testnet with Alchemy
  const provider = new RpcProvider({
    nodeUrl: `https://starknet-sepolia.g.alchemy.com/starknet/version/rpc/v0_8/${alchemyApiKey}`,
  });

  // Contract address to monitor for events
  const contractAddress = "0x04718f5a0fc34cc1af16a1cdee98ffb20c31f5cd61d6ab07201858f4287c938d";

  // Track the last processed block to avoid re-processing events
  let lastBlock = 0;

  async function checkForEvents() {
    // Get the current block number from the network
    const currentBlock = await provider.getBlockNumber();

    // Only check for new events if there are new blocks
    if (currentBlock > lastBlock) {
      // Query for Transfer events between the last processed block and current block
      const events = await provider.getEvents({
        address: contractAddress,     // Only events from our target contract
        keys: [["0x99cd8bde557814842a3121e8ddfd433a539b8c9f14bf31ebf108d12e6196e9"]],  // Transfer event selector (keccak hash)
        from_block: { block_number: lastBlock + 1 },  // Start from next unprocessed block
        to_block: { block_number: currentBlock },    // Query up to current block
        chunk_size: 100     // Process events in batches of 100
      });

      // Process each detected Transfer event
      events.events.forEach(event => {
        console.log("Transfer event detected!", event);
      });

      // Update last processed block to current block
      lastBlock = currentBlock;
    }
  }

  // Set up polling: check for new events every 10 seconds
  setInterval(checkForEvents, 10000);

  // Run initial check immediately
  checkForEvents();
}

// Start the event listener
listenToTransfers();
```

When you run `yarn dev`, you will see new transactions at intervals in your terminal output until you press `Ctrl+C`.

![Transfer events detected by the script](https://r2media.rareskills.io/StarknetEvents/image9.png)

## Filtering Events by Range

When you need historical data analysis and queries, you can use `provider.getEvents()` from `Starknet.js` to query historical events within a specific block range.

Replace `src/event.ts` again with this following code example that searches blocks 8000 to 9000 (1000 blocks total) for `Transfer` events from the specified contract:

```tsx
import { RpcProvider } from "starknet";
import * as dotenv from "dotenv";

dotenv.config();

async function filterTransferEvents() {
    const alchemyApiKey = process.env.ALCHEMY_API_KEY;

    // initialize provider for Sepolia testnet with Alchemy
    const provider = new RpcProvider({
      nodeUrl: `https://starknet-sepolia.g.alchemy.com/starknet/version/rpc/v0_8/${alchemyApiKey}`,
    });


  // Target contract address to query for Transfer events
  const contractAddress = "0x04718f5a0fc34cc1af16a1cdee98ffb20c31f5cd61d6ab07201858f4287c938d";
  const transferSelector = "0x99cd8bde557814842a3121e8ddfd433a539b8c9f14bf31ebf108d12e6196e9";

  // Query for Transfer events within a specific block range
  const events = await provider.getEvents({
    address: contractAddress,                   // Only events from our target contract
    keys: [[transferSelector]],                 // Filter for Transfer events only
    from_block: { block_number: 8000 },         // Start searching from block 8000
    to_block: { block_number: 9000 },           // Search up to block 9000 (1000 block range)
    chunk_size: 100                             // Process events in batches of 100
  });

  // Display total number of Transfer events found
  console.log(`Found ${events.events.length} Transfer events`);

  // Process and display details for each Transfer event
  events.events.forEach((event, index) => {
    console.log(`\n--- Transfer Event ${index + 1} ---`);
    console.log("From:", event.keys[1]);
    console.log("To:", event.keys[2]);
    console.log("Amount (hex):", event.data[0]);
    console.log("Amount (decimal):", parseInt(event.data[0], 16));
    console.log("Block:", event.block_number);
    console.log("Transaction:", event.transaction_hash);
  });
}

// Execute the event filtering function
filterTransferEvents();
```

Run `yarn dev` and you will get the filtered events within the specified block range. The filtering uses these parameters:

- `contractAddress`: The specific contract to query for events
- `transferSelector`: The event signature hash that identifies `Transfer` events
- `keys`: Filters events by type; only `Transfer` events are returned
- `from_block/to_block`: Defines the block range to search within
- `chunk_size`: Controls pagination to avoid overwhelming responses

The event data is then decoded to extract the information; sender address (`keys[1]`), recipient address (`keys[2]`), and transfer amount (`data[0]`).

# Are variable names in events optional like in Solidity?

As expected, **variable names are NOT optional in Cairo events.** While Solidity allows anonymous event parameters, Cairo requires explicit field names in event struct definitions for all parameters.

# Can events be inherited through parent contracts and interfaces?
Cairo does not support event inheritance. Instead, to reuse events across contracts, you use components. Components define their own events, and when you include a component in your contract, you reference the component's event type in your contract's `#[event]` enum using the `#[flat]` attribute. This allows multiple contracts to emit the same events by using the same component, without needing to redefine the events in each contract.

For a refresher on key differences between events in Solidity and Cairo, here's a table that shows a clear comparison:

## **Events: Key Differences Between Cairo and Solidity**

| Aspect | **Cairo** | **Solidity** |
| --- | --- | --- |
| **Variable Names** | Required for all parameters | Optional (anonymous params allowed) |
| **Indexed Parameters** | `#[key]` attribute (up to 50 indexed params) | `indexed` keyword (max 3, or 4 for anonymous events) |
| **Total Parameters** | No hard limit (practical constraints) | 17 total arguments (arrays count as 2) |
| **Inheritance** | No inheritance - use component embedding | Full inheritance supported |
| **Event Declaration** | `#[derive(starknet::Event)]` struct | `event EventName(...)` |
| **Event Emission** | `self.emit(Event::EventName { ... })` | `emit EventName(...)` |
| **Nested Events** | `#[flat]` attribute for flattening | Not supported |

# **Conclusion**

Cairo events require more explicit structure compared to Solidity and enforce strict type definitions and composition patterns. In Cairo, events rely on three traits working together:

- `Serde` handles serialization of complex fields into `felt252` values.
- `Event` prepares keys and data arrays for receipt.
- `EventEmitter` emits the structured event.

Structs with nested or non-primitive types must derive `Serde` to compile. Indexed fields marked with `#[key]` are stored separately for filtering, use `#[key]` on primitive types like `u32`, `felt252`, or `ContractAddress` for effective querying, as complex types get hashed and become unreadable. The `#[flat]` attribute applies to nested event enums to flatten naming hierarchy, enabling distinct event selectors for better query granularity.

*This article is part of a tutorial series on [Cairo Programming on Starknet](https://rareskills.io/cairo-tutorial)*