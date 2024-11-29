# Quick Tutorial

## Guideline 
This tutorial will guide you through the process of creating a simple zkWasm application in minutes.

## Prerequisites
- Basic knowledge of Rust programming language and Cargo.
- Basic knowledge of Makefile.
- Basic knowledge of TypeScript and Node.js.

## Step 1: Setup Environment

### Node.js Setup

!!! note "Install Node.js and npm"
    To install Node.js and npm, follow these steps:

    1. Visit the [Node.js download page](https://nodejs.org/) and download the LTS version for your operating system.
    2. Run the installer and follow the setup instructions.
    3. Ensure that you check the option to install npm along with Node.js.

!!! tip "Verify Installation"
    After installation, verify Node.js and npm are properly installed:
    ```bash
    node --version
    npm --version
    ```

!!! warning "Note for Windows Users"
    If you encounter issues with permissions or paths, consider using [nvm-windows](https://github.com/coreybutler/nvm-windows) to manage Node.js versions.

!!! example "Using nvm for Node.js"
    If you prefer using a version manager, you can use nvm (Node Version Manager):

    - For macOS/Linux:
    ```bash
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
    source ~/.bashrc
    nvm install --lts
    ```

    - For Windows, use [nvm-windows](https://github.com/coreybutler/nvm-windows) and follow the installation instructions on the GitHub page.

!!! info "Package Manager Installation"
    If you need to install package managers first:

    - Homebrew (macOS): [Installation Guide](https://brew.sh/)
    - apt (Ubuntu/Debian): Pre-installed, or use:
     ```bash
     sudo apt update && sudo apt install apt
     ```
    - Chocolatey (Windows): [Installation Guide](https://chocolatey.org/install)

### Rust Setup

!!! note "Install Rust"
    If you haven't installed Rust, you can install it using rustup:
    ```bash
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
    ```

!!! tip "Verify Installation"
    After installation, verify Rust and Cargo are properly installed:
    ```bash
    rustc --version
    cargo --version
    ```

!!! warning "Note for Windows Users"
    If you're using Windows, we recommend:

    1. Using Windows Subsystem for Linux (WSL)
    2. Or installing Rust through the official installer from [rustup.rs](https://rustup.rs/)

!!! example "Optional: IDE Setup"
    We recommend using VS Code with the following extensions:

    - rust-analyzer
    - Even Better TOML
    - CodeLLDB

### Install Make

!!! note "Linux/macOS Users"
    For Unix-based systems, Make is usually pre-installed. If not:

    Ubuntu/Debian:
    ```bash
    sudo apt update
    sudo apt install build-essential
    ```

    macOS:
    ```bash
    xcode-select --install
    ```

!!! note "Windows Setup for Make"
    Windows users have several options:

    1. Using Chocolatey (Recommended):
    ```bash
    choco install make
    ```

    2. Using MSYS2:
    ```bash
    # First install MSYS2 from https://www.msys2.org/
    # Then open MSYS2 terminal and run:
    pacman -S make
    ```

    3. Using WSL (Best Option):
    ```bash
    # Install WSL first
    wsl --install

    # After WSL is installed, open Ubuntu terminal and run:
    sudo apt update
    sudo apt install build-essential
    ```

!!! tip "Verify Installation"
    After installation, verify Make is properly installed:
    ```bash
    make --version
    ```

!!! info "Package Manager Installation"
    If you need to install package managers first:

    - Chocolatey: [Installation Guide](https://chocolatey.org/install)
    - MSYS2: [Download Page](https://www.msys2.org/)
    - WSL: [Microsoft Guide](https://learn.microsoft.com/en-us/windows/wsl/install)

    After installing the package manager, make sure you correctly setup the environment variables for the package manager.

### Get the Template Project

!!! info "Clone Template"
    Clone the example project to get started:
    ```bash
    git clone https://github.com/riddles-are-us/helloworld-rollup
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

!!! info "Install Dependencies for ts"
    Install the dependencies for ts:
    ```bash
    cd ts
    npm install
    ```

!!! example "Build the Project"
    Build the project using make:
    ```bash
    cd ..   #Move to the root directory of the project
    make build
    ```

## Step 2: The Code Overview

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

!!! info "Key Components"
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

!!! info "Configuration Details"
    - Defines application configuration
    - Uses `lazy_static` for singleton pattern
    - Provides JSON serialization for config values
    - Controls auto-tick behavior

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

!!! info "Important Notes"
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

!!! info "Player Management"
    - Defines player-specific data structure, here we use `PlayerData` to store the player's counter.
    - Implements serialization and storage traits
    - Provides default values for new players

#### 2. State Structure
