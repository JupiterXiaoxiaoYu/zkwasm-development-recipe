## Database Service (/dbservice)
The Database Service is a crucial component of the zkWasm Mini Rollup system, providing persistent storage and state management for zkWasm applications. It implements a MongoDB-based Merkle tree structure for secure and verifiable state storage, along with JSON-RPC endpoints for state access and modification.

### Architecture
The Database Service consists of three main components:

1. Merkle Tree Storage
    - Implements a MongoDB-backed Merkle tree
    - Provides cryptographic verification of state
    - Supports efficient state updates and proofs

2. Data Hash Records
    - Stores arbitrary data with hash verification
    - Maintains data integrity through cryptographic hashing
    - Enables efficient data retrieval

3. JSON-RPC Interface
    - Exposes state management endpoints
    - Handles concurrent state updates
    - Provides performance metrics

### Core Data Structures
The service defines several key data structures for state management:
```rust
// Request structure for updating Merkle tree leaves
#[derive(Clone, Deserialize, Serialize)]
pub struct UpdateLeafRequest {
    root: [u8; 32],      // Current Merkle root
    data: [u8; 32],      // New leaf data
    index: String,       // Leaf index (u64 encoded as string)
}

// Request structure for retrieving Merkle tree leaves
#[derive(Clone, Deserialize, Serialize)]
pub struct GetLeafRequest {
    root: [u8; 32],      // Merkle root to query
    index: String,       // Leaf index to retrieve
}

// Request structure for storing data records
#[derive(Clone, Deserialize, Serialize)]
pub struct UpdateRecordRequest {
    hash: [u8; 32],      // Hash of the data
    data: Vec<String>,   // Actual data as vector of u64 strings
}

// Request structure for retrieving data records
#[derive(Clone, Deserialize, Serialize)]
pub struct GetRecordRequest {
    hash: [u8; 32],      // Hash of the data to retrieve
}
```

### JSON-RPC API Endpoints
The service exposes several JSON-RPC endpoints for state management:
#### Merkle Tree Operations
```rust
// Update a leaf in the Merkle tree
async fn update_leaf(request: UpdateLeafRequest) -> Result<[u8; 32], Error> {
    // Parameters:
    // - root: Current Merkle root
    // - data: New leaf data
    // - index: Position to update
    // Returns:
    // - New Merkle root after update
}

// Retrieve a leaf from the Merkle tree
async fn get_leaf(request: GetLeafRequest) -> Result<[u8; 32], Error> {
    // Parameters:
    // - root: Merkle root to query
    // - index: Leaf position
    // Returns:
    // - Leaf data with proof
}
```

#### Data Record Operations
```rust
// Store a new data record
async fn update_record(request: UpdateRecordRequest) -> Result<(), Error> {
    // Parameters:
    // - hash: Hash of the data
    // - data: Vector of u64 values as strings
    // Returns:
    // - Success/failure status
}

// Retrieve a data record
async fn get_record(request: GetRecordRequest) -> Result<Vec<String>, Error> {
    // Parameters:
    // - hash: Hash of the data to retrieve
    // Returns:
    // - Vector of u64 values as strings
}
```
