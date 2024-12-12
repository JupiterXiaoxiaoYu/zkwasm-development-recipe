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
