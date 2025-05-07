# Deposit Workflow

## Introduction

In this section, we will use what we have learned from [zkWasm Protocol Overview](zkWasm%20Protocol.md) to implement a deposit workflow. 

## The Topup Function
In the proxy contract, we have a function `topup` which can be called by any user to deposit specific token into the zkWasm Rollup Application. 

```solidity
function topup (
    uint128 tidx,
    uint64 pid_1,
    uint64 pid_2,
    uint128 amount  //in wei
) nonReentrant public {
    uint256 tokenid = get_token_uid(tidx);
    require (_is_local(tokenid), "token is not a local erc token");
    address token = address(uint160(tokenid));
    IERC20 underlying_token = IERC20(token);

    uint256 balance = underlying_token.balanceOf(msg.sender);
    require(balance >= amount, "Insufficient Balance");

    uint256 allowance = underlying_token.allowance(msg.sender, address(this));
    require(allowance >= amount, "Insufficient Allowance");

    //USDT does not follow ERC20 interface so have to use the following safer method
    TransferHelper.safeTransferFrom(address(underlying_token), msg.sender, address(this), amount);

    //TBD: Charge fees to developer
    emit TopUp(_l1_address(token), msg.sender, pid_1, pid_2, amount);
}
```

The `topup` function accepts four parameters that control the deposit process:

 - The `tidx` parameter (uint128) represents the token index in the contract's token registry. This index is used to retrieve the actual token contract address through the `get_token_uid` function.
 - The `pid_1` and `pid_2` parameters (both uint64) together form a unique player identifier within the zkWasm system. This two-part identifier corresponds to the `player_id: [u64; 2]` structure used in the `Player` struct in the zkWasm mini rollup ABI. Therefore, we can use the `pid_1` and `pid_2` to identify which player is depositing the token.
 - The `amount` parameter (uint128) specifies the deposit amount in wei, representing the smallest unit of the token being deposited, this means the amount has to be multiplied by 10^18 to represent the actual amount of the token.

When a user calls the `topup` function, the following steps will be performed:

1. The `topup` function will first check if the token is a local token (which means the token is supported by the blockchain) by calling the `_is_local` function. If the token is not a local token, the function will revert with an error message.
2. The function will then check if the user has enough balance and allowance to deposit the token. If not, the function will revert with an error message.
3. The function will then transfer the token from the user's account to the proxy contract.
4. The function will then emit a `TopUp` event to notify the zkWasm Rollup Application that the user has deposited the token.

You can add more features to the `topup` function, such as charging fees to the developer, or checking if the user is on the whitelist.

Imagine a user has deposited a token into the zkWasm Rollup Application and the `TopUp` event has been emitted. What will happen next? 

Actually... nothing will happen next. The `TopUp` event is just a notification event, it does not trigger any further actions. So how does the zkWasm Rollup Application know that the user has deposited the token? We need to implement a mechanism to listen to the `TopUp` event and trigger the corresponding actions to process the deposit in the zkWasm Rollup Application. A good way to do this is to use a deposit monitor which operates by the server's admin.

## Deposit Monitor

