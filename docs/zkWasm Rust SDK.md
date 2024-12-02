# zkWasm Rust SDK and Rest ABI
## Overview of zkWasm Rust SDK and Rest ABI

The zkWasm Rust SDK provides essential building blocks for developing zero-knowledge WebAssembly applications. Such as Host(Builtin) Functions like Input/Output, Merkel Tree, Poseidon Signature, etc, and also some useful traits for state management and trace such as key-value pair storage, debug print, etc.

You can import the SDK in your rollup application by adding the following line to your ```Cargo.toml``` file:
```toml
[dependencies]
...
zkwasm-rust-sdk = { git = "https://github.com/DelphinusLab/zkWasm-rust.git", default-features = true }
```

You can view the available modules in the [lib.rs](https://github.com/DelphinusLab/zkWasm-rust/blob/main/src/lib.rs) file.

for example, if you want to use the Merkel Tree, you can import it as follows:
```rust
use zkwasm_rust_sdk::Merkle;
```
And then you can use the Merkel Tree in your code:
```rust
let merkle = zkwasm_rust_sdk::Merkle::new();
```

if you want to debug the state, you can insert the following code into your rust code:
```rust
zkwasm_rust_sdk::dbg!("debug message");
```


Let's explore the key components through the following example, the zkwasm Rest ABI, which defines the interface between the zkWasm rollup and zkWasm Application Server using zkWasm Rust SDK. You can view the full code in [zkwasm-mini-rollup/abi/src/lib.rs](https://github.com/DelphinusLab/zkwasm-mini-rollup/blob/main/abi/src/lib.rs) file.

### 1. Import Key Modules
In the example code, we import the following modules from the zkWasm Rust SDK:

```rust
...
use zkwasm_rust_sdk::jubjub::BabyJubjubPoint;
use zkwasm_rust_sdk::jubjub::JubjubSignature;
use zkwasm_rust_sdk::kvpair::KeyValueMap;
use zkwasm_rust_sdk::Merkle;
...
```
Let's explore how these modules are used in the following sections.

### 1. Storage and State Management

#### Merkle Tree State
```rust
pub static mut MERKLE_MAP: KeyValueMap<Merkle> = KeyValueMap {
    merkle: Merkle { root: [...] }
};
```

- Manages application state using a Merkle tree structure
- Provides cryptographic verification of state changes
- Uses a key-value store for efficient data access

#### StorageData Trait
```rust
pub trait StorageData {
    fn from_data(u64data: &mut IterMut<u64>) -> Self;
    fn to_data(&self, u64data: &mut Vec<u64>);
}
```

- Defines interface for serializing/deserializing state data
- Enables custom data structures to be stored in Merkle tree
- Provides consistent data handling across the application

### 2. Player Management System
When developing a zkWasm application, you need to manage players (or users) and their associated data. The player management system is essential for tracking user accounts and their state.

#### Player Structure
```rust
#[derive(Debug, Serialize)]
pub struct Player<T: StorageData + Default> {
    pub player_id: [u64; 2],
    pub nonce: u64,
    pub data: T,
}
```
Key features:

- Generic implementation allowing custom player data
- Built-in nonce management for transaction ordering
- Serializable for state persistence
- `StorageData` trait bounds ensuring proper state handling

#### Player Operations
```rust
impl<T: StorageData + Default> Player<T> {
    pub fn store(&self) { ... }
    pub fn new_from_pid(pid: [u64; 2]) -> Self { ... }
    pub fn get_from_pid(pid: &[u64; 2]) -> Option<Self> { ... }
    pub fn get_and_check_nonce(pid: &[u64; 2], nonce: u64) -> Self { ... }
}
```

- State persistence with Merkle tree integration
- Player creation and retrieval functionality
- Nonce validation and management

### 3. ZkWasm REST ABI

In the Application side, this macro from zkwasm_rest_abi generates essential WebAssembly bindings for zkWasm applications:
```rust
zkwasm_rest_abi::create_zkwasm_apis!(Transaction, State, Config);
```

#### System Architecture and Flow

![zkWasm System Architecture](./media/minirollup-bundled.png)

Now we delve into the REST service ABI to see how it supports the zkWasm rollup Application. Above figure shows the system execution flow of zkWasm rollup application, and below we will explain each phase in detail.

##### **1. Initialization Phase**

The initialization phase in zkWasm rollup serves two critical purposes:

- Loading previous chunk's finalized state
- Setting up new execution environment for transactions

```rust
pub fn initialize(root: Vec<u64>) {
    // Load initial Merkle root state
    let merkle = zkwasm_rust_sdk::Merkle::load([root[0], root[1], root[2], root[3]]);
    MERKLE_MAP.merkle = merkle;
    $S::initialize();
}
```
The code above:

- Loads initial state from provided Merkle root
- Sets up state for new transaction input bundle
- Called at the start of each transaction chunk processing

##### **2. Transaction Processing Phase**

The transaction processing phase in zkWasm rollup fulfills three essential roles:

- Sequential execution of user transactions
- Maintaining state consistency during updates
- Accumulating settlement information for later batching

```rust
// In zkmain()
for _ in 0..tx_length {
    let mut params = Vec::with_capacity(24);
    for _ in 0..24 {
        params.push(unsafe {wasm_input(0)});
    }
    verify_tx_signature(params.clone());
    handle_tx(params);
}
```
The code above:

- Processes transactions until preemption point
- Each transaction verified and executed sequentially
- State updates tracked in Merkle tree

##### **3. Preemption Check**
The preemption mechanism in zkWasm rollup serves two vital functions:

- Controlling chunk size for computational efficiency
- Ensuring deterministic chunk boundaries

```rust
pub fn preempt() -> bool {
    $S::preempt()  // Check if state reached preemption point
}
```
The code above:

- Determines when to stop processing transactions
- Usually based on state conditions or limits, such as a counter threshold
- Critical for chunk management

```rust
pub fn preempt() -> bool {
    let state = unsafe {&STATE};
    return state.counter >= 20;
}
```
For example, in code above, if the counter is greater than 20, the preemption point is reached and the transaction processing will stop.

##### **4. State Finalization**
The finalization phase in zkWasm rollup serves two critical purposes:

- Processing all pending settlements
- Generating verifiable outputs for the chain

The finalize() function in zkWasm API triggers the settlement process:
```rust
pub fn finalize() -> Vec<u8> {
    unsafe {
        $S::flush_settlement()
    }
}
```

Here's an Example, in the circuit's main function (zkmain()), the finalization produces two key outputs:

- New State Root (root)
    - 4 u64 values representing the Merkle root
    - Captures final state after all transactions
    - Used to initialize next chunk's state
- Settlement Hash (txdata)
    - 4 u64 values containing SHA256 hash
    - Computed from all withdrawals/settlements
    - Used for on-chain verification of batched operations    

```rust
// Step 1: Get settlement data the same way as in the finalize() function
let bytes = $S::flush_settlement();

// Step 2: Generate transaction info hash
let txdata = conclude_tx_info(bytes.as_slice());
// txdata is [u64; 4] - contains SHA256 hash of settlement data

// Step 3: Get final Merkle root
let root = merkle_ref.merkle.root;  // [u64; 4]

// Step 4: Output all data
unsafe {
    // Output Merkle root (4 u64 values)
    wasm_output(root[0]);
    wasm_output(root[1]);
    wasm_output(root[2]);
    wasm_output(root[3]);

    // Output transaction data hash (4 u64 values)
    wasm_output(txdata[0]);
    wasm_output(txdata[1]);
    wasm_output(txdata[2]);
    wasm_output(txdata[3]);
}
```
The code above:

- Generates new Merkle root for next chunk
- Outputs settlement information
- Ensures state consistency across chunks


#### Rollup Development Requirements

In application side, the following requirements (API) shall be met to ensure the rollup process works properly:

##### **1. State Implementation**
```rust
impl State {
    fn preempt() -> bool { ... }  // Define chunk boundary conditions
    fn initialize() { ... }        // Setup initial state
    fn flush_settlement() -> Vec<u8> { ... } // Prepare state updates
}
```

And some other functions can be implemented in the `State` trait.
```rust
pub trait State {
    fn preempt() -> bool;
    fn initialize();
    fn flush_settlement() -> Vec<u8>;
    // Some other functions
    fn get_state(pid: Vec<u64>) -> String { ... } // Get state for specific player ID
    fn snapshot() -> String { ... } // Get current global state
    fn tick(&mut self) { ... } // Update state if auto-tick is enabled
    fn store(&self) { ... } // Store state to Merkle tree
    fn new() -> Self { ... } // Create new state instance
    fn rand_seed() -> u64 { ... } // Generate random seed
}
```

##### **2. Transaction Processing**
```rust
impl Transaction {
    fn decode(command: [u64; 4]) -> Self { ... } // Decode Command      
    fn process(&self, user_address: &[u64; 4], sig_r: &[u64; 4]) -> u32 { ... } // Process Transaction
    fn install_player(&self, pkey: &[u64; 4]) -> u32 { ... } // Create player / user Account
}
```

##### **3. Configuration Management**
```rust
impl Config {
    fn autotick() -> bool { ... } // Auto-tick configuration
    fn to_json_string() -> String { ... } // Convert to JSON string
}
```

#### Example Rollup Flow
```rust
// 1. Initialize state with previous Merkle root
initialize(previous_root);

// 2. Process transaction batch
while !preempt() {
    verify_tx_signature(params);
    handle_tx(params);
}

// 3. Generate new Merkle root and settlement data
finalize();
// ...
```

#### Function Overview

##### **1. Transaction Handling**
```rust
pub fn handle_tx(params: Vec<u64>) -> u32 // Process transaction
```

- Params format (24 u64 values)
    - [0-3]: Command data
    - [4-7]: User address
    - [20-23]: Signature data
- Returns error code (0 for success)

##### **2. State Management**
```rust
pub fn get_state(pid: Vec<u64>) -> String // Get state for specific player ID
pub fn snapshot() -> String // Get current global state
```

- `get_state`: Retrieves state for specific player ID
- `snapshot`: Gets current global state
- Both return JSON-formatted strings

##### **3. Configuration and Control**
```rust
pub fn get_config() -> String // Get configuration
pub fn preempt() -> bool  // Preemption check
pub fn autotick() -> bool // Auto-tick configuration
pub fn randSeed() -> u64  // Random seed
```

- Configuration retrieval and system controls
- All WebAssembly-exposed functions