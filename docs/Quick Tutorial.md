# Quick Tutorial

## Guideline 
This tutorial will guide you through the process of creating a simple zkWasm application in minutes. Please make sure you have already setup the environment by following the [Setup Environment](Setup Environment.md) guide.

## Step 1: Install the zkWasm Mini-Rollup service
The zkWasm Mini-Rollup service is a RESTful service that provides the zkWasm runtime environment. It provides the following functionalities:

- Serve the zkWasm runtime environment
- Provide the zkWasm REST API
- Maintain the zkWasm state through merkle tree enabled database and Redis
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
Make sure your running environment has the permission to access the Docker daemon.

!!! info "Note"
    One zkWasm Mini-Rollup service must correspond to one zkWasm rollup application. This will be improved in the future by supporting multiple rollup applications in one service.


## Step 2: Get the Template Project

Clone the template project - The Hello World Rollup:
```bash
git clone https://github.com/DelphinusLab/helloworld-rollup
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

Build the project using make:
```bash
cd ..   #Move to the root directory of the project
make build
```

## Step 3: Test the Rollup Application





## Step 4: The Code Overview

Let's examine the core components of our zkWasm application. The project is structured into several key Rust files, each handling specific functionality.

### Main Entry Point (`src/lib.rs`)

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

### Configuration (`src/config.rs`)

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
- Controls auto-tick behavior - the system will automatically advance its state through tick events, facilitating time-based state transitions in the zkWasm runtime

### Settlement Management (`src/settlement.rs`)

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

### State Management (`src/state.rs`)

#### 1. Player Data Structure
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

- Defines player-specific data structure with a counter field
- Implements `Default` trait for initializing new players with counter set to 0
- Implements `StorageData` trait for data serialization and deserialization
- Creates a type alias `HelloWorldPlayer` for Player with PlayerData

#### 2. State Structure
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
        return state.counter >= 20;
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
- Provides methods for state manipulation and querying
- Implements serialization for state snapshots
- Handles settlement flushing and state updates

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

## Step 5: Implementing your own Rollup Application