In this section, we will implement a typescript node.js service that can listen to the `TopUp` event and trigger the corresponding actions to process the deposit in the zkWasm Rollup Application. The original code is in the `deposit.ts` [file](https://github.com/riddles-are-us/zkwasm-automata/blob/main/ts/src/deposit.ts) in a game called [automata](https://automata.zkplay.app/).

Let's start by implementing the service that can:
1. retrieve all past `TopUp` events from the proxy contract and process each event. 
2. listen to new `TopUp` events from the proxy contract and process each event.

The service can be concluded as the following code:
```ts
async function main() {
    const dbName = `${process.env.SETTLEMENT_CONTRACT_ADDRESS}_deposit`; // Dynamically set DB name using contract address
    // Connect to MongoDB (without deprecated options)
    await mongoose.connect(process.env.MONGO_URI, {
        dbName,
    }).then(() => console.log('MongoDB connected'));
    // Get all TopUp events from the contract
    await getTopUpEvents();
    // Listen for new TopUp events
    proxyContract.on('TopUp', async (l1token, address, pid_1, pid_2, amount, event) => {
        console.log(event);
        //const eventLog = event as EventLog;  // Explicitly cast to EventLog
        //console.log(eventLog);
        console.log(`New TopUp event detected: ${event.log.transactionHash}`);
        // Process the new TopUp event
        await processTopUpEvent(event.log);
    });
}
```

In order to make sure the events will not be processed multiple times, we need to store the events transaction information in the database. Here we use MongoDB to do this.

```typescript
const txSchema = new mongoose.Schema({
    txHash: { type: String, required: true, unique: true }, // Ensure txHash is unique
    state: { type: String, enum: ['pending', 'in-progress', 'completed', 'failed'], default: 'pending' },
    timestamp: { type: Date, default: Date.now },
    l1token: { type: String, required: true },
    address: { type: String, required: true },
    pid_1: { type: BigInt, required: true },
    pid_2: { type: BigInt, required: true },
    amount: { type: BigInt, required: true },
});
const TxHash = mongoose.model('TxHash', txSchema);
```
The above code defines a schema for the `TxHash` collection in the database. Each `TxHash` document will have fields that correspond to the parameters of the `TopUp` event. Notably, the state field is used to indicate the status of the event processing which can be used to prevent the event from being processed multiple times.

When we start the service, after connecting to the database, we will first retrieve all past `TopUp` events from the proxy contract and process each event. This is specifically useful for the server's admin to check if there are any deposits that need to be processed when initializing the monitor:

```typescript
// Function to retrieve all past TopUp events
async function getTopUpEvents() {
    try {
        // You can specify a block range, or use `fromBlock: 0` to query all events
        const filter = proxyContract.filters.TopUp();
        const events = await proxyContract.queryFilter(filter, 0, 'latest');
        console.log(`Found ${events.length} TopUp events.`);
        for (const event of events) {
            const eventLog = event;
            // Process each past event
            await processTopUpEvent(eventLog);
        }
    }
    catch (error) {
        console.error('Error retrieving TopUp events:', error);
    }
}
```

Then the service will listen to new `TopUp` events from the proxy contract and process each event:
```typescript
proxyContract.on('TopUp', async (l1token, address, pid_1, pid_2, amount, event) => {
    console.log(event);
    //const eventLog = event as EventLog;  // Explicitly cast to EventLog
    //console.log(eventLog);
    console.log(`New TopUp event detected: ${event.log.transactionHash}`);
    // Process the new TopUp event
    await processTopUpEvent(event.log);
});
```

Once there is a new `TopUp` event, the service will process the event by calling the `processTopUpEvent` function. 

Let's take a look at the `processTopUpEvent` function:

```ts
async function processTopUpEvent(event) {
    const eventLog = event;
    const [l1token, address, pid_1, pid_2, amount] = eventLog.args;
    if (l1token !== (await proxyContract._tokens(0))) {
        console.log('Skip not the right token: ', l1token);
        return;
    }
    console.log(`TopUp event received: pid_1=${pid_1.toString()}, pid_2=${pid_2.toString()}, amount=${amount.toString()} wei`);
    // Check if this transaction is already in the database and in 'pending' or 'in-progress' state
    let tx = await findTxByHash(event.transactionHash);
    ...
}
```

First, the function will check if the token is the right token (the token that can be deposited into the zkWasm Rollup Application) by comparing the `l1token` (which is the uid of the token) with the token uid stored in the proxy contract. If the token is not the right token, the function will skip the event. 

!!!tip "Tip"
    You can implement your own logic to check if the token is the right token by checking the token's uid, for example, if you want to make all the tokens that have been registered in the proxy contract can be deposited into the zkWasm Rollup Application, you can skip the check (because the token has been checked in the `topup` function in the proxy contract when user deposit the token) or check if the token is supported by the proxy contract by calling the `allTokens` function and check if the `l1token` is in the `allTokens` array.

Then the function will check if the transaction is already in the database by calling the `findTxByHash` function:

```ts
let tx = await findTxByHash(event.transactionHash);
```

If the transaction is not in the database, the function will create a new transaction record which has the initial state as `pending` in the database:

```ts
        if (!tx) {
            console.log(`Transaction hash not found: ${event.transactionHash}`);
            // Save tx hash and initial state as pending, along with other details
            tx = new TxHash({
                txHash: event.transactionHash,
                state: 'pending', // Initially set to pending
                l1token,
                address,
                pid_1,
                pid_2,
                amount,
            });
            await tx.save();
        }
```

If the transaction is already in the database or just created through the above code, the function will check if the transaction is in the `pending` status, if so, it will proceed to process the deposit in the zkWasm Rollup Application:

```ts
if (tx && tx.state === 'pending') {
    try {
        // Set transaction state to "in-progress"
        await updateTxState(event.transactionHash, 'in-progress');
        // Convert amount from wei to ether
        let amountInEther = amount / BigInt(10 ** 18);
        console.log("Deposited amount (in ether): ", amountInEther);
        if (amountInEther < 1n) {
            console.error(`--------------Skip: Amount must be at least 1 Titan (in ether instead of wei) ${event.transactionHash}\n`);
        }
        else {
            // Proceed with the deposit
            await admin.deposit(pid_1, pid_2, amountInEther);
            console.log(`------------------Deposit successful! ${event.transactionHash}\n`);
        }
        // After successful deposit, set state to 'completed'
        await updateTxState(event.transactionHash, 'completed');
    }
    catch (error) {
        console.error('Error during deposit:', error);
        // In case of failure, mark as 'failed'
        // await updateTxState(event.transactionHash, 'failed');
    }
}
```

The above code will set the transaction state to `in-progress`, then convert the amount from wei to ether, and then call the `deposit` function in the zkWasm Rollup Application to process the deposit. After the deposit is successful, the function will set the transaction state to `completed`. However, if the deposit fails, the status of the transaction will not be updated to `completed` and will remain as `in-progress`.

!!!info "Admin Setup"
    The `deposit` function is a function in the zkWasm Rollup Application, which is only callable by the admin. You will need to setup the admin key as `SERVER_ADMIN_KEY` in the `.env` file first before you can call the `deposit` function. Also, as a kind of "player", you will need to install the admin into the zkWasm Rollup Application by calling the `installPlayer` function:
    ```typescript
    const rpc = new ZKWasmAppRpc("https://rpc.zkplay.app");
    let admin = new Player(process.env.SERVER_ADMIN_KEY, rpc);
    console.log("install admin ...\n");
    await admin.installPlayer();
    ```

If the transaction is in the `in-progress` state, the function will skip the deposit process as this indicates that something went wrong during the deposit process and requires manual intervention.

If the transaction is in the `completed` state, the function will skip the deposit process as this indicates that the deposit has already been processed.

## Deposit Action

Let's take a look at how admin deposits the token into the zkWasm Rollup Application, in `process` function in state.rs of the zkWasm Rollup Application ([automata](https://github.com/riddles-are-us/zkwasm-automata/blob/main/src/state.rs)):

```rs
DEPOSIT => {
    unsafe { require(*pkey == *ADMIN_PUBKEY) };
    self.deposit(&AutomataPlayer::pkey_to_pid(pkey))
        .map_or_else(|e| e, |_| 0)
}
```
When it receives a deposit request (`admin.deposit(pid_1, pid_2, amountInEther)` called in the deposit monitor), it will check if the request is coming from the admin by checking if the public key is the admin's public key (You will get the admin's public key generated from your private key when you build the zkWasm Rollup Application using `make build`). If it is, it will call the `deposit` function.

