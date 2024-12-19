# Withdraw Workflow

## Introduction

In this section, we will use what we have learned from [zkWasm Protocol Overview](zkWasm%20Protocol.md) and [Rollup Settlement Monitor](../zkwasm-mini-rollup/Rollup%20Server.md#rollup-settlement-monitor-tssettlets) to implement a withdraw workflow. 

## Withdraw flow in the application

As a kind of transaction, the withdraw function can be implemented in the `Transaction` struct which we have discussed a little bit in the [Quick Tutorial](../Quick%20Tutorial.md#3-transaction-handler). Take [automata](https://github.com/riddles-are-us/zkwasm-automata/blob/main/src/state.rs) as an example, the withdraw function is implemented in the `Transaction` struct as follows:

```rs
pub fn withdraw(&self, pid: &[u64; 2]) -> Result<(), u32> {
    let mut player = AutomataPlayer::get_from_pid(pid);
    match player.as_mut() {
        None => Err(ERROR_PLAYER_NOT_EXIST),
        Some(player) => {
            player.check_and_inc_nonce(self.nonce);
            let amount = self.data[0] & 0xffffffff;
            player.data.cost_balance(amount as i64)?;
            let withdrawinfo =
                WithdrawInfo::new(&[self.data[0], self.data[1], self.data[2]], 0);
            SettlementInfo::append_settlement(withdrawinfo);
            player.store();
            Ok(())
        }
    }
}
```

The above code first checks if the player exists, then checks if the nonce is valid, then deducts the balance of the player, and finally appends the withdraw information to the settlement information. And here's how a player can trigger the withdraw function, in the frontend: 

```ts
async withdrawRewards(address: string, amount: bigint) {
    let nonce = await this.getNonce();
    let addressBN = new BN(address, 16);
    let a = addressBN.toArray("be", 20); // 20 bytes = 160 bits and split into 4, 8, 8
    /*
    (32 bit amount | 32 bit highbit of address)
    (64 bit mid bit of address (be))
    (64 bit tail bit of address (be))
     */
    let firstLimb = BigInt('0x' + bytesToHex(a.slice(0,4).reverse()));
    let sndLimb = BigInt('0x' + bytesToHex(a.slice(4,12).reverse()));
    let thirdLimb = BigInt('0x' + bytesToHex(a.slice(12, 20).reverse()));
    try {
      let processStamp = await this.rpc.sendTransaction(
        new BigUint64Array([
          createCommand(nonce, CMD_WITHDRAW, 0n),
          (firstLimb << 32n) + amount,
          sndLimb,
          thirdLimb
        ]), this.processingKey);
      console.log("withdraw rewards processed at:", processStamp);
    } catch(e) {
      if (e instanceof Error) {
        console.log(e.message);
      }
      console.log("collect reward error at address:", address);
    }
}
```

The function first converts the hexadecimal address string into a byte array, then the 20-byte address is then split into three parts. For example, if we have an address `0x742d35Cc6634C0532925a3b844Bc454e4438f44e`, it is split as follows:

- firstLimb:  `[74 2d 35 cc]`           (4 bytes, 32 bits, used to combine with the withdrawal amount) 
- sndLimb:    `[66 34 c0 53 29 25 a3 b8]` (8 bytes, 64 bits)
- thirdLimb:  `[44 bc 45 4e 44 38 f4 4e]` (8 bytes, 64 bits)

The withdrawal amount needs to be combined with part of the address. This is done by:
```typescript
(firstLimb << 32n) + amount
```

This operation:

1. Shifts the first 4 bytes of the address left by 32 bits
2. Adds the withdrawal amount in the lower 32 bits

Finally we send the transaction to the rollup server by creating a command with the `CMD_WITHDRAW` command.

The users can withdraw their rewards by calling the `withdrawRewards` function:

```ts
await this.withdrawRewards(address, amount);
```

And this will be processed in the `process` function of the `Transaction` struct in `state.rs` and the withdraw function will be called:

```rs
WITHDRAW => self
    .withdraw(&AutomataPlayer::pkey_to_pid(pkey))
    .map_or_else(|e| e, |_| 0),
```

## Withdraw flow in the rollup server

You may notice that the withdraw function just appends the withdraw information to the settlement information and then returns. Remember that the withdraw transactions will be submitted with `merkle_root` and `transactions_witness` to the zkWasm Hub or your own zkWasm Prover to generate proofs when the rollup server is ready to settle, you may have a look at [Transaction Installation (Rollup)](../zkwasm-mini-rollup/Rollup%20Server.md#transaction-installation-rollup). This takes several steps:

1. First, the server will check if the application has reached the [preemption threshold](../zkWasm%20Rust%20SDK.md#3-preemption-check):

    ```
    if (application.preempt()){
        ...
    }
    ```

2. If the application has reached the preemption threshold, the server will call the `finalize` function to get the settlement information (withdraw information):

    ```
    let txdata = application.finalize();
    ```

    Let's take a look at the `finalize` function in [zkWasm Mini Rollup's ABI](https://github.com/DelphinusLab/zkwasm-mini-rollup/blob/main/abi/src/lib.rs):

    ```rs
    #[wasm_bindgen]
    pub fn finalize() -> Vec<u8> {
        unsafe {
            let bytes = $S::flush_settlement();
            $S::store();
            bytes // This is the txdata
        }
    }
    ```

    The `finalize` function will call the implemented `flush_settlement` function in your application to get the settlement information which the application has appended in the withdraw function and return it to the server.

    !!! note
        For the details of the `ABI` which represents the workflow of rollup, you may have a look at [zkWasm Rust SDK](../zkWasm%20Rust%20SDK.md#5-zkwasm-rest-abi)


3. Finally, if your server are not in the tryrun mode (which means you are in [deploy mode](../Quick%20Tutorial.md#deploy-your-rollup-application)), the txdata will be sent to the zkWasm Hub or your own zkWasm Prover with other data to generate proofs:

    ```ts
    let task_id = await submitProofWithRetry(merkle_root, transactions_witness, txdata);
    ```

    Then a bundleRecord will be created in the zkWasm Mini Rollup's database:

    ```ts
    const bundleRecord = new modelBundle({
        merkleRoot: merkleRootToBeHexString(merkle_root),
        taskId: task_id,
    });
    await bundleRecord.save();
    ```

    This will be used later when we verify the proofs and settle on the blockchain.

    !!!note "Merkle Root"
        The submitted `merkle_root` is the old root (which indicates the root before the current batch of transactions).

## Settle on the blockchain

Let's assume the proof has been generated successfully by zkWasm Hub or your own zkWasm Prover, and you have already deployed your [proxy contract](./zkWasm%20Protocol.md). The next step is to settle on the blockchain. In order to do this, we need a settlement monitor to periodically verify the proofs. You may first take a look on [Rollup Settlement Monitor](../zkwasm-mini-rollup/Rollup%20Server.md#rollup-settlement-monitor-tssettlets) to learn how to implement a settlement monitor. In its core function `trySettle`, you can find the following code:

```ts
let merkleRoot = await getMerkle();
```

The function first gets the merkle root from the proxy contract, which is the old root as we have mentioned in the note of the previous section. Then it will get the bundleRecord from the database by the old root: 

```ts
let record = await modelBundle.findOne({ merkleRoot: merkleRoot });
```

It then gets all the data needed for the proof verification from zkWasm Hub by the taskId from the bundleRecord:

```ts
let taskId = record.taskId;
let data0 = await getTaskWithTimeout(taskId, 60000);
```

After getting all the data, the monitor will verify the proof and settle on the blockchain if the proof is valid:

```ts
let txData = new Uint8Array(data0.input_context);
let proofArr = new U8ArrayUtil(data0.proof).toNumber();

let shadowInstances = data0.shadow_instances;
let batchInstances = data0.batch_instances;
let verifyInstancesArr = shadowInstances.length === 0
    ? new U8ArrayUtil(batchInstances).toNumber()
    : new U8ArrayUtil(shadowInstances).toNumber();
let auxArr = new U8ArrayUtil(data0.aux).toNumber();
let instArr = new U8ArrayUtil(data0.instances).toNumber();

const tx = await proxy.verify(txData, proofArr, verifyInstancesArr, auxArr, [instArr]);
const receipt = await tx.wait();
```

Then we can update the database with withdraw information and transaction hash after checking the withdraw parameters from blockchain events with the txData we got from the proof generation task of zkWasm Hub:

```ts
const r = decodeWithdraw(txData);
const s = await getWithdrawEventParameters(proxy, receipt);
const withdrawArray = [];

// Check if the lengths of the arrays are the same
let status = 'Done';
if (r.length !== s.length) {
    status = 'Fail';
    console.error("Arrays have different lengths,", r, s);
}
else {
    for (let i = 0; i < r.length; i++) {
        const rItem = r[i];
        const sItem = s[i];
        if (rItem.address !== sItem.address || rItem.amount !== sItem.amount) {
            console.log("Crash(Need manual review):");
            console.error(`Mismatch found: ${rItem.address}:${rItem.amount} ${sItem.address}:${sItem.amount}`);
            while (1); //This is serious error, while loop to trigger manual review. This can be replaced by other methods to decently handle this error.
            status = 'Fail';
            break;
        }
        else {
            record.withdrawArray.push({
                address: rItem.address,
                amount: rItem.amount,
            });
        }
    }
}
record.settleTxHash = tx.hash;
record.settleStatus = status;
await record.save();
```

Now you can refer to the [Settlement Flow](./zkWasm%20Protocol.md#settlement-flow) and [Transaction Execution](./zkWasm%20Protocol.md#transaction-execution) in zkWasm Protocol to learn how the proxy contract actually processes the withdraw transactions after the settlement monitor request to verify the proofs.



