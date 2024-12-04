# Quick Tutorial

## Guideline 
This tutorial will guide you through the process of creating a simple zkWasm application in minutes. Please make sure you have already setup the environment by following the [Setup Environment](Setup Environment.md) guide.

## Step 1: Install the zkWasm Mini-Rollup service
The zkWasm Mini-Rollup service is a RESTful service that provides the zkWasm runtime environment. It provides the following functionalities:

- Serve the zkWasm runtime environment
- Provide the zkWasm REST ABI
- Maintain the zkWasm state through merkle tree enabled database and Redis
- Generate the witness from the merkle tree database for zkWasm verification
- Calculate the new merkle tree root when receiving the zkWasm transaction batch for settlement

### 1. Get the zkWasm Mini-Rollup service
You can get the zkWasm Mini-Rollup service by cloning the repository:
```bash
git clone https://github.com/DelphinusLab/zkwasm-mini-rollup.git
cd zkwasm-mini-rollup
```
Or, download the zip file from the [Github page](https://github.com/DelphinusLab/zkwasm-mini-rollup) and unzip it.

### 2. Start the zkWasm Mini-Rollup service
In the root directory of the zkWasm Mini-Rollup service, run:
```bash
docker-compose up
```
Make sure your running environment has the permission to access the Docker daemon. This command will start a Docker container named `zkwasm-mini-rollup`.

!!! info "Note"
    One zkWasm Mini-Rollup service must correspond to one zkWasm rollup application. This will be improved in the future by supporting multiple rollup applications in one service.


## Step 2: Get the Template Project

Clone the template project - The Hello World Rollup:
```bash
git clone https://github.com/riddles-are-us/helloworld-rollup.git
cd helloworld-rollup
```

!!! tip "Project Structure"
    The template includes:
    ```
    helloworld-rollup/
    ├── src/           # Rust source code - for application logic
    ├── ts/            # TypeScript code - for API testing
    ├── Cargo.toml     # Rust dependencies
    ├── Makefile       # Build scripts
    └── rust-toolchain # Rust version specification
    ```

Install the dependencies for ts:
```bash
cd ts
npm install
```

After installing the dependencies, compile the ts code:
```bash
npx tsc
```
This will generate the js code in the `ts/` directory and facilitate the backend server running and testing.

Build the project using make:
```bash
cd ..   #Move to the root directory of the project
make build
```

## Step 3: Run the Rollup Application

In the root directory of the project, run:
```bash
make run
```

This will start the backend server by running the ```node ./ts/src/service.js```.

You shall see some output like the following:
```
rpc bind merkle server: http://127.0.0.1:3030
initialize mongoose ...
start express server
Server is running on http://0.0.0.0:3000
connecting redis server: localhost
bootstrapping ... (deploymode: false, remote: false, migrate: false)
loading wasm application ...
check merkle database connection ...
initialize sequener queue ...
waiting Count is: 0  perform draining ...
initialize application merkle db ...
```

Congratulations! You have successfully started the zkWasm rollup application. However, in order to build your own rollup application, you need to understand the core components of the zkWasm rollup application.

## Step 4: The Code Overview

Let's examine the core components of our zkWasm application. This hello world rollup application is structured into several key Rust files, each handling specific functionality. 

### Server Side Code (Backend Code)

#### Main Entry Point (`src/lib.rs`)

```rust
use wasm_bindgen::prelude::*;
use zkwasm_rest_abi::*;
pub mod config;
pub mod state;
pub mod settlement;

use crate::config::Config;
use crate::state::{State, Transaction};
zkwasm_rest_abi::create_zkwasm_apis!(Transaction, State, Config);
```

The above code includes the following key components:

- `wasm_bindgen`: Enables Rust-JavaScript interoperability
- `zkwasm_rest_abi`: Provides core zkWasm functionality
- `create_zkwasm_apis!`: Macro that generates necessary API endpoints

#### Configuration (`src/config.rs`)

```rust
use serde::Serialize;

#[derive(Serialize, Clone)]
pub struct Config {
    version: &'static str,
}

lazy_static::lazy_static! {
    pub static ref CONFIG: Config = Config {
        version: "1.0"
    };
}

impl Config {
    pub fn to_json_string() -> String {
        serde_json::to_string(&CONFIG.clone()).unwrap()
    }

    pub fn autotick() -> bool {
        true
    }
}
```

The `Config` struct:

- Defines application configuration
- Provides JSON serialization for config values
- Controls auto-tick behavior - the system will automatically advance its state through tick events, facilitating time-based state transitions in the zkWasm runtime. 

!!! tip "Note"
    Currently, the time interval is set to 5 seconds in the server side, which you can modify in the `service.ts` file in the `/src` directory of the `zkwasm-ts-server` package.

#### Settlement Management (`src/settlement.rs`)

```rust
use zkwasm_rest_abi::WithdrawInfo;

pub struct SettlementInfo(Vec<WithdrawInfo>);
pub static mut SETTLEMENT: SettlementInfo = SettlementInfo(vec![]);

impl SettlementInfo {
    pub fn append_settlement(info: WithdrawInfo) {
        unsafe { SETTLEMENT.0.push(info) };
    }

    pub fn flush_settlement() -> Vec<u8> {
        zkwasm_rust_sdk::dbg!("flush settlement\n");
        let sinfo = unsafe { &mut SETTLEMENT };
        let mut bytes: Vec<u8> = Vec::with_capacity(sinfo.0.len() * 32);
        for s in &sinfo.0 {
            bytes.extend_from_slice(&s.feature.to_le_bytes());
            bytes.extend_from_slice(&s.address);
            bytes.extend_from_slice(&s.amount.to_le_bytes());
        }
        sinfo.0 = vec![];
        bytes
    }
}
```

The `SettlementInfo` struct:

- Handles withdrawal information
- Converts settlement data to bytes for processing
- Implements flush mechanism for batch processing

!!! note "Note"
    In the current architecture of zkWasm Rollup Application, the withdrawal requests from users are handled in the server side. When settlement is triggered, the server will collect all the withdrawals and send them with merkle tree root to the zkWasm protocol contract for verification. 

#### State Management (`src/state.rs`)

##### 1. Player Data Structure
```rust
#[derive(Debug, Serialize)]
pub struct PlayerData {
    pub counter: u64,
}

impl Default for PlayerData {
    fn default() -> Self {
        Self { counter: 0 }
    }
}

impl StorageData for PlayerData {
    fn from_data(u64data: &mut IterMut<u64>) -> Self {
        let counter = *u64data.next().unwrap();
        PlayerData { counter }
    }
    fn to_data(&self, data: &mut Vec<u64>) {
        data.push(self.counter);
    }
}

pub type HelloWorldPlayer = Player<PlayerData>;
```

The `PlayerData` struct:

- Defines player-specific data structure with a ```counter``` field
- Implements `Default` trait for initializing new players with counter set to 0
- Implements `StorageData` trait for data serialization and deserialization
- Creates a type alias `HelloWorldPlayer` for Player with PlayerData

##### 2. State Structure
```rust
#[derive(Serialize)]
pub struct State {
    counter: u64
}

impl State {
    pub fn get_state(pkey: Vec<u64>) -> String {
        let player = HelloWorldPlayer::get_from_pid(&HelloWorldPlayer::pkey_to_pid(&pkey.try_into().unwrap()));
        serde_json::to_string(&player).unwrap()
    }

    pub fn rand_seed() -> u64 {
        0
    }

    pub fn store(&self) {
    }

    pub fn initialize() {
    }

    pub fn new() -> Self {
        State {
            counter: 0,
        }
    }

    pub fn snapshot() -> String {
        let state = unsafe { &STATE };
        serde_json::to_string(&state).unwrap()
    }

    pub fn preempt() -> bool {
        let state = unsafe { &STATE };
        return state.counter % 20 == 0; 
    }

    pub fn flush_settlement() -> Vec<u8> {
        let data = SettlementInfo::flush_settlement();
        unsafe { STATE.store() };
        data
    }

    pub fn tick(&mut self) {
        self.counter += 1;
    }
}
```

The `State` struct:

- Maintains global state with a counter field - The counter can be used to track the number of transactions processed
- Provides methods for state manipulation and querying, such as ```get_state```, ```store```
- Implements serialization for state snapshots
- Handles settlement flushing and state updates, such as ```flush_settlement```

We can also notice that there is a global variable ```STATE``` with field ```counter``` in the code. This is the state of the zkWasm rollup application, which shall be distinguished from the ```counter``` field in the ```PlayerData``` struct:
```rust
pub static mut STATE: State = State {
    counter: 0
};
```

#### 3. Transaction Handler
```rust
pub struct Transaction {
    pub command: u64,
    pub data: Vec<u64>,
}

const AUTOTICK: u64 = 0;
const INSTALL_PLAYER: u64 = 1;
const INC_COUNTER: u64 = 2;

const ERROR_PLAYER_ALREADY_EXIST: u32 = 1;
const ERROR_PLAYER_NOT_EXIST: u32 = 2;

impl Transaction {
    pub fn decode_error(e: u32) -> &'static str {
        match e {
            ERROR_PLAYER_NOT_EXIST => "PlayerNotExist",
            ERROR_PLAYER_ALREADY_EXIST => "PlayerAlreadyExist",
            _ => "Unknown"
        }
    }

    pub fn decode(params: [u64; 4]) -> Self {
        let command = params[0] & 0xff;
        let data = vec![params[1], params[2], params[3]]; // pkey[0], pkey[1], amount
        Transaction {
            command,
            data,
        }
    }

    pub fn install_player(&self, pkey: &[u64; 4]) -> u32 {
        zkwasm_rust_sdk::dbg!("install \n");
        let pid = HelloWorldPlayer::pkey_to_pid(pkey);
        let player = HelloWorldPlayer::get_from_pid(&pid);
        match player {
            Some(_) => ERROR_PLAYER_ALREADY_EXIST,
            None => {
                let player = HelloWorldPlayer::new_from_pid(pid);
                player.store();
                0
            }
        }
    }

    pub fn inc_counter(&self, _pkey: &[u64; 4]) -> u32 {
        todo!()
    }

    pub fn process(&self, pkey: &[u64; 4], _rand: &[u64; 4]) -> u32 {
        match self.command {
            AUTOTICK => {
                unsafe { STATE.tick() };
                return 0;
            },
            INSTALL_PLAYER => self.install_player(pkey),
            INC_COUNTER => self.inc_counter(pkey),
            _ => {
                return 0
            }
        }
    }
}
```

The `Transaction` struct:

- Defines transaction structure and command types
- Handles player installation and counter increment operations (todo)
- Implements error handling with specific error codes
- Provides transaction decoding and processing functionality
- Uses pattern matching for command routing

You may notice that the ```process``` method in the `Transaction` struct is the core method that handles the transaction processing:

```rust
pub fn process(&self, pkey: &[u64; 4], _rand: &[u64; 4]) -> u32 {
    match self.command {
        AUTOTICK => {
            unsafe { STATE.tick() };
            return 0;
        },
        INSTALL_PLAYER => self.install_player(pkey),
        INC_COUNTER => self.inc_counter(pkey),
        _ => {
            return 0
        }
    }
}
```
It receives the transaction command and then routes the transaction to the corresponding handler based on the command. In this case, we have three commands:

- ```AUTOTICK```: Automatically tick the state of the rollup application, which increments the ```counter``` field in the ```State``` struct by 1
- ```INSTALL_PLAYER```: Install a new player, which creates a new player with a unique ```pid``` and initializes its ```PlayerData```
- ```INC_COUNTER```: Increment the counter of a player, which increments the ```counter``` field in the ```PlayerData``` struct by 1

This process is the core logic of the zkWasm rollup application, which you can refer to implement your own application logic.

### Client Side Code (Frontend Code)

The client-side code is written in TypeScript and provides a convenient interface for interacting with the hello world zkWasm rollup application. Let's examine the key components:

#### Constants and Helper Functions

```typescript
const CMD_INSTALL_PLAYER = 1n;
const CMD_INC_COUNTER = 2n;

function createCommand(nonce: bigint, command: bigint, feature: bigint) {
    return (nonce << 16n) + (feature << 8n) + command;
}
```

- Two command constants are defined for player installation and counter incrementing
- `createCommand` helper function constructs command values by combining:
    - `nonce`: Transaction sequence number
    - `command`: Operation type (install or increment)
    - `feature`: Additional features (currently unused)

You can customize the `createCommand` function to pack different types of data based on your application's needs. Here are some examples of how you might modify the bit layout:

1. **Game Commands**:
```typescript
// 32 bits nonce + 8 bits gameType + 8 bits playerId + 16 bits command
function createGameCommand(nonce: bigint, gameType: bigint, playerId: bigint, command: bigint) {
    return (nonce << 32n) + (gameType << 24n) + (playerId << 16n) + command;
}
```

2. **Transaction Commands**:
```typescript
// 32 bits nonce + 16 bits amount + 8 bits tokenId + 8 bits command
function createTxCommand(nonce: bigint, amount: bigint, tokenId: bigint, command: bigint) {
    return (nonce << 32n) + (amount << 16n) + (tokenId << 8n) + command;
}
```

3. **NFT Commands**:
```typescript
// 16 bits nonce + 32 bits tokenId + 8 bits collection + 8 bits command
function createNFTCommand(nonce: bigint, tokenId: bigint, collection: bigint, command: bigint) {
    return (nonce << 48n) + (tokenId << 16n) + (collection << 8n) + command;
}
```

When designing your command structure, consider:

- The size needed for each field
- Priority and access frequency of fields
- Future extensibility requirements

Remember to provide corresponding extraction functions for unpacking the data when needed.

#### Player Class

The `Player` class serves as the main interface for interacting with the rollup:

```typescript
export class Player {
    processingKey: string;
    rpc: ZKWasmAppRpc;

    constructor(key: string, rpc: string) {
        this.processingKey = key
        this.rpc = new ZKWasmAppRpc(rpc);
    }
    // ...
}
```

Key methods in the Player class:

##### 1. State Query
```typescript
async getState(): Promise<any> {
    let state:any = await this.rpc.queryState(this.processingKey);
    return JSON.parse(state.data);
}
```
The ```getState``` method:

- Retrieves the current state for a player
- Returns parsed JSON data containing player information

##### 2. Nonce Management
```typescript
async getNonce(): Promise<bigint> {
    let state:any = await this.rpc.queryState(this.processingKey);
    let nonce = 0n;
    if (state.data) {
        let data = JSON.parse(state.data);
        if (data.player) {
            nonce = BigInt(data.player.nonce);
        }
    }
    return nonce;
}
```

The ```getNonce``` method:

- Retrieves the current nonce (transaction sequence number) for a player
- Essential for transaction ordering and replay protection

##### 3. Player Registration
```typescript
async register() {
    let nonce = await this.getNonce();
    try {
        let result = await this.rpc.sendTransaction(
            new BigUint64Array([createCommand(nonce, CMD_INSTALL_PLAYER, 0n), 0n, 0n, 0n]),
            this.processingKey
        );
        return result
    } catch(e) {
        if (e instanceof Error) {
            console.log(e.message);
        }
    }
}
```

The ```register``` method:

- Registers a new player in the system
- Creates and sends an installation transaction
- Handles potential errors during registration

##### 4. Counter Increment
```typescript
async incCounter() {
    let nonce = await this.getNonce();
    try {
        let result = await this.rpc.sendTransaction(
            new BigUint64Array([createCommand(nonce, CMD_INC_COUNTER, 0n), 0n, 0n, 0n]), 
            this.processingKey
        );
        return result;
    } catch(e) {
        if (e instanceof Error) {
            console.log(e.message);
        }
    }
}
```

The ```incCounter``` method:

- Increments the player's counter
- Creates and sends an increment transaction
- Handles potential errors during the operation

#### Usage Example

Here's how you might use the client-side API to interact or test with the hello world zkWasm rollup backend, this is also the way to integrate the API into your frontend application:

```typescript
// Initialize a player
const player = new Player("processingKey", "http://localhost:3000");

// Register the player
await player.register();

// Get player state
const state = await player.getState();
console.log("Player state:", state);

// Increment counter
await player.incCounter();
```
!!! note "Note"
    You may notice that the "processingKey" is actually the key for accessing the zkWasm rollup application, it is required and used to sign the data in every transaction to the zkWasm rollup application. In real implementation, you need to generate a processingKey from the user's signature, which is derived from a unique message signed by the user's private key. Please remind your user to keep this key secure and never expose it to the public, as well as never signing a same message with the same private key.

## Step 5: Implementing your own Rollup Application

Now that you have a basic understanding of the zkWasm rollup application, you can start to implement your own application by referring to the hello world rollup application.

Let's first complete the hello world rollup application by implementing the `inc_counter` method in `src/state.rs`. This method increments the `counter` field in the `PlayerData` struct by 1. You can use this pattern to implement similar state changes in your own rollup application.

```rust
pub fn inc_counter(&self, _pkey: &[u64; 4]) -> u32 {
    // Convert player's public key to player ID
    let pid = HelloWorldPlayer::pkey_to_pid(_pkey);
    // Try to get the player instance using the ID
    let player = HelloWorldPlayer::get_from_pid(&pid);
    
    // Match on the optional player result
    match player {
        // If player exists
        Some(mut p) => {
            // Increment the player's counter
            p.data.counter += 1;
            // Store the updated state
            p.store();
            // Return 0 to indicate success
            0
        },
        // If player doesn't exist, return error
        None => ERROR_PLAYER_NOT_EXIST
    }
}
```

Let's break down the key components of this implementation:

#### 1. Player Identification
When you want to access or modify the state of a player, you need to identify the player first. In the hello world rollup application, the player is identified by the player's ID, which is a unique identifier derived from the player's public key.

- `HelloWorldPlayer::pkey_to_pid(_pkey)`: Converts the public key to a player ID
- `HelloWorldPlayer::get_from_pid(&pid)`: Retrieves the player instance using the ID

#### 2. State Management
Remember that player may not exist, so you need to check if the player exists before accessing or modifying its state.

- Uses pattern matching (`match`) to handle both existing and non-existing player cases
- For existing players:
    - Increments the counter: `p.data.counter += 1`
    - Persists the change: `p.store()`
    - Returns 0 to indicate success
- For non-existing players:
    - Returns `ERROR_PLAYER_NOT_EXIST`

#### 3. Error Handling
- Returns appropriate error codes based on the operation result
- Uses the previously defined `ERROR_PLAYER_NOT_EXIST` constant

This implementation demonstrates several important patterns for building your own rollup application:

1. **State Access**: How to access and modify player-specific state
2. **Error Handling**: How to handle various edge cases and error conditions
3. **State Persistence**: How to properly store updated state
4. **Player Management**: How to handle player existence checks

When implementing your own rollup application, you can follow similar patterns to:

- Define your own state structures
- Implement state modification methods
- Handle errors appropriately
- Ensure proper state persistence

### Modifying the Global State

The global state of the rollup application is maintained in the ```STATE``` variable, which is a global variable. When you want to modify the global state, you need to update the ```STATE``` variable. For example, in process method in the ```Transaction``` struct, we have the following code:

```rust
match self.command {
    AUTOTICK => {
        unsafe { STATE.tick() };
        return 0;
    },
    ...
}
```

This is the way to modify the global state of the rollup application, and the tick method is defined in the ```State``` struct as:

```rust
pub fn tick(&mut self) {
    self.counter += 1;
}
```

Remember that any state modifications for players should be:

- Atomic and consistent
- Properly persisted using the `store()` method
- Protected with appropriate existence checks
- Accompanied by proper error handling

However, for Global State, you don't need to consider the existence of players, and you can directly modify the ```STATE``` variable as it is defined as mutable.

By following these patterns, you can implement various types of state changes in your own rollup application while maintaining consistency and reliability.

## Step 6: Interacting with zkWasm Hub

zkWasm Hub is a hosted cloud service provided by DelphinusLab for finding and sharing zkWasm application images. Using zkWasm Hub, developers can access it using public rest services and create their own private zkWasm space. zkWasm Hub provides automated proving and batching service for applications' workloads with customizable WASM extensions (via WASM host application interfaces). Moreover, users can distribute their GitHub applications onto zkWasm Hub by its auto compilation and updating service. Overall, it provides:

1. Application image deployment and setup
2. Batching and generating zkWasm proofs for applications

zkWasm Hub operates through a permissionless proving node pool, allowing anyone to participate and provide proving services for applications using the zkWasm cloud service.

!!! note "Note"
    This section aims to provide a easy hands-on way to interact with zkWasm Hub. For more details, please refer to:

    - [zkWasm Service CLI](https://github.com/DelphinusLab/zkWasm-service-cli)
    - [zkWasm Service Helper](https://github.com/DelphinusLab/zkWasm-service-helper)

### Submit your rollup application image to zkWasm Hub

Let's back to our hello world rollup application, and see how to interact with zkWasm Hub.

In the root directory of the hello world rollup application, if you haven't built the modified application, run:

```bash
make build
```

then, we go to the `ts` directory and run:

```bash
./publish.sh
```

or

```bash
sh publish.sh
```

And you may expect the following output:

```bash
Begin adding image for  .../helloworld-rollup/ts/node_modules/zkwasm-ts-server/src/application/application_bg.wasm
msg is: application_bg.(....)
Run success.
signature is: ...
get addNewWasmImage response: [object Object]
Add Image Response {
  md5: '...',
  id: '...'
}
Finish addNewWasmImage!
```

You shall record the `md5` value, which will be used when submitting the proof task to zkWasm Hub via your application server.

### Deploy your rollup application

Let's back to the root directory of the hello world rollup application, and run:

```bash
DEPLOY=TRUE IMAGE="YOUR_MD5_HASH" make run
```
This will tell the server to automatically submit proof tasks to zkWasm Hub when `preempt` method (defined in `src/state.rs`) is triggered. More details about the `preempt` method please refer to [zkWasm Rust SDK](https://jupiterxiaoxiaoyu.github.io/zkwasm-development-recipe/zkWasm%20Rust%20SDK.html#3-preemption-check).

Now we have deployed our rollup application onto zkWasm Hub, and it is ready to be used. You can find the tasks related to your rollup application in the zkWasm Hub Explorer by searching your application md5 hash:

- [zkWasm Hub Explorer](https://explorer.zkwasmhub.com/)












