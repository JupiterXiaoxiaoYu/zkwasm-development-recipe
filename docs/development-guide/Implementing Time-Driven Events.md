# Implementing Time-Driven Events

## Overview

Time-Driven Events are events that are triggered by the passage of time. For example, in a farming game, crops growing over time and eventually becoming harvestable would be a time-driven event.

These events can affect the state of a minirollup application. Using the farming game example, crop growth would be represented by changes in the crop's state, which could then trigger other events.

Traditional blockchain systems don't natively support time-driven events since blockchains cannot autonomously trigger state changes. However, we can simulate time-driven events using an external account controlled by an off-chain service.

In zkWasm, we have a server-side sequencer that manages the transaction queue. This sequencer can generate timetick transactions to trigger time-driven events (see the [Sequencer section in the server documentation](../zkwasm-mini-rollup/Rollup%20Server.md#sequencer-tsservicets)). This provides native support for time-driven events in zkWasm.

## Implementation Components

To implement time-driven events in zkWasm, we'll work with several components:

- zkWasm Mini Rollup (SDK & Server): 
    - zkWasm-ts-server: Generates timetick transactions
    - Event conventions: Defines data structures and event handling interfaces
- zkWasm Application:
    - state.rs: Manages application state
    - event.rs: Implements event handling methods

!!! note
    This guide uses the [automata game](https://github.com/riddles-are-us/zkwasm-automata) as an example. Your application may use different file names than `state.rs` and `event.rs`.

### Timetick Transactions

First, enable timetick transactions in your application's `config.ts` (default is true):

```rs
impl Config {
    ...
    pub fn autotick() -> bool {
        true
    }
}
```

Once enabled, the server (`service.ts` in `zkWasm-ts-server/src`) will generate timetick transactions at regular intervals:

```ts
// Generate timetick transactions if autotick is enabled
if (application.autotick()) {
    setInterval(async () => {
        try {
            await myQueue.add('autoJob', {command:0});
        } catch (error) {
            console.error('Error adding automatic job to the queue:', error);
            process.exit(1);
        }
    }, 5000); // Default interval: 5 seconds (adjustable)
}
```

You can customize the interval by modifying the `setInterval` timing parameter.

### Event Convention

The [event convention](../zkwasm-mini-rollup/Rollup%20Convention.md#event) defines the core data structures and interfaces for handling time-driven events. Let's examine the key components:

#### EventQueue Structure
The EventQueue implements a differential time queue (DTQ) for efficient event scheduling and processing:

```rs
pub struct EventQueue<T: EventHandler + Sized> {
    pub counter: u64,  // Total number of timeticks processed
    pub list: std::collections::LinkedList<T>,  // Ordered queue of pending events
}
```

#### EventHandler Interface
Events must implement the EventHandler trait, which defines the core event handling behavior:

```rs
pub trait EventHandler: Clone + StorageData {
    fn get_delta(&self) -> usize;  // Time until event triggers
    fn progress(&mut self, d: usize);  // Update event's remaining time
    fn handle(&mut self, counter: u64) -> Option<Self>;  // Process event and optionally chain a new one
    fn u64size() -> usize;  // Number of u64 fields in the event
}
```

#### EventQueue Implementation
The EventQueue provides several methods for managing events:

```rs
impl<T: EventHandler> EventQueue<T> {
    // Debug helper - prints queue state
    fn dump(&self, counter: u64)

    // Add new event to queue
    pub fn insert(&mut self, node: T)

    // Process due events and advance time
    pub fn tick(&mut self)

    // Storage management
    fn get_old_entries(&self, counter: u64) -> Vec<u64>
    fn set_entries(&self, entries: &Vec<u64>, counter: u64)
    pub fn store(&mut self)
}
```

Key operations include:

- `dump`: Prints the event queue and the delta of each event for debugging purposes. Shows the current counter and the delta time for each event in the queue.

- `insert`: Adds a new event into the event queue based on its delta time. The event queue is a differential time queue (DTQ), meaning events are sorted by their relative time differences. The method adjusts delta times of subsequent events to maintain proper time relationships.

- `tick`: The core processing function that:
    1. Retrieves and processes historical events from storage:
       ```rs
        let mut entries_data = self.get_old_entries(counter);
        let entries_nb = entries_data.len() / E::u64size();
        let mut dataiter = entries_data.iter_mut();
        let mut entries = Vec::with_capacity(entries_nb);
        for _ in 0..entries_nb {
            entries.push(E::from_data(&mut dataiter));
        }
       ```
    2. Handles all events with zero delta time in the queue through the `handle()` method. Each event's `handle()` method may generate new events that get inserted back into the queue:
    ```rs
    for mut e in entries {
        let m = e.handle(counter);

        if let Some(event) = m {
            self.insert(event);
        }
    }

    while let Some(head) = self.list.front_mut() {
        if head.get_delta() == 0 {
            let m = head.handle(counter);
            self.list.pop_front();
            if let Some(event) = m {
                self.insert(event); // new event generated and inserted back into the queue
            }
        }else{
            ...
        }
        ...
    }
    ```
    3. Progresses remaining events by reducing their delta by 1:
    ```rs
    if head.get_delta() == 0 {
        ...
    } else {
        head.progress(1);
        break; // break the while loop
    }
    ```
    4. Increments the counter by 1.

- `get_old_entries`: Retrieves historical event data from storage for a given counter value. Used to load previously stored events that need processing.

- `set_entries`: Stores event data in the merkle key-value storage for a specific counter value. This helps manage memory by persisting events that will be processed at the same time.

- `store`: Persists the entire event queue to the merkle storage. This ensures event data survives between application restarts as well as the rollup process.

Keep in mind these data structures, interface and methods, now we will leverage these components to implement time-driven events in zkWasm.

### State Management

In [automata game](https://github.com/riddles-are-us/zkwasm-automata), we have a `state.rs` file which defines and manages the state of the application, including the events.

First, in order to use the EventQueue Struct, we can import it as follows:
```
use zkwasm_rest_convention::EventQueue;
```

!!!note
    In your rust project dependency, you need to add the `zkwasm-rest-convention` crate to your `Cargo.toml` file:
    ```toml
    zkwasm-rest-convention = { git = "https://github.com/DelphinusLab/zkwasm-mini-rollup" }
    ```

Then, we simply add the EventQueue Struct to the state of the application:

```rs
pub struct State {
    // Other state fields...
    queue: EventQueue<Event>,
}
```

Note that in the `initialize()` function of the State implementation, we need to retrieve the queue field. This happens every time the application is initialized, either when the server restarts or after the rollup process is finalized:

```rs
pub fn initialize() {
    let mut state = STATE.0.borrow_mut();
    let kvpair = unsafe { &mut MERKLE_MAP };
    let mut data = kvpair.get(&[0, 0, 0, 0]);
    
    if !data.is_empty() {
        let mut data = data.iter_mut();
        state.supplier = *data.next().unwrap();
        state.queue = EventQueue::from_data(&mut data);
    }
}
```

And in the `process` function of the Transaction struct, we have:

```rs
impl Transaction {
    pub fn process(&self) -> i32 {
        match self.command {
            // Handle timetick transactions
            0 => {
                // Ensure only the admin/sequencer can trigger ticks
                unsafe { require(*pkey == *ADMIN_PUBKEY) };
                STATE.0.borrow_mut().queue.tick();
                0
            }
            // Other transaction types...
        }
    }
}
```

This is the place where we call the `tick()` method of the event queue to process the events. You may notice that we have a `require(*pkey == *ADMIN_PUBKEY)` statement, which is used to check if the transaction is sent by the admin account. This is to ensure that only the admin account which controlled by the server or sequencer can trigger the event processing, which is a good practice to prevent any malicious behavior that attempt to process the events without proper timetick transactions.

### Event Handling

Now we have the event queue in the state, we can implement the event handling methods in the `event.rs` file, this may be the most relevant and important part of the time-driven events.

In event.rs, we need to import several components:

```rs
use crate::player::AutomataPlayer;
use core::slice::IterMut;
use zkwasm_rest_abi::StorageData;
use zkwasm_rest_convention::EventHandler;
```

- We need `AutomataPlayer` because when handling the event, we need to update the state of the player.

- We need `core::slice::IterMut` to iterate over the event fields which we will define later, to get data of each field from database. This is because we serialize the event fields into a u64 array, and we need to deserialize them back to the original data structure through iterating the u64 array.

- We need `zkwasm_rest_abi::StorageData` to implement the necessary methods for serializing and deserializing a event into a u64 array.

- We need `zkwasm_rest_convention::EventHandler` to implement the methods for event handling interface.

Let's define an simple event as an example:

```rs
pub struct Event {
    pub owner: [u64; 2],
    pub object_index: usize,
    pub delta: usize,
}
```

Where the `owner` field is a 2-element array of u64 indicating the ID of a specific player, and the `object_index` indicates the index of the object in the game. The delta field indicates the time interval of the event, which is the time period after which the event will be triggered.

And here we implement the StorageData Trait for the Event:

```rs
impl StorageData for Event {
    fn to_data(&self, buf: &mut Vec<u64>) {
        buf.push(self.owner[0]);
        buf.push(self.owner[1]);
        buf.push(((self.object_index as u64) << 32) | self.delta as u64);
    }
    fn from_data(u64data: &mut IterMut<u64>) -> Event {
        let owner = [*u64data.next().unwrap(), *u64data.next().unwrap()];
        let f = *u64data.next().unwrap();
        Event {
            owner,
            object_index: (f >> 32) as usize,
            delta: (f & 0xffffffff) as usize,
        }
    }
}
```

In the `to_data()` method, we serialize the event fields into a u64 array, and in the `from_data()` method, we deserialize the u64 array back to the original data structure. 

Now we can Implement the EventHandler Trait for the Event:

```rs
impl EventHandler for Event {
    fn u64size() -> usize {
        3
    }
    fn get_delta(&self) -> usize {
        self.delta
    }
    fn progress(&mut self, d: usize) {
        self.delta -= d;
    }
    fn handle(&mut self, counter: u64) -> Option<Self> {
        ...
    }
}
```

The `u64size()` method returns the number of u64 elements in the event, in our implementation, we have 3 u64 elements in the event, so we return 3.

The `get_delta()` method returns the delta time of the event.

The `progress()` method is used to progress the event by the given delta time, which is used to reduce the delta time of the event. In our implementation of `tick()` method, we use `progress(1)` to progress the event by 1.

The `handle()` method is the most important one, which is used to handle the event, and maybe return the next event:

```rs
fn handle(&mut self, counter: u64) -> Option<Self> {
    let owner_id = self.owner;
    let object_index = self.object_index;
    let mut player = AutomataPlayer::get_from_pid(&owner_id).unwrap();
    let m = if player.data.energy == 0 {
        player.data.objects.get_mut(object_index).unwrap().halt();
        None
    } else {
        player.data.apply_object_card(object_index, counter)
    };
    let event = if let Some(delta) = m {
        if player.data.objects[object_index].get_modifier_index() == 0 {
            player.data.energy -= 1;
        }
        Some(Event {
            owner: owner_id,
            object_index,
            delta,
        })
    } else {
        None
    };
    player.store();
    event
}
```

Notice that in handle() method, we:

1. Get the player data through `owner_id`
2. Get the object data through `object_index` from the player data
3. Check if the player has enough energy:
    - If energy is 0, halt the object and return None
    - If energy > 0, call apply_object_card() to process the object's card effects
4. Process the result from apply_object_card():
    - If it returns Some(delta), create a new Event with:
        - Same owner_id
        - Same object_index
        - The returned delta as the new time interval
        - Additionally, if the object's modifier_index is 0, decrease player's energy by 1
    - If it returns None, no new event is created
5. Store the updated player state back to storage
6. Return the optional new Event


!!!info "Summary"
    In summary, in this method, we can:

    1. Retrieve the player data or object data we need to modify
    2. Modify the player data or object data based on the timetick. For example, every timetick (5 seconds), we can decrease the energy of the player by 1. In the above case, we have a condition to check if the player has enough energy, if not, we halt the object. if the player has enough energy, we can apply the object's card effects.
    3. Check the data after modification based on some conditions, if the conditions are met, we can create a new event.
    4. Return the new event if it is created, otherwise return None. Don't forget to store the updated player state back to storage.


