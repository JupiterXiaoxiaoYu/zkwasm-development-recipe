# zkWasm Mini Rollup Overview

As a rest service, zkWasm Mini Rollup provides a way for transaction handling and state management (i.e. Database service) for zkWasm server side applications. 

You may have delved into the details of how its ABI constructs the execution flow of rollup through [zkWasm Rust SDK](https://github.com/DelphinusLab/zkWasm-rust). Now we will walk through other implementation details of zkWasm Mini Rollup.

## zkWasm Mini Rollup Package Structure

Remind that [zkWasm Mini Rollup Repository](https://github.com/DelphinusLab/zkwasm-mini-rollup) is a monorepo that contains multiple packages:

```
- ABI (Application Binary Interface)
  - Location: /abi
  - Purpose: Defines the interface between the WASM application and the host environment
  - Key Features:
    - Implements the bundled logic within the zkmain function
    - Handles transaction verification and processing
    - Manages state transitions and merkle tree operations

- Host Environment
  - Location: /host
  - Purpose: Provides the runtime environment for the WASM application
  - Components:
    - WASM bootstrap implementation
    - Host API interfaces
    - Runtime service management

- Database Service
  - Location: /dbservice
  - Purpose: Manages data persistence and state management
  - Features:
    - Database interface implementations
    - State management utilities
    - Data persistence layer

- TypeScript Server Implementation
  - Location: /ts
  - Purpose: Frontend toolkit and service implementation
  - Components:
    - REST service implementation
    - WASM bootstrapping
    - Worker thread management

- Convention
  - Location: /convention
  - Purpose: Defines shared utilities for application development
  - Contents:
    - Common interfaces
    - Shared types and constants
    - Project-wide utilities
```

As we have covered the ABI in [zkWasm Rust SDK](https://github.com/DelphinusLab/zkWasm-rust), we will focus on the other packages.

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

## Server Implementation (zkWasm-ts-Server) (/ts)
The Server implementation serves as the service layer of the zkWasm Mini Rollup system, providing essential functionality for application bootstrapping, transaction processing, proof generation, and service management. This implementation bridges the gap between client applications and the underlying zkWasm infrastructure. 

### Core Service Architecture (ts/service.ts)
The Service layer is organized into several key components that work together to provide a complete rollup service:

- Transaction batch processing and scheduling
- State synchronization between different system components
- Service lifecycle management including startup and shutdown
- Error handling and recovery mechanisms

Let's take a look at the core implementation of the service layer [service.ts](https://github.com/DelphinusLab/zkwasm-mini-rollup/blob/main/ts/src/service.ts). This file contains the main logic for starting and running the zkWasm Mini Rollup Appication. 

Remind how to run the zkWasm Mini Rollup Application, you can find the example code in the [helloworld-rollup](https://github.com/riddles-are-us/helloworld-rollup/blob/main/ts/src/service.ts):

```ts
import { Service } from "zkwasm-ts-server";
const service = new Service(() => { return; });
service.initialize();
service.serve();
```

We can notice that the `Service` class is instantiated at first, here's how the `Service` class is defined:

```ts
export class Service {
    worker: null | Worker;
    queue: null | Queue;
    txCallback: (arg: TxWitness) => void;
    
    constructor(cb: (arg: TxWitness) => void) {
        this.worker = null;
        this.queue = null;
        this.txCallback = cb;
    }
}
```

The Service class is the core orchestrator for the zkWasm Mini Rollup system. It manages transaction processing, job queuing, and callback handling through a worker-based architecture, here's the explanation of the components:

1. Worker Management (worker: null | Worker)
    - Purpose: Handles computationally intensive tasks like proof generation
    - Type: Worker from the bullmq package
    - Initially set to null and initialized during service startup
    - Used for:
        - Running proof generation tasks in the background
        - Processing transactions asynchronously
        - Managing computational resources

2. Queue Management (queue: null | Queue)
    - Purpose: Manages transaction processing order and scheduling
    - Type: Queue from the bullmq package
    - Initially set to null and initialized during service startup
    - Used for:
        - Maintaining transaction order
        - Ensuring sequential processing
        - Managing backpressure

3. Transaction Callback (txCallback: (arg: TxWitness) => void)
    - Purpose: Handles post-transaction processing notifications
    - Type: Function that takes a TxWitness argument
    - Called after successful transaction processing
    - Used for:
        - Notifying external systems of transaction completion
        - Triggering state updates
        - Handling transaction-specific logic


After the Service class is instantiated, the `initialize` method is called, which sets up necessary connections and bootstraps the application:
```ts
async initialize() {
    // Connect to MongoDB
    await mongoose.connect(get_mongoose_db());

    const db = mongoose.connection; 
    // Handle connection error
    db.on('error', () => {
      console.error('fatal: mongoose connection error ... process will terminate');
      process.exit(1);
    });
    // Handle connection success
    db.once('open', () => {
      console.log('Connected to MongoDB');
    });
    
    // Setup Redis connection
    const connection = new IORedis(
      {
        host: redisHost,  // Your Redis server host
        port: 6379,        // Your Redis server port
        reconnectOnError: (err) => {
          console.log("reconnect on error", err);
          return true;
        },
        maxRetriesPerRequest: null  // Important: set this to null
      }
    );

    // Bootstrap application
    await initBootstrap();
    await initApplication(bootstrap);
}
```

It bootstraps WASM by implementing zkWasm's host interfaces which are another WASM image that is preloaded before loading the main WASM image. The implementation of these host APIs can be found in ts/src/bootstrap/ which is compiled from the rust bootstrap code in host directory.

### Bootstrap (ts/bootstrap)
The Bootstrap process, as mentioned above, serves as the critical bridge between the service layer and the WebAssembly-based host environment. It provides the necessary bindings and initialization logic to enable seamless communication between different components.

```
bootstrap/
├── bootstrap.d.ts     # TypeScript definitions for WASM bindings
├── bootstrap.js       # Main bootstrap implementation
├── bootstrap_bg.wasm  # WebAssembly binary
├── dbprocess.ts      # Database processing utilities
└── rpcbind.js        # RPC binding implementation
``` 

#### Host Function Bindings
The bootstrap directly interfaces with the host environment (/host directory) by providing TypeScript bindings for the core host functions:

```
// Host function bindings examples
export function cache_set_mode(mode: bigint): void;
export function cache_store_data(data: bigint): void;
export function cache_fetch_data(): bigint;
```

These bindings correspond to the Rust implementations in the host environment:

```rust
// In host/src/lib.rs
#[wasm_bindgen]
pub fn cache_set_mode(mode: u64) { ... }
pub fn cache_store_data(data: u64) { ... }
pub fn cache_fetch_data() -> u64 { ... }
```

And the bootstrap will be compiled into WASM binary and preloaded before loading the main WASM image to provide the host functions for the application.

#### Database Operations
The zkWasm Mini Rollup system implements a multi-layered RPC architecture for database operations, consisting of three main components:

1. RPC Binding Layer (rpcbind.js)

    ```ts
    // High-level interface exposing database operations
    export function get_leaf(root, index) { ... }  // Retrieve a leaf node from the Merkle tree
    export function update_leaf(root, index, data) { ... }  // Update a leaf node in the Merkle tree
    export function get_record(hash) { ... }  // Retrieve a record by its hash
    export function update_record(hash, data) { ... }  // Update a record's data
    ```

    This layer provides high-level functions for database operations with built-in performance monitoring. Each function formats the JSON-RPC request and delegates to syncrpc.cjs through the `requestMerkleData` function.

2. Synchronous RPC Layer (syncrpc.cjs)

    ```ts
    function requestMerkleData(requestData) {
        // Manages a persistent Node.js child process running dbprocess.js
        // Handles synchronous IPC communication
    }
    ```

    This intermediate layer maintains a persistent Node.js child process running dbprocess.js. It implements synchronous inter-process communication (IPC) using file descriptors for reliable data exchange.

3. Database Process Layer (dbprocess.ts)

    ```ts
    export class MerkleServiceRpc {
        private baseUrl: string;
        private instance;

        constructor(baseUrl: string) {
            this.baseUrl = baseUrl;
            this.instance = axios.create({
                baseURL: this.baseUrl,
                headers: { 'Content-Type': 'application/json' }
            });
        }

        public async queryDB(request: any): Promise<void> {
            // Handles actual HTTP communication with the Merkle tree database service
        }
    }
    ```

    This layer runs as a separate process and handles the actual HTTP communication with the Merkle tree database service. It processes JSON-RPC requests and manages the HTTP connection pool.

Communication Flow

1. Client code calls exported functions from rpcbind.js
2. rpcbind.js formats the JSON-RPC request and calls requestMerkleData from syncrpc.cjs
3. syncrpc.cjs writes the request to the child process's stdin and waits for response on stdout
4. dbprocess.ts receives the request through stdin, makes an HTTP request to the Merkle service, and writes the response back to stdout
5. The response flows back through the layers to the original caller

Remind that we initialize the application by:

```ts
await initBootstrap();
await initApplication(bootstrap);
```

This will allow the application to use the specific WebAssembly module and the infrastructure provided by the bootstrap module.

### Server Configuration (ts/service.ts)

After bootstrapping the application, the initialization process checks the environment variables.

We have five environment variables that can be set to configure the server:

1. `process.env.DEPLOY`: set to `TRUE` if you want to interact with the zkWasm hub such as submitting tasks for your application.
2. `process.env.REMOTE`: set to `TRUE` if you want to get the latest merkle root from the latest proof generated by the zkWasm hub for your application.
3. `process.env.MIGRATE`: set to `TRUE` if you have upgraded your application and need to migrate your data.
4. `process.env.REDISHOST`: the host of the Redis server.
5. `process.env.TASKID`: the task id of your application, used for retrieving the latest proof generated by the zkWasm hub for your application.

You can refer to the [Restore the State](Quick Tutorial.md#restore-the-state) section in [Quick Tutorial](Quick Tutorial.md) for more details about using these environment variables. You can either add those variables in the `.env` file or set them directly when you run the application in the terminal.


#### Migrate
First, the initialization process checks if the `migrate` environment variable is set to `TRUE`. If it is, it will proceed to migrate the data using the lastest merkle root from the proxy contract (the main contract of the [zkWasm Protocol Contracts](https://github.com/DelphinusLab/zkWasm-protocol)).

```ts
if (migrate) {
    if (remote) {throw Error("Can't migrate in remote mode");} // Can't migrate in remote mode because you have updated your application which has a different image hash.
    merkle_root = await getMerkleArray(); // Get the merkle root from the proxy contract.
    console.log("Migrate: updated merkle root", merkle_root);
}
```

This will retrieve the lastest merkle root from the proxy contract via the `getMerkleArray` function:

```ts
export async function getMerkleArray(): Promise<BigUint64Array>{
  // Connect to the Proxy contract
  const proxy = new ethers.Contract(constants.proxyAddress, abiData.abi, provider);
  // Fetch the proxy information
  let proxyInfo = await proxy.getProxyInfo();
  // Extract the old Merkle root
  const oldRoot = proxyInfo.merkle_root;

  return convertToBigUint64Array(oldRoot);
}
```

In order to use the `getMerkleArray` function for migration, you need to set two more environment variables:

1. `process.env.SETTLEMENT_CONTRACT_ADDRESS`: the address of the proxy contract.
2. `process.env.RPC_PROVIDER`: the rpc provider of the blockchain that the proxy contract is deployed on.

#### Restore
After the migration check, the initialization process will check if the `remote` environment variable is set to `TRUE`. If it is, it will proceed to restore the data using the lastest merkle root from the latest proof generated by the zkWasm hub for your application. 

```ts
if (remote) {
    while (true) { // Keep waiting until there are no uncompleted tasks on the zkWasm hub.
        const hasTasks = await has_uncomplete_task(); // Check if there are any uncompleted tasks on the zkWasm hub.
        if (hasTasks) 
            {
                console.log("remote = 1, There are uncompleted tasks. Trying again in 5 second...");
                await new Promise(resolve => setTimeout(resolve, 5000)); // Sleep for 5 second
            } else {
                console.log("remote = 1, No incomplete tasks. Proceeding...");
                break; // Exit the loop if there are no incomplete tasks
            }
    }

    let task = await get_latest_proof(taskid); // Get the latest proof from the zkWasm hub.
    if (task) {
        const instances = ZkWasmUtil.bytesToBN(task?.instances);
        merkle_root = new BigUint64Array([
            BigInt(instances[4].toString()),
            BigInt(instances[5].toString()),
            BigInt(instances[6].toString()),
            BigInt(instances[7].toString()),
        ]);
    }
}
```

After the restore check, we could initialize the application with the merkle root, which will be used to restore or initialize the state of the application.

```ts
application.initialize(merkle_root);
```

### Sequencer (ts/service.ts)

After the initialization process, the server will start the sequencer to process the transactions. To ensure the state consistency and clean start of the server, the sequencer queue will be drained to prevent any previous tasks from being processed which may cause unexpected state.

```ts
const myQueue = new Queue('sequencer', {connection});
const waitingCount = await myQueue.getWaitingCount();
console.log("waiting Count is:", waitingCount, " perform draining ...");
await myQueue.drain();
this.queue = myQueue;
```

!!!info 
    ***What is a Sequencer?***
    
    A sequencer is a component that manages the order of transactions in a blockchain or distributed system. It ensures that transactions are processed in a sequential manner, maintaining the integrity and consistency of the system's state.

#### Timetick transaction

The sequencer handles the timetick function for the application. Timetick is a special type of system transaction that is automatically generated at regular intervals to drive time-dependent state transitions in the application. It can be used in time-related applications such as Game, DeFi (for interest calculations), or any application that requires periodic state updates.

```ts
// Check if application needs automatic ticking
if (application.autotick()) {
    // Generate timetick transaction every 5 seconds
    setInterval(async () => {
        try {
            await myQueue.add('autoJob', {command:0});
        } catch (error) {
            console.error('Error adding automatic job to the queue:', error);
            process.exit(1);
        }
    }, 5000); // This can be set to any interval you want, the default is 5 seconds.
}
```
!!!info
    By default, the timetick interval is set to 5 seconds, you can change it here at the moment, we will provider better setting for the timetick interval in the future.

#### Transaction Processing Worker Implementation
The sequencer worker serves as the core transaction processing engine in our zkWasm Mini Rollup system. It operates on a Redis-backed queue system using BullMQ (which you have seen in the [Sequencer](#sequencer) main section), processing two distinct types of transactions:

   1. automatic timetick transactions, specified by `autoJob` (which we have mentioned in the [Timetick transaction](#timetick-transaction) section)
   2. user-submitted transactions, specified by `transaction`

##### Automatic Timetick Transaction Flow


```ts
if (job.name == 'autoJob') {
    try {
        let rand = await generateRandomSeed(); // Returns a seed commitment
        let oldSeed = application.randSeed(); // Returns the previous seed commitment
        let seed = 0n;
        if (oldSeed != 0n) { // If there is a previous seed commitment
            const randRecord = await modelRand.find({
                commitment: oldSeed.toString(),
            });
            seed = randRecord[0].seed!.readBigInt64LE(); // Retrieve the previous seed from the database
        };
        let signature = sign(new BigUint64Array([0n, seed, rand, 0n]), get_server_admin_key()); // Create an admin-signed transaction combining the seed and the generated commitment
        let u64array = signature_to_u64array(signature);
        application.handle_tx(u64array); // Execute the transaction in local
        await this.install_transactions(signature, job.id); //Install transaction into rollup
    } catch (error) {
        console.log("fatal: handling auto tick error, process will terminate.", error);
        process.exit(1);
    }
}
```

When processing an autoJob, the system follows these steps:

1. Generates a new random seed and returns its commitment.
2. If there is a previous seed commitment, retrieves the previous seed.
3. Creates an admin-signed transaction combining the seed and the generated commitment.
4. Executes the transaction and updates the system state

##### User Transaction Flow
```ts
if (job.name == 'transaction') {
    console.log("handle transaction ...");
    try {
        let signature = job.data.value; // Get the transaction signature
        let u64array = signature_to_u64array(signature);
        console.log("tx data", signature);
        application.verify_tx_signature(u64array); // Verify the transaction signature
        let error = application.handle_tx(u64array); // Execute the transaction
        if (error == 0) {
            // make sure install transaction will succeed
            await this.install_transactions(signature, job.id);
            try {
                const jobRecord = new modelJob({
                    jobId: signature.sigx,
                    message: signature.message,
                    result: "succeed",
                });
                await jobRecord.save();
            } catch (e) {
                console.log("Error: store transaction job error");
                throw e
            }
        } else {
            let errorMsg = application.decode_error(error); // Decode the transaction error
            throw Error(errorMsg)
        }
        console.log("done");
        const pkx = new LeHexBN(job.data.value.pkx).toU64Array();
        let jstr = application.get_state(pkx); // Get the updated player state
        let player = JSON.parse(jstr); // Parse the state into a JSON object
        let result = {
            player: player,
            state: snapshot
        }; // Return the updated player state and system snapshot
        return result
    } catch (e) {
        throw e
    }
}
```

For user-submitted transactions, the process includes:

1. Signature verification using application-specific logic
2. Transaction execution with error checking
3. State updates and job record maintenance
4. Return of updated player state and system snapshot

The system maintains transaction atomicity through careful error handling and database transaction management.

#### Transaction Installation (Rollup)
After verifying the signature, the sequencer will install the transaction into the rollup. Let's delve into the `install_transactions` function.

The function begins by adding the transaction to the witness array and triggering necessary callbacks:

```ts
transactions_witness.push(tx);
this.txCallback(tx);
snapshot = JSON.parse(application.snapshot());
```
This process captures the current state and prepares for persistence. The transaction is then stored in the database using a dedicated transaction model:
```
const txRecord = new modelTx(tx);
txRecord.save();
```
When the rollup reaches its preemption point, indicating a batch is ready for processing, the system initiates the proof generation process:

```ts
if (application.preempt()) {
    console.log("rollup reach its preemption point, generating proof:");
    let txdata = application.finalize(); // This will perform flush_settlement and return the transaction data
    ...
}
```

In deployment mode, the system submits the proof task and creates a bundle record linking the Merkle root with the task ID:

```ts
let task_id = await submitProofWithRetry(merkle_root, transactions_witness, txdata);
const bundleRecord = new modelBundle({
    merkleRoot: merkleRootToBeHexString(merkle_root),
    taskId: task_id,
});
```

After successful task submission, the system performs a complete state reset through the updated merkle root:

```ts
transactions_witness = new Array();
merkle_root = application.query_root();
await (initApplication as any)(bootstrap);
application.initialize(merkle_root);
```

### Serve Endpoints (ts/service.ts)

Once the main WASM application is initialized, we start the minirollup PRC server using nodejs express. It contains four endpoints.

1. query: This endpoint allows user to query their current state and the game public state:
```ts
app.post('/query', async (req, res) => {
    const value = req.body;
    if (!value) {
        return res.status(400).send('Value is required');
    }
    try {
        const pkx = new LeHexBN(value.pkx).toU64Array();
        let u64array = new BigUint64Array(4);
        u64array.set(pkx);
        let jstr = application.get_state(pkx);   // here we use the get_state function from application wasm binary
        let player = JSON.parse(jstr);
        let result = {
            player: player,
            state: snapshot
        };
        res.status(201).send({
            success: true,
            data: JSON.stringify(result),
        });
    }
    catch (error) {
        res.status(500).send('Get Status Error');
    }
});
```

2. send: An endpoint handles user transactions.
```ts
app.post('/send', async (req, res) => {
    const value = req.body;
    if (!value) {
        return res.status(400).send('Value is required');
    }
    try {
        const msg = new LeHexBN(value.msg);
        const pkx = new LeHexBN(value.pkx);
        const pky = new LeHexBN(value.pky);
        const sigx = new LeHexBN(value.sigx);
        const sigy = new LeHexBN(value.sigy);
        const sigr = new LeHexBN(value.sigr);
        if (verify_sign(msg, pkx, pky, sigx, sigy, sigr) == false) {
            res.status(500).send('Invalid signature');
        }
        else {
            const job = await myQueue.add('transaction', { value });
            res.status(201).send({
                success: true,
                jobid: job.id
            });
        }
    }
    catch (error) {
        res.status(500).send('Failed to add job to the queue');
    }
});

```
This endpoint will add transactions into the global job sequencer where each job is handled via the exposed wasm function `handle_tx`.


3. config: An endpoint that returns all static configuration of the application.
```ts
app.post('/config', async (req, res) => {
    try {
        let jstr = application.get_config();
        res.status(201).send({
            success: true,
            data: jstr
        });
    }
    catch (error) {
            res.status(500).send('Get Status Error');
        }
    });
```

4. jobId: An endpoint allows user to query the job information by jobId.
```ts
app.get('/job/:id', async (req, res) => {
    try {
        let jobId = req.params.id;
        const job = await Job.fromId(this.queue!, jobId);
        return res.status(201).json(job);
    } catch (err) {
        // job not tracked
        console.log(err);
        res.status(500).json({ message: (err as Error).toString()});
    }
});
```

Those endpoints are wrapped into RPC methods in [ts/src/rpc.ts](https://github.com/DelphinusLab/zkwasm-mini-rollup/blob/main/ts/src/rpc.ts) which can be used by any client to interact with the zkWasm Mini Rollup Application and you can find usage examples in [Quick Tutorial](Quick Tutorial.md#player-class).

### Server Admin Operations (ts/init_admin.js)

When changing the state of the rollup, some operations are only allowed for the server admin. For example, only the admin can perform timetick or deposit functions.

In your root directory, you can add the .env file which can specify the admin private key as:

```env
SERVER_ADMIN_KEY = YOUR_ADMIN_PRIVATE_KEY
```

In the Makefile of [helloworld rollup](https://github.com/DelphinusLab/zkwasm-mini-rollup/blob/main/helloworld/Makefile), you can find the following commands:

```bash
./src/admin.pubkey: ./ts/node_modules/zkwasm-ts-server/src/init_admin.js
	node ./ts/node_modules/zkwasm-ts-server/src/init_admin.js ./src/admin.pubkey
```

This command will generate the admin.pubkey file by using the `init_admin.js` script, which is used to initialize the admin public key drived from the admin private key. 

Then, in your config.rs of the application, you can specify the admin public key as:

```rust
// In src/config.rs
lazy_static::lazy_static! {
    pub static ref ADMIN_PUBKEY: [u64; 4] = {
        let bytes = include_bytes!("./admin.prikey");
        // Interpret the bytes as an array of u64
        let u64s = unsafe { std::slice::from_raw_parts(bytes.as_ptr() as *const u64, 4) };
        u64s.try_into().unwrap()
    };
}
```

When you need it, you can import the admin public key as:

```rust
use crate::config::ADMIN_PUBKEY;
```

When handling the user command including admin operatuons, you can use it like:
```rs
  COMMAND => {
    unsafe { require(*pkey == *ADMIN_PUBKEY) }; // check if the command is from the admin
    ... // handle admin operations
  }
```

You can find more concrete examples in the state implementation of [automata game](https://github.com/riddles-are-us/zkwasm-automata/blob/main/src/state.rs).


### Rollup Settlement Monitor (ts/settle.ts)
The settlement module handles the crucial process of finalizing transactions on the blockchain. It verifies proofs and processes withdrawals through a series of carefully orchestrated steps.

#### Environment Setup
In order to run the settle.ts script, you need to firstly set enviroment variables in .env in the root directory of your application:

```env
SETTLER_PRIVATE_ACCOUNT = YOUR_PRIVATE_ACCOUNT_PRIVATE_KEY
SETTLEMENT_CONTRACT_ADDRESS = YOUR_SETTLEMENT_CONTRACT_ADDRESS
IMAGE = YOUR_IMAGE_HASH
```

In settle.ts:

```ts
const signer = new ethers.Wallet(get_settle_private_account(), provider);
const constants = {
    proxyAddress: get_contract_addr(),
};
```

#### Settlement Process 

Here's the main settlement function that:

1. Retrieves the current Merkle root
2. Fetches associated proof data
3. Verifies and submits the proof to the blockchain
4. Checks if the transaction is successful and updates the record

```ts
async function trySettle() {
    let merkleRoot = await getMerkle(); // get the merkle root from the proxy contract
    let record = await modelBundle.findOne({ merkleRoot: merkleRoot});
    if (record) {
        // Fetch and verify proof
        let data0 = await getTaskWithTimeout(taskId, 60000);

        // get the proof data with verification instances
        let shadowInstances = data0.shadow_instances; 
        let batchInstances = data0.batch_instances;
        let proofArr = new U8ArrayUtil(data0.proof).toNumber();
        let auxArr = new U8ArrayUtil(data0.aux).toNumber();
        let verifyInstancesArr =  shadowInstances.length === 0
        ? new U8ArrayUtil(batchInstances).toNumber()
        : new U8ArrayUtil(shadowInstances).toNumber();
        let instArr = new U8ArrayUtil(data0.instances).toNumber();
        let txData = new Uint8Array(data0.input_context);

        // Process proof data
        const proxy = new ethers.Contract(constants.proxyAddress, abiData.abi, signer);
        // Submit verification transaction
        const tx = await proxy.verify(txData, proofArr, verifyInstancesArr, auxArr, [instArr]);
        const receipt = await tx.wait();
        .... // check the receipt
        //update record
        record.settleTxHash = tx.hash;
        record.settleStatus = status;
        await record.save();
    }
}
```

You can run the settle.ts script by:

```bash
node ts/settle.ts
```

And this will continueously check the proof and settle the transactions every 60 seconds:

```ts
// start monitoring and settle
async function main() {
    while (true) {
        try {
            await trySettle();
        } catch (error) {
            console.error("Error during trySettle:", error);
        }
     await new Promise(resolve => setTimeout(resolve, 60000));
 }
}
```

### Proof Task Debugging (ts/reproduce.ts)
This module provides functionality to debug and reproduce zkWASM proof tasks by recreating their execution environment. It's particularly useful for investigating failed proof generations or verifying proof task behavior. 

!!!info
    To reproduce the proof task, you need to have the [zkWasm Project](https://github.com/DelphinusLab/zkWasm) Installed.

#### Task Retrieval

In order to reproduce the proof task, you need to retrieve the task information from the service.

```ts
async function getTask(taskid: string) {
    const queryParams: QueryParams = {
        id: taskid,
        tasktype: "Prove",
        taskstatus: null,
        user_address: null,
        md5: null,
        total: 1,
    };
    return (await ServiceHelper.loadTasks(queryParams)).data[0];
}
```

This code can fetch detailed task information from the service, including:

- Public and private inputs
- Task status and metadata

#### Replay the Task

When performing the proof task, you can use the following code to replay the task:

```ts
async function replay(taskId: string) {
  const data0 = await getTask(taskId);
  const public_inputs = data0.public_inputs;
  const private_inputs = data0.private_inputs;
  ... // handle the task
}
```

##### Environment Setup

```ts
fs.writeFileSync('run-image.sh', `CLI=$HOME/zkWasm/target/release/zkwasm-cli\n`);
fs.appendFileSync('run-image.sh', `PARAMS=$HOME/zkWasm/params\n`);
```
The above code: 

- `CLI`: Defines the path to the zkWASM CLI executable
- `PARAMS`: Specifies the parameter directory containing necessary proving parameters

##### File Configuration
```ts
fs.appendFileSync('run-image.sh', `IMAGE=image.wasm\n`);
fs.appendFileSync('run-image.sh', `OUTPUT=output\n`);
```

The above code:

- `IMAGE`: Specifies the WASM binary to be executed
- `OUTPUT`: Defines the directory for proof generation output

##### CLI Command Construction
```ts
fs.appendFileSync('run-image.sh', `$CLI --params $PARAMS test dry-run --wasm $IMAGE \\\n`, 'utf-8');
```

The above code constructs the base command with:

- `test dry-run`: Executes in test mode for debugging
- `--params`: Points to proving parameters
- `--wasm`: Specifies the WASM image to execute

##### Input Parameters
```ts
for (const p of public_inputs) {
    fs.appendFileSync('run-image.sh', `--public ${p} \\\n`, 'utf-8');
}
for (const p of private_inputs) {
    fs.appendFileSync('run-image.sh', `--private ${p} \\\n`, 'utf-8');
}
```
The above code appends the public and private inputs to the CLI command.

##### Output Configuration
```ts
fs.appendFileSync('run-image.sh', `--output $OUTPUT`, 'utf-8');
```
The above code specifies the output directory for proof generation.

The resulting script will look something like:
```bash
CLI=$HOME/zkWasm/target/release/zkwasm-cli
PARAMS=$HOME/zkWasm/params
IMAGE=image.wasm
OUTPUT=output
$CLI --params $PARAMS test dry-run --wasm $IMAGE \
--public <public_input_1> \
--public <public_input_2> \
--private <private_input_1> \
--private <private_input_2> \
--output $OUTPUT
```