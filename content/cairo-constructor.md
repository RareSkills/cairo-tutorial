# Constructors in Cairo

A constructor is a one-time-call function executed during contract deployment to initialize state variables, perform contract setup tasks, make cross-contract interactions and so on.

In Cairo, constructors are defined using the `#[constructor]` attribute inside the `mod` block of a contract.

This article will cover how constructors work in Cairo, manual and automatic serialization of constructor arguments inside Scarb tests, and how constructor return values differ from Solidity’s.

## A Simple Cairo Constructor

Let’s take a simple Solidity contract that initializes a state variable `count` in its constructor:

```solidity
contract Counter {
    uint256 public count;

    constructor(uint256 _count) {
        count = _count;
    }
}

```

Here’s the equivalent Cairo version:

```rust
#[starknet::interface]
pub trait IHelloStarknet<TContractState> {
    fn get_count(self: @TContractState) -> felt252;
}

#[starknet::contract]
mod HelloStarknet {
    use starknet::storage::{StoragePointerReadAccess,StoragePointerWriteAccess};

    #[storage]
    struct Storage {
        count: felt252
    }

    // ************ CONSTRUCTOR FUNCTION ************* //
    #[constructor]
    fn constructor(ref self: ContractState, _count: felt252) {
        self.count.write(_count);
    }

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn get_count(self: @ContractState) -> felt252 {
            self.count.read()
        }
    }
}

```

The `#[constructor]` attribute in the above code marks the function as the contract’s constructor. The function must be named `constructor` and is executed once during deployment. It takes `ref self` to allow write access to the contract’s storage, along with any parameters needed for initialization, in this case, `_count`.

The constructor above simply initializes the `count` storage variable with the value passed as an argument.

Let’s test it out, create a new project with Scarb:

```bash
scarb new counter
```

Next, replace the generated code in the `src/lib.cairo` file with the `HelloStarknet` contract code from above.

To verify that `count` is correctly initialized, we will write a test that deploys the contract with a specific value and then asserts that the stored value matches what we passed.

Open the test file (`tests/test_contract.cairo`), then replace the generated code with the one below:

```rust
use counter::{IHelloStarknetDispatcher, IHelloStarknetDispatcherTrait};
use snforge_std::{ContractClassTrait, DeclareResultTrait, declare};
use starknet::ContractAddress;

fn deploy_contract(name: ByteArray) -> ContractAddress {
    let contract = declare(name).unwrap().contract_class();

    // CREATE ARGUMENT ARRAY FOR CONSTRUCTOR. WE'RE PASSING `5` AS THE INITIAL VALUE FOR `count`
    let mut args = ArrayTrait::new();
    args.append(5_felt252);

    // DEPLOY THE CONTRACT WITH THE PROVIDED CONSTRUCTOR ARGUMENTS
    let (contract_address, _) = contract.deploy(@args).unwrap();

    contract_address // Return address of deployed contract
}

#[test]
fn test_count() {
    let contract_address = deploy_contract("HelloStarknet");

    let dispatcher = IHelloStarknetDispatcher { contract_address };

    // CALL THE `get_count` FUNCTION TO READ THE CURRENT VALUE OF `count`
    let result = dispatcher.get_count();

    // ASSERT THAT THE INITIALIZED VALUE MATCHES WHAT WE PASSED DURING DEPLOYMENT
    assert!(result == 5, "failed {} != 5", result);
}

```

The key part of this test is how we pass the constructor argument **as an array of `felt252` values** during deployment. This part is easy to miss, but it is important; we append the value `5` to the `args` array before calling `deploy`, which is how we initialize the contract with `5`.

```rust
// CREATE ARGUMENT ARRAY FOR CONSTRUCTOR. WE'RE PASSING `5` AS THE INITIAL VALUE FOR `count`
let mut args = ArrayTrait::new();
args.append(5_felt252);

// DEPLOY THE CONTRACT WITH THE PROVIDED CONSTRUCTOR ARGUMENTS
let (contract_address, _) = contract.deploy(@args).unwrap();

```

The rest of the test confirms that this initialization worked as expected. We call `get_count`, which returns the current value of the `count` variable, and then assert that it's equal to `5`.

Unlike Solidity's foundry test, where constructor arguments can be of various types (integers, strings, arrays, structs, addresses, etc.), contract deployment inside Scarb tests requires all constructor arguments to be passed as `felt252` values. This is because the Starknet VM is a `felt252`-based system, everything is encoded and passed around as felts under the hood.

That said, we don’t always need to manually convert every argument into felts ourselves during deployment in test. Starknet Foundry provides a helper function that automatically handles constructor-argument serialization and contract deployment in tests. We will cover that shortly but before using the helper, we will learn how to serialize primitive and complex types so we know what the helper is doing behind the scenes.

## Passing Primitive Types Other Than `felt252` to the Constructor

