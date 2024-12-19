# zkWasm Protocol Overview

## Overview

zkWasm-Protocol is a specialized blockchain protocol designed to facilitate the settlement of zkWASM mini-rollups. It provides a robust framework for handling zero-knowledge proof verification and transaction processing in a secure and efficient manner. Specifically, it enables deposits and withdrawals of assets between the zkWASM mini-rollup application and the on-chain zkWasm-Protocol.

## Core Components

- Proxy Contract: The central hub for transaction processing and state management for a specific application.
- Verifier (interface): Handles zero-knowledge proof verification, each blockchain can have its own verifier, and it can be used for all the applications which have deployed its proxy contract.
- Transaction (interface): Supports standardized 32-byte transaction format for processing side effects such as withdrawals which are triggered by settlement (proof verification). 
- Other Contracts: Additional contracts like ERC20 and NFT Bidding for extended functionality

### Proxy Contract

#### Initialization

When you deploy a proxy contract, you set the chain id and the merkle root of the zkwasm application image: 

```solidity
constructor(uint32 chain_id, uint256 root) {
    _proxy_info.chain_id = chain_id;
    _proxy_info.owner = msg.sender;
    merkle_root = root;
    rid = 0;
}
```

And the owner is set to be the deployer of the proxy contract.

#### State Management

In proxy contract, we have plenty of state variables to manage the state of the application:

```solidity
TokenInfo[] public _tokens;
Transaction[] public transactions;
DelphinusVerifier public verifier;
ProxyInfo _proxy_info;

address internal _settler;
uint256[3] public zk_image_commitments;
uint256 public merkle_root;
uint256 public rid;
uint256 public withdrawLimit = 10000 * 1e18; //10000 Ti limit per settle

mapping(uint256 => bool) private _tmap;
mapping(uint256 => bool) private hasSideEffect;
```

Where:

- `TokenInfo[] public _tokens;` is an array of all the tokens supported by the application.
- `Transaction[] public transactions;` is an array of all the transaction types added to the proxy contract, such as withdrawals. 
- `DelphinusVerifier public verifier;` is the verifier for the application.
- `ProxyInfo _proxy_info;` is the information of the proxy contract, including the chain id, added token amount, added pool amount, owner address, merkle root, rid, verifier id, etc.
- `address internal _settler;` is the address of the settler, which can be set by the owner of the proxy contract.
- `uint256[3] public zk_image_commitments;` is the commitments of the zkwasm application image.
- `uint256 public merkle_root;` is the merkle root of the zkwasm application image.
- `uint256 public rid;` indicates the settlement round id or number.
- `uint256 public withdrawLimit = 10000 * 1e18; //10000 Ti limit per settle` is the limit of the withdrawals.
- `mapping(uint256 => bool) private _tmap;` is a mapping of the token uids to the boolean value, indicating whether the token is supported by the application.
- `mapping(uint256 => bool) private hasSideEffect;` is a mapping of the transaction uids to the boolean value, indicating whether the transaction has side effects.

After the proxy contract is deployed, you have to specify the verifier contract address for the application, which is the verifier that will be used to verify the proofs of the transactions:

```solidity
function setVerifier(address vaddr) public onlyOwner {
    verifier = DelphinusVerifier(vaddr);
}
```

You can also set the withdraw limit for the application, which is the maximum amount of withdrawal that can be processed in a single transaction:

```solidity
function setWithdrawLimit(uint256 amount) public onlyOwner {
    withdrawLimit = amount;
}
```

Moreover, in order to ensure the verification is not misused (for example, someone may verify a proof generated from a different zkWasm application), we have to set the image commitment for the application:

```solidity
function setVerifierImageCommitments(uint256[3] calldata commitments) external onlyOwner {
    zk_image_commitments[0] = commitments[0];
    zk_image_commitments[1] = commitments[1];
    zk_image_commitments[2] = commitments[2];
}
``` 

