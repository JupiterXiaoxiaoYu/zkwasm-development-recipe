
## Host Environment (/host)
The Host Environment provides the foundational runtime infrastructure for zkWasm applications, implementing essential services for cryptographic operations, state management, and WebAssembly integration. It serves as the critical bridge between WebAssembly applications and the underlying system resources. 

You can find the implementation in the `/host` directory of the zkWasm Mini Rollup repository. The main components are organized as follows:

```
host/
├── src/
│   ├── context/     // Context management implementations
│   ├── jubjub.rs    // Elliptic curve operations
│   ├── poseidon.rs  // Hash function implementation
│   └── lib.rs       // Core API and WebAssembly bindings
```

### Core Context System
The context system forms the backbone of the Host Environment, managing different execution contexts through thread-safe implementations. Here's how to use the various contexts:

```rust
use std::sync::Mutex;
use context::{
    datacache::CacheContext,
    jubjub::sum::BabyJubjubSumContext,
    merkle::MerkleContext,
    poseidon::PoseidonContext
};

// Global context initialization
lazy_static::lazy_static! {
    // CacheContext: Manages temporary data storage during proof generation
    pub static ref DATACACHE_CONTEXT: Mutex<CacheContext> = Mutex::new(CacheContext::new());
    
    // MerkleContext: Handles state tree operations for your application's state
    pub static ref MERKLE_CONTEXT: Mutex<MerkleContext> = Mutex::new(MerkleContext::new(0));
    
    // PoseidonContext: Provides cryptographic hash functions for state commitment
    pub static ref POSEIDON_CONTEXT: Mutex<PoseidonContext> = Mutex::new(PoseidonContext::default(0));
    
    // JubjubContext: Manages elliptic curve operations for signatures
    pub static ref JUBJUB_CONTEXT: Mutex<BabyJubjubSumContext> = Mutex::new(BabyJubjubSumContext::default(0));
}
```
These contexts are thread-safe and globally accessible throughout your application's runtime.

### Data Cache Operations
The data cache system provides temporary storage during proof generation:

```rust
#[wasm_bindgen]
pub fn cache_set_mode(mode: u64) {
    // Set the cache mode: STORE_MODE for writing, READ_MODE for reading
    DATACACHE_CONTEXT.lock().unwrap().set_mode(mode);
}

#[wasm_bindgen]
pub fn cache_store_data(data: u64) {
    // Store a piece of data in the cache
    DATACACHE_CONTEXT.lock().unwrap().store_data(data);
}

#[wasm_bindgen]
pub fn cache_fetch_data() -> u64 {
    // Retrieve data from the cache
    DATACACHE_CONTEXT.lock().unwrap().fetch_data()
}
```

### State Management and Cryptographic Operations

#### Poseidon Hash Operations
```rust
#[wasm_bindgen]
pub fn poseidon_new(arg: u64) {
    // Initialize a new Poseidon hash computation
    // arg: size of the input to be hashed
    POSEIDON_CONTEXT.lock().unwrap().poseidon_new(arg as usize);
}

#[wasm_bindgen]
pub fn poseidon_push(arg: u64) {
    // Add a value to the current hash computation
    // Used to build up the state that needs to be hashed
    POSEIDON_CONTEXT.lock().unwrap().poseidon_push(arg);
}

#[wasm_bindgen]
pub fn poseidon_finalize() -> u64 {
    // Complete the hash computation and return the result
    // The hash can be used as a commitment to the state
    POSEIDON_CONTEXT.lock().unwrap().poseidon_finalize()
}
```

#### Merkle Tree State Management
```rust
#[wasm_bindgen]
pub fn merkle_setroot(arg: u64) {
    // Set the root of the Merkle tree
    // Used when initializing your application's state
    MERKLE_CONTEXT.lock().unwrap().set_root(arg);
}

#[wasm_bindgen]
pub fn merkle_get() -> u64 {
    // Get the current value at the selected Merkle tree node
    // Used to read state values from your application
    MERKLE_CONTEXT.lock().unwrap().get()
}

#[wasm_bindgen]
pub fn merkle_set(arg: u64) {
    // Update a value in the Merkle tree
    // Used when your application modifies its state
    MERKLE_CONTEXT.lock().unwrap().set(arg);
}
```

!!!info 
    For host functions, you can also refer to [zkWasm Overview](zkWasm Overview.md#3-host-builtin-functions)