Let's take a look at the `deposit` function in the zkWasm Rollup Application:

```rs
pub fn deposit(&self, pid: &[u64; 2]) -> Result<(), u32> {
    //zkwasm_rust_sdk::dbg!("deposit\n");
    let mut admin = AutomataPlayer::get_from_pid(pid).unwrap();
    admin.check_and_inc_nonce(self.nonce);
    let mut player = AutomataPlayer::get_from_pid(&[self.data[0], self.data[1]]);
    match player.as_mut() {
        None => {
            let mut player = AutomataPlayer::new_from_pid([self.data[0], self.data[1]]);
            player.data.cost_balance(-(self.data[2] as i64))?;
            player.store();
        }
        Some(player) => {
            player.data.cost_balance(-(self.data[2] as i64))?;
            player.store();
        }
    };
    admin.store();
    Ok(()) // no error occurred
}
```

Remember how we call the `deposit` function in the deposit monitor, we use the `pid_1` and `pid_2` to identify the player, and the `amount` to specify the amount of the token to be deposited:

```typescript
await admin.deposit(pid_1, pid_2, amountInEther);
```

So the `deposit` function will first get the player from the pid, then check if the player exists, if not, it will create a new player with the pid, then it will add the amount to the player's balance by `player.data.cost_balance(-(self.data[2] as i64))?;`.

!!!tip "Balance Update"
    The `cost_balance` function in the zkWasm Rollup Application handles both deposit and withdrawal operations through a clever use of positive and negative values. Here's how it works:

    When processing a deposit with `cost_balance(-x)`, the function:

    1. Checks if `treasure >= -x`
    2. Executes `treasure -= (-x)`, which is equivalent to `treasure += x`

    This is why in the deposit function we see `cost_balance(-(self.data[2] as i64))`:

    - First converts `self.data[2]` to a 64-bit integer
    - Takes its negative value to make it a deposit operation
    - Uses `cost_balance` to update the balance

    This is a common design pattern where a single function handles both increasing and decreasing balances, using the sign of the value to determine the operation type. When the input is negative, it becomes a deposit (increasing balance), and when positive, it becomes a withdrawal (decreasing balance).

    Here's the implementation:
    ```rust
    pub fn cost_balance(&mut self, b: i64) -> Result<(), u32> {
        if let Some(treasure) = self.local.0.last_mut() {
            if *treasure >= b {
                *treasure -= b;
                Ok(())
            } else {
                Err(ERROR_NOT_ENOUGH_BALANCE)
            }
        } else {
            unreachable!();
        }
    }
    ```

    The balance (or "treasure") is stored in the last element of the player's `local` array, which is part of the player's data structure containing balance and other information.












