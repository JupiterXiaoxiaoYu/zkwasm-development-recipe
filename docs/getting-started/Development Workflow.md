# Development Workflow

## Guideline

This workflow will guide you through the process of developing a zkWasm application. You can find reference chapters corresponding to each step in the workflow. The workflow follows a five-category cycle:

1. Learn: Learn the basics of zkWasm and the zkWasm Mini-Rollup service
2. Design: Design your application as a state machine
3. Develop: Develop your application in Rust
4. Test: Test your application with zkwasm-ts-server
5. Deploy: Deploy your application to the zkWasm Hub and blockchain

## Development Workflow

### Learn
1. Understand [Web Application Development](../Core%20Concepts#1-understanding-web-application-development) 
2. Understand [Blockchain Engineering](../Core%20Concepts#3-understanding-blockchain-engineering) and [Zero-Knowledge Proof](../Core%20Concepts#4-understanding-zero-knowledge-proofs)
3. Understand [zkWasm Basics](../zkWasm%20Overview.md#1-introduction-to-zkwasm) and [Architecture](../zkWasm%20Overview.md#6-architecture-of-a-zkwasm-rollup-application)

### Design
1. Design your application as a [State Machine](../Design%20Application%20as%20State%20Machine.md)
2. Choose your [Development Language and Frameworks](../development-guide/Web3%20Development%20Frameworks.md)

### Develop
1. Set up [Environment](Setup%20Environment.md)
2. Get started with [Quick Tutorial](Quick%20Tutorial.md)

#### Backend
3. Install [zkWasm-Mini-Rollup](Quick%20Tutorial.md#step-1-install-the-zkwasm-mini-rollup-service)
4. Develop your zkWasm application in Rust using [Rust SDK](../development-guide/zkWasm%20Rust%20SDK.md)
5. [Optional] Implement [Time-Driven Events](../development-guide/Implementing%20Time-Driven%20Events.md) and [Random Numbers](../development-guide/Generating%20Random%20Numbers.md) features

#### Frontend
6. Develop Frontend and User Interface using [Web3 Development Frameworks](../development-guide/Web3%20Development%20Frameworks.md)
7. [Implement RPC calls](./Quick%20Tutorial.md#client-side-code-frontend-code) in the frontend to interact with the zkWasm application

#### Smart Contract
8. [Optional] Develop custom smart contracts like Token or Custom Transaction Contract
9. [Optional] Develop custom [Settlement Monitor](../zkwasm-mini-rollup/Rollup%20Server.md#rollup-settlement-monitor-tssettlets) and [Deposit Monitor](../zkwasm-protocol/Deposit.md#deposit-monitor)

### Test
1. Test the zkWasm application internally with [State Validation](../Design%20Application%20as%20State%20Machine.md#2-state-testing)
2. Test the zkWasm application from the frontend with [zkwasm-ts-server](./Quick%20Tutorial.md#usage-example)
3. [Optional] Test the [state retrival by merkle tree root](./Quick%20Tutorial.md#restore-the-state)
4. Test your smart contracts with [deposit](../zkwasm-protocol/Deposit.md#deposit-monitor) and [settlement](../zkwasm-mini-rollup/Rollup%20Server.md#rollup-settlement-monitor-tssettlets) monitors

### Deploy
1. Deploy the zkWasm Mini Rollup application to the [zkWasm Hub](Quick%20Tutorial.md#step-6-interacting-with-zkwasm-hub)
2. Deploy the Frontend code
3. Deploy the zkWasm Protocol [smart contract](../zkwasm-protocol/zkWasm%20Protocol.md) to the target blockchain
4. Deploy the [Withdraw Transaction Contract](../zkwasm-protocol/Withdraw.md) to the target blockchain
5. Deploy other smart contracts like token or custom Transaction Contract
6. Deploy the [Settlement Monitor](../zkwasm-mini-rollup/Rollup%20Server.md#rollup-settlement-monitor-tssettlets)
7. Deploy the [Deposit Monitor](../zkwasm-protocol/Deposit.md#deposit-monitor)