You can get the image commitment when you add the image to the [zkWasm Hub](https://explorer.zkwasmhub.com/) or compute it by yourself using `ts/commitment.ts` in the [zkWasm Mini=Rollup Repository](https://github.com/DelphinusLab/zkwasm-mini-rollup).

#### Access Control

There are two kinds of access control in the proxy contract:

```solidity
modifier onlyOwner() {
    require(msg.sender == _proxy_info.owner, "Only owner can call this function");
    _;
}

modifier onlySettler() {
    require(msg.sender == _settler, "Only settler can call this function");
    _;
}
```

- `onlyOwner` modifier is used to restrict access to the owner of the proxy contract.
- `onlySettler` modifier is used to restrict access to the settler of the proxy contract.

The owner and settler can be set by the owner of the proxy contract:

```solidity
function setOwner(address new_owner) external onlyOwner {
    _proxy_info.owner = new_owner;
}

function setSettler(address settler) external onlyOwner {
    _settler = settler;
}
```

#### Token Management

An important feature of the proxy contract is to manage the tokens supported by the application, which is used for settlement operations such as transfer, withdraw, deposit, etc. 

The owner of the proxy contract can add or modify the tokens supported by the application:

```solidity
function addToken(uint256 token) public onlyOwner returns (uint32) {
    uint32 cursor = uint32(_tokens.length); // cursor is the index of the token in the _tokens array
    _tokens.push(TokenInfo(token)); // add the token to the _tokens array
    _proxy_info.amount_token = cursor + 1; // update the amount of the tokens supported by the application
    require(_tmap[token] == false, "AddToken: Token Already Exist"); // check if the token is already supported
    if (token != 0) {
        _tmap[token] = true; // set the token to be supported
    }
    return cursor; // return the index of the token in the _tokens array
}


function modifyToken(uint32 index, uint256 token) public onlyOwner{
    require(_tmap[token] == false, "AddToken: Token Already Exist");

	// Check if the index is within bounds of the array
	require(index < _tokens.length, "Index out of bounds");

	// Modify the token at the specified index
    _tokens[index].token_uid = token;
}
```

You may notice that the `token` parameter in the `addToken` and `modifyToken` functions is of type `uint256` instead of `address`. This is because the `token` is the uid of the token, which is derived from the token address and the chain id, for example, in `test/test_utils.ts`, we have the following code to derive the uid of the token and add it to the proxy contract:

```ts
import { ethers, getChainId } from "hardhat";
import { encodeL1address } from "web3subscriber/src/addresses";
const chainId = await getChainId();
let tokenUid = encodeL1address(tokenAddress.replace("0x", ""), Number(chainId).toString(16));
let tokenUidString = tokenUid.toString(16);
await proxy.addToken(ethers.toBigInt("0x" + tokenUidString));
```

Or in the proxy contract itself, we have `_l1_address` function to derive the uid of a token or an account:

```solidity
function _l1_address(address account) public view returns (uint256) {
    return
        (uint256(uint160(account))) +
        (uint256(_proxy_info.chain_id) << 160);
}
```

Why add chain id to the uid? This is because the uid is used to identify the token, and the chain id is used to identify the chain, so the uid is unique for a specific token on a specific chain. 

You can find if an uid is specific to the blockchain (by checking if the chain id included in the uid is the same as the chain id of the proxy contract) by calling the `_is_local` function:

```solidity
function _is_local(uint256 l1address) public view returns (bool) {
    return ((l1address >> 160) == (uint256(_proxy_info.chain_id)));
}
```

You can view all the uids of the tokens supported by the application by calling the `allTokens` function:

```solidity
function allTokens() public view returns (TokenInfo[] memory) {
    return _tokens;
}
```

#### Transaction Management

The owner of the proxy contract can add the transaction types supported by the application:

```solidity
function addTransaction(address txaddr, bool sideEffect) public onlyOwner returns (uint256) {
    uint256 cursor = transactions.length; // cursor is the index of the transaction type in the transactions array
    require(transactions.length < 255, "TX index out of bound"); // check if the index of the transaction type is within bounds of the array
    transactions.push(Transaction(txaddr)); // add the transaction type to the transactions array
    if (sideEffect) {
        hasSideEffect[cursor] = sideEffect; // set if the transaction type has side effects
    }
    return cursor; // return the index of the transaction type in the transactions array
}
```

A common misunderstanding is to think that the transaction type is a "real" transaction which can be executed, but it is not. It is just a transaction type which represents a specific kind of transaction such as withdrawals. The parameter `txaddr` is the address of the deployed transaction type contract which implements the `Transaction` interface:

```solidity
pragma solidity ^0.8.0;

uint8 constant _WITHDRAW = 0x0;

interface Transaction {
    function sideEffect(bytes calldata args, uint cursor) external pure returns (uint256[] memory);
}
```

Here's an example of Withdraw.sol which implements the `Transaction` interface:
```solidity
pragma solidity ^0.8.0;
import "../Transaction.sol";

contract Withdraw is Transaction {
    /*
     * u8:op
     * u8:token_index
     * u16: reserve (le mode)
     * u160: addr (be mode)
     * u64: amount (be mode)
     */
    function sideEffect(bytes memory witness, uint256 cursor) public pure override returns (uint256[] memory) {
        uint256[] memory ops = new uint256[](4);
        uint256 data32; 
        uint256 offset = cursor + 32;
        assembly {
            // Load the 32 bytes of data from memory
            data32 := mload(add(witness, offset))
        }
        //ops[0] = uint256( (data32 >> (31*8)) & 0xFF );
        ops[0] = _WITHDRAW;
        // token index
        ops[1] = uint256( (data32 >> (30*8)) & 0x00FF );
        // amount to withdraw (wei not considered
        ops[2] = uint256( data32 & 0xFFFFFFFFFFFFFFFF );
        // recipient address
        ops[3] = uint256( (data32 >> (8*8)) & 0x00FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF );
        return ops;
    }
}
```
You will need to deploy the Withdraw.sol contract and add its address to the proxy contract through the `addTransaction` function to enable the withdrawals.

#### Settlement Flow
If you have deployed the proxy contract, transaction types contracts, and added the transaction types and tokens, you can start the settlement flow by calling the `verify` function:

```solidity
function verify(
    bytes calldata tx_data,
    uint256[] calldata proof,
    uint256[] calldata verify_instance,
    uint256[] calldata aux,
    uint256[][] calldata instances
) onlySettler nonReentrant public{...}
```

The `verify` function takes the following parameters:

- `tx_data`: the data of the transactions to be processed.
- `proof`: the proof of the transactions and state transition.
- `verify_instance`: the commitments of the application image.
- `aux`: the auxiliary data of the proof which is used to support the verification.
- `instances`: the instances of the proof, including the sha256 hash of the transactions and merkle root before and after the state transition.


To verify the proof, the function first checks if the commitments of the application image are correct:

```solidity
if (zk_image_commitments[0] != 0) {
    require(verify_instance[1] == zk_image_commitments[0], "Invalid image commitment 0");
    require(verify_instance[2] == zk_image_commitments[1], "Invalid image commitment 1");
    require(verify_instance[3] == zk_image_commitments[2], "Invalid image commitment 2");
}
```

Then it checks if the transaction data is in valid format and the sha256 hash of the transaction data is consistent with the instances:

```solidity
if (tx_data.length > 1) {
    require(
        tx_data.length % OP_SIZE == 0,
        "Verify: Insufficient delta operations"
    );

    uint256 sha_pack = uint256(sha256(tx_data));
    require(
        sha_pack ==
            (instances[0][8] << 192) +
                (instances[0][9] << 128) +
                (instances[0][10] << 64) +
                instances[0][11],
        "Inconsistent: Sha data inconsistent"
    );
}
```

Then it checks if the merkle root before the state transition is consistent with the instances:

```solidity
require(
    merkle_root ==
        (instances[0][0] << 192) +
            (instances[0][1] << 128) +
            (instances[0][2] << 64) +
            instances[0][3],
    "Inconsistent: Merkle root dismatch"
);
```

Then it calls the `verify` function of the verifier contract to verify the proof and process the transactions:

```solidity
if (tx_data.length > 1) {
    sideEffectCalled = perform_txs(tx_data);
}

uint256 new_merkle_root = (instances[0][4] << 192) +
    (instances[0][5] << 128) +
    (instances[0][6] << 64) +
    instances[0][7];
```

Finally, it checks if the new merkle root is consistent with the instances and updates the old merkle root with the new merkle root:

```solidity
uint256 new_merkle_root = (instances[0][4] << 192) +
    (instances[0][5] << 128) +
    (instances[0][6] << 64) +
    instances[0][7];
rid = rid + 1; // increment the rid
merkle_root = new_merkle_root; // update the merkle root
```

You may notice that the instances array use bit shift to pack the data, which is a common technique to save space in the proof, here's how the data is packed:

```solidity
instances[0] layout (each element is 64 bits):

[0-3]:   Current Merkle Root (256 bits total)
  [0] - Highest 64 bits
  [1] - Second highest 64 bits
  [2] - Third highest 64 bits
  [3] - Lowest 64 bits

[4-7]:   New Merkle Root (256 bits total)
  [4] - Highest 64 bits
  [5] - Second highest 64 bits
  [6] - Third highest 64 bits
  [7] - Lowest 64 bits

[8-11]:  Transaction Data Hash (256 bits total)
  [8] - Highest 64 bits
  [9] - Second highest 64 bits
  [10] - Third highest 64 bits
  [11] - Lowest 64 bits

instances[0] array (12 elements):
+---------------+---------------+---------------+---------------+
|     [0]       |     [1]       |     [2]       |     [3]       |
| Current Root  | Current Root  | Current Root  | Current Root  |
| (bits 255-192)| (bits 191-128)| (bits 127-64) | (bits 63-0)   |
+---------------+---------------+---------------+---------------+
|     [4]       |     [5]       |     [6]       |     [7]       |
|  New Root     |  New Root     |  New Root     |  New Root     |
| (bits 255-192)| (bits 191-128)| (bits 127-64) | (bits 63-0)   |
+---------------+---------------+---------------+---------------+
|     [8]       |     [9]       |    [10]       |    [11]       |
|   TX Hash     |   TX Hash     |   TX Hash     |   TX Hash     |
| (bits 255-192)| (bits 191-128)| (bits 127-64) | (bits 63-0)   |
+---------------+---------------+---------------+---------------+
```

#### Transaction Execution

In settlement, the `perform_txs` function is called to execute the transactions:

```solidity
function perform_txs(
    bytes calldata tx_data
) private returns (uint256){
    uint256 ret = 0;
    uint256 batch_size = tx_data.length / OP_SIZE;
    for (uint i = 0; i < batch_size; i++) {
        uint8 op_code = uint8(bytesToUint(tx_data, i * OP_SIZE, 1));
        require(transactions.length > op_code, "TX index out of bound");
        if (hasSideEffect[op_code]) {
            Transaction transaction = _get_transaction(op_code);
            uint256[] memory update = transaction.sideEffect(tx_data, i * OP_SIZE);
            ret += 1;
            _update_state(update);
        }
    }
    return ret;
}
```

The `tx_data` is the data of the transactions to be processed, each transaction is encoded in 32 bytes (`OP_SIZE = 32` so we divide the `tx_data` by `OP_SIZE` to get the number of transactions), and the `op_code` is the index of the transaction type in the `transactions` array: 

```
Transaction Data Structure (32 bytes total)
   [1 byte]  op_code (uint8)     // Operation code for transaction type, maximum 255 types
   [1 byte]  token_index         // Index of the token in registry, maximum 255 tokens
   [2 bytes] reserved            // Reserved for future use
   [20 bytes] address            // Ethereum address (160 bits)
   [8 bytes] amount             // Transaction amount value
```

For each transaction that has side effects, the function get the transaction type from the `transactions` array and call the `sideEffect` function of the transaction type contract to get the state updates information. Let's take a look at the `Withdraw.sol` contract to see how the state updates information is returned:

```solidity
uint8 constant _WITHDRAW = 0x0;
function sideEffect(bytes memory witness, uint256 cursor)
    public
    pure
    override
    returns (uint256[] memory)
{
    uint256[] memory ops = new uint256[](4);

    uint256 data32;
    uint256 offset = cursor + 32;
    assembly {
        // Load the 32 bytes of data from memory
        data32 := mload(add(witness, offset))
    }
    //ops[0] = uint256( (data32 >> (31*8)) & 0xFF );
    ops[0] = _WITHDRAW;
    // token index
    ops[1] = uint256( (data32 >> (30*8)) & 0x00FF );
    // amount to withdraw (wei not considered
    ops[2] = uint256( data32 & 0xFFFFFFFFFFFFFFFF );
    // recipient address
    ops[3] = uint256( (data32 >> (8*8)) & 0x00FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF );
    return ops;
}
```

Where `witness` is the complete transaction data (tx_data) and `cursor` is the starting position of current transaction (i * OP_SIZE, where i is the index of the transaction in the batch and OP_SIZE is 32). 

It returns an array of 4 elements (ops), where the ops has the following layout:
```
+-------------+-------------+------------------+--------------------+
|    ops[0]   |   ops[1]    |      ops[2]      |       ops[3]       |
|  Operation  | Token Index |     Amount       | Recipient Address  |
| (constant)  |  (8 bits)   |    (64 bits)     |    (160 bits)      |
+-------------+-------------+------------------+--------------------+
```

!!! note "Note"
    You may find it hard to understand the assembly code and bit shift operations. You may skip this section if you are not interested in the details. 
    
    First, let's understand how bytes memory is laid out:
    ```
    Memory Layout for 'bytes':
    +---------------+------------------------+-----------------+
    | Length (32B)  | Actual Data            | ...             |
    +---------------+------------------------+-----------------+
    ← 0x00          ← 0x20                  ← 0x40 ...
    ```

    - First 32 bytes (0x00-0x1F) stores the length of the bytes array located at the start of the memory allocation.
    - Next 32 bytes (0x20-0x3F) stores the actual data of the bytes array. This is where our first transaction starts.
    
    And here's structure of the transaction data:
    ```
    Memory Position Calculation:
    +---------------+------------------+------------------+------------------+
    | Length (32B)  |  Transaction 1   | Transaction 2    | Transaction 3    |
    +---------------+------------------+------------------+------------------+
    ← 0x00          ← 0x20             ← 0x40             ← 0x60
                    ← cursor=0         ← cursor=32        ← cursor=64
                    ← offset=32        ← offset=64        ← offset=96
    ```

    And that's why cursor needs to be added by 32, it is used to get the memory position of the transaction data to skip the length of the bytes array.

    ```solidity
    uint256 data32;                    // Will hold our 32 bytes of data
    uint256 offset = cursor + 32;      // Calculate the memory position
    assembly {
        // add(witness, offset): Calculate final memory address
        // mload: Load 32 bytes from that address into data32
        data32 := mload(add(witness, offset))
    }
    ``` 

    The `add` function is used to calculate the final memory address by adding the offset to the witness pointer. The `mload` function is used to load 32 bytes from that address into `data32`. Therefore, `data32` will hold the 32 bytes of the transaction data we want to extract.

    And the `data32` is then used to extract the operation code, token index, amount, and recipient address from the transaction data. For example:

    ```solidity
    ops[1] = uint256( (data32 >> (30*8)) & 0x00FF ); //extract the token index
    ```

    This operation shifts `data32` right by 30*8 bits (which is 240 bits) and then performs a bitwise AND with `0x00FF` (which is 255 in decimal). This effectively extracts the last 8 bits of `data32`, which correspond to the token index: 

    ```
    Original data32 (32 bytes):
    +----+----+------+--------------------+------------------+
    |op  |tidx|reserv|       address      |      amount      |
    +----+----+------+--------------------+------------------+
     31    30   29-28        27-8                 7-0

    Step 1: Right Shift (>> 30*8)
    Before: 0x[01][02][0000][1234...5678][000F4240]
                    ↓ shift right 240 bits (30 bytes)
    After:  0x0000...0000[01][02]

    Step 2: AND with 0x00FF
    0x0000...0000[01][02]
             AND
    0x0000...0000[00][FF]
             =
    0x0000...0000[00][02] ← Final token index value
    ```

After get the ops (which contains the operation code, token index, amount, and recipient address), the function `perform_txs` will call the `_update_state` function to execute the transaction and update the state of the application:

```solidity
/*
* @dev side effect encoded in the update function
* deltas = [| opcode; args |]
*/
function _update_state(uint256[] memory deltas) private {
    uint256 cursor = 0;
    while (cursor < deltas.length) {
        uint256 delta_code = deltas[cursor];
        if (delta_code == _WITHDRAW) {
            require(
                deltas.length >= cursor + 4,
                "Withdraw: Insufficient arg number"
            );
            _withdraw(
                uint128(deltas[cursor + 1]),
                uint128(deltas[cursor + 2]),
                deltas[cursor + 3]
            );
            cursor = cursor + 4;
        } else {
            revert("SideEffect: UnknownSideEffectCode");
        }
    }
}
```

Note that the `deltas` array is the `update` array in the `perform_txs` function and it is the `ops` array in the `sideEffect` function. The function matches the `delta_code` with the operation code and call the corresponding function (`_withdraw`) to update the state of the application: 

```solidity
/* In convention, the wasm image does not take wei into consideration thus we need to apply  amout * 1e18 * to get the actual amount of withdraw. Please make sure the wei of the withdraw token is 18
*/
function _withdraw(
    uint128 tidx,
    uint128 amount,  //in ether
    uint256 l1recipent
) private {
    uint256 tokenid = get_token_uid(tidx);
    if (_is_local(tokenid)) {
        address token = address(uint160(tokenid));
        address recipent = address(uint160(l1recipent));
        // Sanity checks
        require(recipent != address(0), "Withdraw to the zero address");
        IERC20 underlying_token = IERC20(token);
        uint256 balance = underlying_token.balanceOf(address(this));
        require(balance >= amount * 1e18, "Insufficient balance for withdraw");
        require(amount * 1e18 <= withdrawLimit, "Withdraw amount exceed limit");
        TransferHelper.safeTransfer(address(underlying_token), recipent, amount * 1e18);
        emit WithDraw(token, recipent, amount * 1e18);
    }
}
```

The `_withdraw` function is straightforward, it will transfer the amount of the withdraw token to the recipient address.

### Verifier Interface

This is the interface of the verifier contract, which is used to verify the proof and process the transactions in the proxy contract.

```solidity
interface DelphinusVerifier {
    /**
     * @dev snark verification stub
     */
    function verify (
        uint256[] calldata proof,
        uint256[] calldata verify_instance,
        uint256[] calldata aux,
        uint256[][] calldata target_instance
    ) external view;
}
```
Parameters

 - proof
    - Contains the zero-knowledge proof data
    - Generated by the zkWasm prover
    - Used to verify state transitions
 - verify_instance
    - Contains verification instance data
    - Includes image commitments
 - aux
    - Auxiliary data for proof verification
    - Contains helper values for computation
    - Used to optimize verification process
 - target_instance
    - 2D array containing state transition data
    - First dimension: Multiple instances
    - Second dimension: State values such as merkle root and transaction hash.
    - Structure matches the instances array format described earlier



### Transaction Interface

The transaction interface is used to define the transaction types supported by the application. The transaction type contract must implement the `sideEffect` function to return the state updates information.

```solidity
uint8 constant _WITHDRAW = 0x0;
// Additional operation codes can be defined here

interface Transaction {
    function sideEffect(bytes calldata args, uint cursor) external pure returns (uint256[] memory);
}
```

You can define your own operation code for the transaction type contract which implements this interface. The `sideEffect` function shall return the state updates information in the format like follows:

```
[operation, param1, param2, param3, ...]

For WITHDRAW operations:
[_WITHDRAW, token_index, amount, recipient_address]
```



