# Setup Environment

## Prerequisites
- Basic knowledge of Rust programming language and Cargo.
- Basic knowledge of Makefile.
- Basic knowledge of TypeScript and Node.js.


## Node.js Setup

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