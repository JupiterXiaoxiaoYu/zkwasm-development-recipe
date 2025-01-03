# Custom Transaction

## Introduction

In this section, we will implement a custom transaction workflow for minting NFTs. This example demonstrates how to create a custom transaction that interacts with an NFT contract on the blockchain through the zkWasm protocol.

!!! Prerequisite
    Before implementing the NFT minting workflow, you need to:
    
    1. Deploy a proxy contract
    2. Deploy an NFT contract
    3. Add the NFT contract to the proxy contract using the `AddTransaction` method mentioned in the [zkWasm Protocol Overview](zkWasm%20Protocol.md#proxy-contract).

## NFT Minting flow in the application

As a custom transaction, the NFT minting function can be implemented in the `Transaction` struct. Here's an example implementation:

```rs
pub fn mint_nft(&self, pid: &[u64; 2]) -> Result<(), u32> {
    let mut player = Player::get_from_pid(pid);
    match player.as_mut() {
        None => Err(ERROR_PLAYER_NOT_EXIST),
        Some(player) => {
            // Check and increment nonce to prevent replay attacks
            player.check_and_inc_nonce(self.nonce);
            
            // Extract NFT data from transaction data
            // data[0]: contains both address high bits and NFT level
            // data[1], data[2]: rest of the address
            let level = self.data[0] & 0xffffffff;
            let address_data = [self.data[0] >> 32, self.data[1], self.data[2]];
            
            // Create mint information for settlement
            let mint_info = MintInfo::new(&address_data, level);
            SettlementInfo::append_settlement(mint_info);
            
            // Store updated player state
            player.store();
            Ok(())
        }
    }
}

pub struct MintInfo {
    pub feature: u32,     // 4 bytes: token type (0xFF for NFT)
    pub address: [u8; 20], // 20 bytes: recipient address
    pub level: u64,       // 8 bytes: NFT level
}

impl MintInfo {
    pub fn new(limbs: &[u64; 3], level: u32) -> Self {
        let mut address = ((limbs[0]) as u32).to_le_bytes().to_vec();
        address.extend_from_slice(&limbs[1].to_le_bytes());
        address.extend_from_slice(&limbs[2].to_le_bytes());

        MintInfo {
            feature: 0xFF, // Special token index for NFT
            address: address.try_into().unwrap(),
            level: level as u64
        }
    }

    pub fn flush(&self, bytes: &mut Vec<u8>) {
        bytes.extend_from_slice(&self.feature.to_le_bytes());
        bytes.extend_from_slice(&self.address);
        bytes.extend_from_slice(&self.level.to_be_bytes()); // solidity needs big endian
    }
}
```

Here's how a user can trigger the NFT minting function from the frontend:

```ts
async mintNFT(address: string, level: bigint) {
    let nonce = await this.getNonce();
    let addressBN = new BN(address, 16);
    let a = addressBN.toArray("be", 20); // 20 bytes = 160 bits and split into 4, 8, 8
    /*
    (32 bit level | 32 bit highbit of address)
    (64 bit mid bit of address (be))
    (64 bit tail bit of address (be))
     */
    let firstLimb = BigInt('0x' + bytesToHex(a.slice(0,4).reverse()));
    let sndLimb = BigInt('0x' + bytesToHex(a.slice(4,12).reverse()));
    let thirdLimb = BigInt('0x' + bytesToHex(a.slice(12, 20).reverse()));
    try {
        let processStamp = await this.rpc.sendTransaction(
            new BigUint64Array([
                createCommand(nonce, CMD_MINT_NFT, 0n),
                (firstLimb << 32n) + level,
                sndLimb,
                thirdLimb
            ]), this.processingKey);
        console.log("NFT mint processed at:", processStamp);
    } catch(e) {
        if (e instanceof Error) {
            console.log(e.message);
        }
        console.log("NFT minting error for address:", address);
    }
}
```

Users can mint NFTs by calling the `mintNFT` function:

```ts
await this.mintNFT("YOUR_ADDRESS", 5n);
```

## NFT Minting flow in the rollup server

The minting process follows a similar pattern to other transactions in the rollup server:

1. The server checks for the preemption threshold
2. When reached, it calls the `finalize` function to collect settlement information
3. The collected mint information is included in the proof generation process:

```ts
let task_id = await submitProofWithRetry(merkle_root, transactions_witness, txdata);
```

The `txdata` in this case will contain the NFT minting information that will be used during on-chain settlement.

## Settle on the blockchain

The settlement monitor needs to handle both withdraw and NFT mint transactions. Here's the implementation:

```typescript
// Decode both withdraw and NFT mint transactions from txdata
export function decodeTxData(txdata: Uint8Array) {
    let withdraws = [];
    let nftMints = [];
    
    if (txdata.length > 1) {
        for (let i = 0; i < txdata.length; i += 32) {
            let extra = txdata.slice(i, i+4);
            let address = txdata.slice(i+4, i+24);
            let valueData = txdata.slice(i+24, i+32);
            
            if (extra[1] === 0xFF) { // NFT mint
                nftMints.push({
                    op: extra[0],
                    index: extra[1],
                    address: ethers.getAddress(bytesToHex(Array.from(address))),
                    level: Number(bytesToDecimal(Array.from(valueData))),
                });
            } else { // Withdraw
                withdraws.push({
                    op: extra[0],
                    index: extra[1],
                    address: ethers.getAddress(bytesToHex(Array.from(address))),
                    amount: ethers.parseEther(bytesToDecimal(Array.from(valueData))),
                });
            }
        }
    }
    return { withdraws, nftMints };
}

// Get NFT mint events from transaction receipt
async function getNFTMintEventParameters(
    proxy: ethers.Contract,
    receipt: ethers.ContractTransactionReceipt
): Promise<any[]> {
    let r: any[] = [];
    try {
        const eventSignature = "event NFTMinted(address nftContract, address recipient, uint256 level)";
        const iface = new ethers.Interface([eventSignature]);

        const logs = receipt.logs;
        logs.forEach(log => {
            try {
                const decoded = iface.parseLog(log);
                if (decoded) {
                    const nftContract = decoded.args.nftContract;
                    const recipient = decoded.args.recipient;
                    const level = decoded.args.level;
                    r.push({
                        nftContract,
                        address: recipient,
                        level: level,
                    });
                }
            } catch (error) {
                // Handle logs that don't match the event signature
            }
        });
    } catch (error) {
        console.error('Error retrieving NFT mint event parameters:', error);
    }
    return r;
}

async function trySettle() {
    let merkleRoot = await getMerkle();
    try {
        let record = await modelBundle.findOne({ merkleRoot: merkleRoot });
        if (record) {
            let taskId = record.taskId;
            let data0 = await getTaskWithTimeout(taskId, 60000);

            // Check failed or just timeout
            if (data0.proof.length == 0) {
                let data1 = await getTask(taskId, null);
                if (data1.status === "DryRunFailed" || data1.status === "Unprovable") {
                    console.log("Crash(Need manual review): task failed with state:", taskId, data1.status, data1.input_context);
                    while(1);
                    return -1;
                } else {
                    console.log(`Task: ${taskId}, ${data1.status}, retry settle later.`);
                    return -1;
                }
            }

            // Prepare proof data
            let shadowInstances = data0.shadow_instances;
            let batchInstances = data0.batch_instances;
            let proofArr = new U8ArrayUtil(data0.proof).toNumber();
            let auxArr = new U8ArrayUtil(data0.aux).toNumber();
            let verifyInstancesArr = shadowInstances.length === 0
                ? new U8ArrayUtil(batchInstances).toNumber()
                : new U8ArrayUtil(shadowInstances).toNumber();
            let instArr = new U8ArrayUtil(data0.instances).toNumber();
            let txData = new Uint8Array(data0.input_context);

            // Verify proof
            const proxy = new ethers.Contract(constants.proxyAddress, abiData.abi, signer);
            const tx = await proxy.verify(txData, proofArr, verifyInstancesArr, auxArr, [instArr]);
            const receipt = await tx.wait();

            // Decode and verify transactions
            const { withdraws, nftMints } = decodeTxData(txData);
            const withdrawEvents = await getWithdrawEventParameters(proxy, receipt);
            const nftMintEvents = await getNFTMintEventParameters(proxy, receipt);

            let status = 'Done';

            // Verify withdraws
            if (withdraws.length > 0) {
                if (withdraws.length !== withdrawEvents.length) {
                    status = 'Fail';
                    console.error("Withdraw arrays have different lengths", withdraws, withdrawEvents);
                } else {
                    for (let i = 0; i < withdraws.length; i++) {
                        const offchain = withdraws[i];
                        const onchain = withdrawEvents[i];
                        if (offchain.address !== onchain.address || offchain.amount !== onchain.amount) {
                            console.log("Crash(Need manual review):");
                            console.error(`Withdraw mismatch: ${offchain.address}:${offchain.amount} ${onchain.address}:${onchain.amount}`);
                            while(1);
                            status = 'Fail';
                            break;
                        }
                    }
                }
            }

            // Verify NFT mints
            if (nftMints.length > 0) {
                if (nftMints.length !== nftMintEvents.length) {
                    status = 'Fail';
                    console.error("NFT mint arrays have different lengths", nftMints, nftMintEvents);
                } else {
                    for (let i = 0; i < nftMints.length; i++) {
                        const offchain = nftMints[i];
                        const onchain = nftMintEvents[i];
                        if (offchain.address !== onchain.address || offchain.level !== onchain.level) {
                            console.log("Crash(Need manual review):");
                            console.error(`NFT mint mismatch: ${offchain.address}:${offchain.level} ${onchain.address}:${onchain.level}`);
                            while(1);
                            status = 'Fail';
                            break;
                        }
                    }
                }
            }

            // Update record
            record.settleTxHash = tx.hash;
            record.settleStatus = status;
            record.transactions = {
                withdraws: withdraws,
                nftMints: nftMints
            };
            await record.save();
            console.log("Receipt verified");
        } else {
            console.log(`proof bundle ${merkleRoot} not found`);
        }
    } catch(e) {
        console.log("Exception happen in trySettle()");
        console.log(e);
    }
}
```

This implementation includes the following key changes:

1. `decodeTxData` function:
    - Handles both withdraw and NFT mint transactions
    - Uses token index (0xFF) to differentiate between transaction types
    - Returns separate arrays for withdraws and NFT mints

2. Added `getNFTMintEventParameters`:
    - Similar to `getWithdrawEventParameters`
    - Decodes NFT mint events from transaction logs

3. Enhanced `trySettle`:
    - Verifies both types of transactions
    - Maintains separate verification logic for each type
    - Updates record with both withdraw and NFT mint information

4. Error handling:
    - Checks for mismatches in both transaction types
    - Triggers manual review if any verification fails
    - Maintains separate status tracking for different transaction types

This implementation ensures that both withdraw and NFT mint transactions are properly verified and recorded in the settlement process.

## Proxy Contract Modification

In [proxy contract](../zkwasm-protocol/zkWasm%20Protocol.md), you can implement the NFT minting functionality by adding the following code:

```solidity
// In your proxy contract
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
        } else if (delta_code == _MINT_NFT) {
            require(
                deltas.length >= cursor + 4,
                "MintNFT: Insufficient arg number"
            );
            _mintNFT(
                uint128(deltas[cursor + 1]), // token index (should be 0xFF)
                uint128(deltas[cursor + 2]), // level
                deltas[cursor + 3]          // recipient address
            );
            cursor = cursor + 4;
        } else {
            revert("SideEffect: UnknownSideEffectCode");
        }
    }
}

function _mintNFT(
    uint128 tidx,
    uint128 level,
    uint256 l1recipient
) private {
    require(tidx == 0xFF, "Invalid token index for NFT mint");
    address recipient = address(uint160(l1recipient));

    // Sanity checks
    require(recipient != address(0), "Mint to the zero address");
    require(level > 0, "Invalid NFT level");

    // Get NFT contract address from registry
    address nftContract = nftContracts[tidx];
    require(nftContract != address(0), "NFT contract not registered");

    // Generate URI based on level (implementation depends on your needs)
    string memory uri = generateNFTURI(level);

    // Call mint function on NFT contract
    INFTContract(nftContract).mint(recipient, level, uri);
    emit NFTMinted(nftContract, recipient, level);
}

// Event for NFT minting
event NFTMinted(address indexed nftContract, address indexed recipient, uint256 level);

// Interface for NFT contract
interface INFTContract {
    function mint(address to, uint256 level, string memory uri) external;
}
```

The key components of the proxy implementation are:

1. `_update_state` modification:
    - Added handling for `_MINT_NFT` operation code
    - Extracts token index, level, and recipient address from deltas
    - Calls `_mintNFT` function with the extracted parameters

2. `_mintNFT` function:
    - Verifies token index is 0xFF (special index for NFT)
    - Performs sanity checks on recipient address and level
    - Retrieves NFT contract address from registry
    - Generates URI based on level
    - Calls mint function on the NFT contract

This implementation allows the proxy to:

- Handle both withdraw and NFT mint operations
- Generate appropriate URIs based on NFT levels
- Emit events for tracking minting operations

The proxy acts as a coordinator between the rollup and the NFT contract, ensuring that minting operations are executed only after successful proof verification.

