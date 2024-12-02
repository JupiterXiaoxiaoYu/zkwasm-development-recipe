# Setup Environment

## Prerequisites
- Basic knowledge of Rust programming language and Cargo.
- Basic knowledge of Makefile.
- Basic knowledge of Docker.
- Basic knowledge of TypeScript and Node.js.

## Node.js Setup

!!! note "Install Node.js and npm"
    To install Node.js and npm, follow these steps:

    1. Visit the [Node.js download page](https://nodejs.org/) and download the LTS version for your operating system.
    2. Run the installer and follow the setup instructions.
    3. Ensure that you check the option to install npm along with Node.js.

!!! tip "Verify Installation"
    After installation, verify Node.js, npm, and npx are properly installed:
    ```bash
    node --version
    npm --version
    npx --version    # npx comes with npm 5.2.0+
    ```

!!! info "About npx"
    - npx comes bundled with npm version 5.2.0 and higher
    - If npx command is not found, update npm:
    ```bash
    npm install -g npm@latest
    ```
    - Or install npx explicitly:
    ```bash
    npm install -g npx
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

## Rust Setup

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

## WebAssembly Tools Setup

### Install wasm-pack

!!! note "Install wasm-pack"
    wasm-pack is a tool for building Rust-generated WebAssembly packages.
    ```bash
    # Install wasm-pack using Cargo
    cargo install wasm-pack
    ```

!!! tip "Verify Installation"
    After installation, verify wasm-pack is properly installed:
    ```bash
    wasm-pack --version
    ```

### Install wasm-opt

!!! note "Install wasm-opt"
    You can install wasm-opt using Cargo:
    ```bash
    cargo install wasm-opt
    ```

    Alternatively, you can install it as part of the Binaryen toolkit:

    1. Download from [GitHub](https://github.com/WebAssembly/binaryen/releases)
    2. Extract and add to PATH:
    
    ```bash
    wget https://github.com/WebAssembly/binaryen/releases/download/version_109/binaryen-version_109-x86_64-linux.tar.gz
    tar -xzf binaryen-version_109-x86_64-linux.tar.gz
    export PATH=$PATH:$(pwd)/binaryen-version_109/bin
    ```

!!! tip "Verify Installation"
    After installation, verify wasm-opt is properly installed:
    ```bash
    wasm-opt --version
    ```

## Install Make

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

## Docker Setup

!!! note "Install Docker"
    Choose your operating system and follow the installation steps:

    1. For Ubuntu/Debian:
    ```bash
    # Remove old versions
    sudo apt-get remove docker docker-engine docker.io containerd runc

    # Install dependencies
    sudo apt-get update
    sudo apt-get install \
        ca-certificates \
        curl \
        gnupg \
        lsb-release

    # Add Docker's official GPG key
    sudo mkdir -m 0755 -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

    # Set up the repository
    echo \
        "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
        $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    # Install Docker Engine
    sudo apt-get update
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```

    2. For macOS:

        - Download [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop)
        - Double-click the downloaded `.dmg` file and drag Docker to Applications
        - Start Docker from Applications

    3. For Windows:

        - Enable WSL 2 feature
        - Download [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop)
        - Run the installer and follow the prompts

!!! tip "Verify Installation"
    After installation, verify Docker is properly installed:
    ```bash
    docker --version
    docker compose version
    ```

!!! warning "Post-installation Steps"
    To use Docker without sudo (Linux):
    ```bash
    # Create docker group
    sudo groupadd docker

    # Add your user to docker group
    sudo usermod -aG docker $USER

    # Apply new group membership (or log out and back in)
    newgrp docker
    ```

!!! example "Test Docker Installation"
    Run a test container:
    ```bash
    docker run hello-world
    ```