As mentioned earlier, we can manually serialize non-`felt252` values (such as `ContractAddress`) into `felt252` during deployment, pass them as an array of `felt252`, and then decode them in the constructor to initialize the contract’s state. Let’s look at an example to see how this works in practice.

The Solidity version:

```solidity
contract SomeContract {
    uint256 count;
    address owner;
    bool isActive;

    constructor(uint256 _count, address _owner, bool _isActive) {
        count = _count;
        owner = _owner;
        isActive = _isActive;
    }
}

```

Here’s the equivalent in Cairo:

```rust
#[starknet::contract]
mod HelloStarknet {    
    use starknet::ContractAddress;
    use starknet::storage::{StoragePointerWriteAccess};

    // Define the contract's storage.
    #[storage]
    struct Storage {
        count: u256,
        owner: ContractAddress,
        is_active: bool,
    }

        // CONSTRUCTOR FUNCTION
        #[constructor]
        fn constructor(
                ref self: ContractState,
                _count: u256,
                _owner: ContractAddress,
                _is_active: bool
            ) {

            // INIT STATE VARS
            self.count.write(_count);
            self.owner.write(_owner);
            self.is_active.write(_is_active);
        }
}

```

Here the constructor accepts three arguments:

- `_count`: The 256-bit count value.
- `_owner`: The address of the contract owner.
- `_is_active`: A boolean flag indicating whether the contract should start in an active state.

Then writes each of the provided arguments into their corresponding storage variables using the `.write()` method, thereby initializing the states on deployment.

### Manually Serializing Constructor Arguments Before Deployment

In the test, the function below `deploy_contract` shows how values that are not of type `felt252` are serialized “manually” before being passed as constructor arguments:

```rust
fn deploy_contract(name: ByteArray) -> ContractAddress {
    let contract = declare(name).unwrap().contract_class();

    // CREATE ARGUMENT ARRAY FOR CONSTRUCTOR
    let mut args = ArrayTrait::new();

    // VALUES TO SERIALIZE
    let count: u256 = 115792089237316195423570985008687907853269984665640564039457584007913129639935;     // count = max value of u256
    let owner: ContractAddress = 0xbeef.try_into().unwrap();
    let is_active: bool = true;

    // Serialize the u256 `count` value into two felt252 elements (low, high)
    // and push them into the constructor `args` array.
    count.serialize(ref args);
    
    // SERIALIZE INTO FELT252, THEN PUSH TO `args` ARRAY
    owner.serialize(ref args);       // ContractAddress -> felt252
    is_active.serialize(ref args);   // bool -> felt252
    

    // DEPLOY THE CONTRACT WITH THE PROVIDED CONSTRUCTOR ARGUMENTS
    let (contract_address, _) = contract.deploy(@args).unwrap();
    
    contract_address
}

```

When dealing with multiple constructor arguments in Starknet, they must be passed as an `Array<felt252>`. 

In this example, because the `count` variable is of type `u256`, the `.serialize()` method splits it value into two 128-bit halves; `low` and `high` before being appended to the array `args`. As we said earlier, this is because a single `felt252` cannot safely hold a full 256-bit integer. By serializing the values properly (*encoding each half as a separate `felt252`*), the constructor function can automatically deserialize them into the original `u256` value.

The remaining constructor values; `owner` (a `ContractAddress`) and `is_active` (a `bool`) each fit into a single `felt252`, so serializing them is straightforward. Serializing them in the exact order that matches the constructor's parameter sequence is important as the deployment will fail or produce unexpected behavior if the arguments don't align or correspond with the constructor's expected parameter order.

The final step uses the `@` operator when passing the array to `contract.deploy(@args)`, which is Starknet's way of passing array data without transferring ownership.

In the next section, we will look at how to automate constructor argument serialization in tests using a helper function that performs all of the above serialization steps for us.

## Passing Complex Type

Just like primitive types, complex constructor arguments must also be serialized into an array of `felt252` values before deployment.

Let’s consider this Solidity contract below that initializes a complex type (a `struct`) via the constructor:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.30;

contract Bank {
    struct Account {
        address wallet;
        uint64 balance;
    }

    Account userAccount;

    constructor(Account memory _userAccount) {
        userAccount = _userAccount;
    }
}
```

This is straightforward in Solidity.

Below is the equivalent in Cairo:

```rust
#[starknet::contract]
pub mod HelloStarknet {
    use starknet::ContractAddress;
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    // `Account` STRUCT
    #[derive(Drop, starknet::Store)]
    pub struct Account {
        pub wallet: ContractAddress,
        pub balance: u64,
    }

    #[storage]
    struct Storage {
        // Use the `Account` struct in storage
        pub user_account: Account,
    }

    // Constructor function
    #[constructor]
    // Each field is declared as its own constructor argument
    fn constructor(
            ref self: ContractState,
            new_user_account: Account,
        ) {
 
        // WRITE `new_user_account` STRUCT TO STORAGE
        self.user_account.write(new_user_account);
        
    }
}

```

Here, the contract module is marked as `pub mod`, making it a public module. This allows external code, such as tests or other modules, to access items defined inside it, like the `Account` struct. Since `Account` is also declared as `pub`, it becomes fully accessible outside the contract module, which is necessary for serializing this struct in the test.

The `starknet::Store` trait derived on the `Account` struct is what allows Cairo to correctly serialize and deserialize the struct for storage. Without this trait, the compiler would not know how to handle the struct when writing to or reading from storage.

Lastly, the constructor takes a single parameter, `new_user_account` of type `Account`, and writes it directly into storage.

## Automatic Constructor Argument Serialization & Deployment with `deploy_for_test`

This method handles all the heavy lifting for us by using a helper function provided by Starknet Foundry  `deploy_for_test`, which automatically serializes constructor inputs before deployment. 

With this approach, there is no need to manually build an `Array<felt252>` or call `.serialize()` on each argument. Instead, the helper function reads the constructor signature of the contract and performs the correct encoding behind the scenes, ensuring every parameter is serialized exactly as the contract expects.

The `deploy_for_test` function signature:

![An image of deploy_for_test function signature showing its parameters and return type](Constructors%20in%20Cairo/Screenshot_2025-12-07_at_06.54.29.png)

**`deploy_for_test` function parameters**

In the red box, are the `deploy_for_test` parameters. The first two are:

- `class_hash`: the compiled class hash of the contract
- `deployment_params`: a structure contains fields needed to deploy a new contract instance

After these two fixed parameters, the function takes as many constructor arguments as the contract defines:

- `<constructor_param1>: <constructor_param_type1>`
- `<constructor_param2>: <constructor_param_type2>`
- …
- `<constructor_paramN>: <constructor_param_typeN>`

In other words, parameters 1 through N maps directly to the constructor parameters of the contract. For example, given a constructor like:

```rust
#[constructor]
fn constructor(
                ref self: ContractState, 
                count: u256, 
                owner: ContractAddress, 
                is_active: bool
            ) {
    ...
}
```

The `deploy_for_test` call would look like:

```rust
deploy_for_test(
    // **** First two params - START **** //
    class_hash,
    deployment_params,
    // **** First two params - END **** //
    
    
    // **** Constructor params - START **** //
    count,
    owner,
    is_active,
    // **** Constructor params - END **** //
);
```

**`deploy_for_test` function return type**

![An image showing the return type of deploy_for_test function](Constructors%20in%20Cairo/de3bc561-47d3-4b6f-8fed-c06da4d9fc7d.png)

In the blue box is the return type. The function returns a `Result`, which can be one of two outcomes:

- `Ok(..)`: meaning the deployment succeeded.
    
    It returns a tuple containing:
    
    1. the `ContractAddress` of the newly deployed contract, and
    2. a `Span<felt252>` representing any values returned by the constructor (covered later in this chapter).
- `Err(..)`: meaning the deployment failed.
    
    In this case, the function returns an `Array<felt252>` containing the error data emitted by the failed deployment.
    

**Practical Example**

To use this function in practice, let’s deploy the contract shown earlier, the one whose constructor takes a single `Account` parameter. Because both the module (contract) and the `Account` struct were marked as `pub`, the test environment can import and serialize them automatically.

Below is a full example demonstrating how to declare the contract, prepare deployment parameters, construct the `Account` argument, and finally deploy the contract using `deploy_for_test`:

```rust
// ********* NEW IMPORTS - START ********* //
use myconstructor::HelloStarknet::{Account, deploy_for_test};
use starknet::deployment::DeploymentParams;
// ********** NEW IMPORTS - END ********** //

use myconstructor::{IHelloStarknetDispatcher, IHelloStarknetDispatcherTrait};
use snforge_std::{DeclareResult, DeclareResultTrait, declare};
use starknet::ContractAddress;

fn deploy_contract(name: ByteArray) -> ContractAddress {
    // 1. Declare contract to get the class_hash
    let declare_result: DeclareResult = declare(name).unwrap();
    let class_hash = declare_result.contract_class().class_hash;

    // 2. Create deployment parameters
    let deployment_params = DeploymentParams { salt: 0, deploy_from_zero: true };

    // 3. Create new account
    let new_account = Account { wallet: 'BOB'.try_into().unwrap(), balance: 5 };

    // 4. Use `deploy_for_test` to deploy the contract
    // It automatically handles serialization of constructor parameters
    let (_contract_address, _) = deploy_for_test(*class_hash, deployment_params, new_account)
        .expect('Deployment failed');

    _contract_address
}
```

- `use myconstructor::HelloStarknet::{Account, deploy_for_test};`
    
    Here, `myconstructor` is the project name, and `HelloStarknet` is the contract module defined inside that project. By importing the HelloStarknet contract module, we have access to the `Account` struct and the `deploy_for_test` helper function.
    
- `use starknet::deployment::DeploymentParams;`
    
    This import brings in `DeploymentParams`, a struct provided by the Starknet core library. It lets configuration of how the contract should be deployed (e.g using a custom salt or deploying from zero). It is always required when calling `deploy_for_test`, because that function expects deployment parameters as its second argument.
    

Lastly, the `deploy_contract` function brings everything together. It first declares the contract to obtain its `class_hash`, then prepares the required `DeploymentParams`, constructs a new `Account` struct that will be passed to the contract’s constructor, and finally calls `deploy_for_test`.

**Exercise:** Solve the `constructor` exercise in [the Cairo-Exercises](https://github.com/RareSkills/Cairo-Exercises) repo.

## Return Values in Constructors

In Solidity, a constructor never returns a value. During deployment, the EVM executes the constructor and treats its only “output” as the runtime bytecode to store on-chain.

Cairo on the other hand works differently. After deployment, it returns a tuple: `(ContractAddress, Span<felt252>)`.

- `ContractAddress`: the deployed contract’s address.
- `Span<felt252>`: a span of `felt252` values holding the constructor’s return data. Any type other than `felt252` is automatically converted before being placed here.

To demonstrate this, lets bootstrap a new scarb project:

```bash
scarb new rett
```

Then we add a constructor to our generated contract in `lib.cairo`, like so:

```rust

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

    // ************************ NEWLY ADDED - START ***********************//
    #[constructor]
    fn constructor(ref self: ContractState) -> felt252 {
        33
    }
    // ************************ NEWLY ADDED - END ************************//

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

To keep things simple, our constructor will just return the value `33`.

To show that the constructor actually returns a value, let’s navigate to the test file `test_contract.cairo` and replace the generated code with this (*simplified the code for readability*):

```rust
use rett::{IHelloStarknetDispatcher, IHelloStarknetDispatcherTrait};
use snforge_std::{ContractClassTrait, DeclareResultTrait, declare};
use starknet::ContractAddress;

fn deploy_contract(name: ByteArray) -> ContractAddress {
    let contract = declare(name).unwrap().contract_class();
    let (contract_address, _) = contract.deploy(@ArrayTrait::new()).unwrap();
    contract_address
}

#[test]
fn test_increase_balance() {
    let contract_address = deploy_contract("HelloStarknet");
}

```

We will make changes to the highlighted part in this screenshot:

![Screenshot of the test code, highlighting the part we will make changes to](Constructors%20in%20Cairo/Screenshot_2025-12-09_at_21.10.38.png)

**The updated test code**

What changed are:

1. Return type: `deploy_contract` function now returns a tuple `(ContractAddress, felt252)` instead of just `ContractAddress`.
2. Constructor output capture: Introduced `ret_vals` of type `Span<felt252>` to hold the constructor’s return values.
3. Tuple return: We return the contract address alongside the first element of `ret_vals`, since the constructor only returns a single value.

Finally, the test asserts that the constructor’s return value is `33`, confirming that the value is correctly passed back during deployment.

```rust
use rett::{IHelloStarknetDispatcher, IHelloStarknetDispatcherTrait};
use snforge_std::{ContractClassTrait, DeclareResultTrait, declare};
use starknet::ContractAddress;

// Change return type to a tuple so we can capture the constructor’s return value.
fn deploy_contract(name: ByteArray) -> (ContractAddress, felt252) {
    let contract = declare(name).unwrap().contract_class();

    // Capture both the contract address and the constructor’s return values (as a Span<felt252>).
    let (contract_address, ret_vals) = contract.deploy(@ArrayTrait::new()).unwrap();

    // Return the address plus the first element in ret_vals (we expect only one value).
    (contract_address, *ret_vals.at(0))
}

#[test]
fn test_increase_balance() {
    let (_, ret_val) = deploy_contract("HelloStarknet");

        // Verify that the constructor actually returned 33 as expected.
    assert(ret_val == 33, 'Invalid return value.');
}

```

To confirm, run `scarb test`, the test should pass. In a later article we will see how to deploy a contract directly from another.

## Payable-Like Constructor in Starknet

While STRK behaves like an ERC-20 token, it also serves as Starknet’s native fee token. However, Starknet does not have a true “native token” in the same sense as Ethereum’s ETH. As a result, Cairo does not support “payable” constructors. If we want to enforce that a contract has a certain STRK balance at deployment, we can transfer STRK to the predicted address, then assert in the constructor that the balance of the contract is at least the desired amount.

*This article is part of a tutorial series on [Cairo Programming on Starknet](https://rareskills.io/cairo-tutorial)*