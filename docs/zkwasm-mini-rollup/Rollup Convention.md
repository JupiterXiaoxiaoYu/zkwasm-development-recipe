## Convention (/convention)
The zkWasm Convention Library provides essential traits and implementations for building zkWasm applications with standardized state management, event handling, and settlement processing. This library serves as the foundation for creating consistent and reliable zkWasm applications.

### Core Components

#### CommonState Trait
A fundamental trait that defines the interface for managing application state.

```rust
pub trait CommonState: Serialize + StorageData + Sized {
    type PlayerData: StorageData + Default + Serialize;
    // ... implementation methods
    // Global State Management
    fn get_global<'a>() -> Ref<'a, Self>;
    fn get_global_mut<'a>() -> RefMut<'a, Self>;
    fn snapshot() -> String
    // Player State Management
    fn get_state(pkey: Vec<u64>) -> String 
    // Rand Seed
    fn rand_seed() -> u64
    // Rollup State Management
    fn preempt() -> bool
    fn store(&self)
    fn initialize()
}
```

Key Features    

- Global State Management: Access and modify global application state
- Player State Handling: Manage individual player states
- State Serialization: Convert states to/from JSON format
- State Persistence: Store and retrieve state from merkle tree storage
- Rollup State Management: Handle rollup-specific state operations

By implementing this trait, you can ensure that your application adheres to a consistent structure and can be easily integrated with the zkWasm Mini Rollup.

#### Settlement

The code below manages withdrawal information and settlement processing, where `append_settlement` is used to add a new withdrawal to the list, and `flush_settlement` process and clear all pending settlements, it returns the transaction data to be sent to the blockchain for settlement.

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
            s.flush(&mut bytes);
        }
        sinfo.0 = vec![];
        bytes
    }
}
```

#### Event 

Event are used to handle time-based events, such as timers, scheduled actions, or periodic updates, as well as their side effects.

There are two things important in the event handling:

1. The EventHandler Trait defines the interface for handling time-based events

    ```rust
    pub trait EventHandler: Clone + StorageData {
        fn get_delta(&self) -> usize; // get the delta time of the event
        fn progress(&mut self, d: usize); // progress the event by the given delta time
        fn handle(&mut self, counter: u64) -> Option<Self>; // handle the event and maybe return the next event
        fn u64size() -> usize; // get the size (fields) of the event type in u64
    }
    ```

2. The EventQueue Struct: implements a differential time queue (DTQ) for efficient event scheduling and processing.

    ```rust
    pub struct EventQueue<T: EventHandler + Sized> {
        pub counter: u64, // the counter of the event queue represents the total number of timeticks.
        pub list: std::collections::LinkedList<T>, // the event queue
    }
    ```

In an EventQueue which implements the EventHandler Trait, we have serveral methods to handle the events:

##### dump
```rust
fn dump(&self, counter: u64) {
    zkwasm_rust_sdk::dbg!("dump queue: {}, ", counter);
    for m in self.list.iter() {
        let delta = m.get_delta();
        zkwasm_rust_sdk::dbg!(" {}", delta);
    }
    zkwasm_rust_sdk::dbg!("\n");
}
```
The above code is used to dump the event queue, it will print the event queue and the delta of each event. This is useful for debugging and understanding the event queue.

##### insert
```rust
/// Insert a event into the event queue
/// The event queue is a differential time queue (DTQ) and the event will
/// be inserted into its proper position based on its delta time
pub fn insert(&mut self, node: E) {
    let mut event = node.clone();
    let mut cursor = self.list.cursor_front_mut();
    while cursor.current().is_some()
        && cursor.current().as_ref().unwrap().get_delta() <= event.get_delta()
    {
        event.progress(cursor.current().as_ref().unwrap().get_delta());
        cursor.move_next();
    }
    match cursor.current() {
        Some(t) => {
            t.progress(event.get_delta());
        }
        None => (),
    };

    cursor.insert_before(event);
}
```

The above code is used to insert a event into the event queue, the event will be inserted into its proper position based on its delta time.

!!!info 
    The event queue is a **differential time queue (DTQ)**, which means in the event queue, the events are sorted by their delta time. For example:

    1. Initial queue state (numbers represent delta time):
      ```[2] -> [3] -> [4]```
    2. When inserting an event with delta=5:
      ```[2] -> [3] -> [4] -> [5]```
    3. When inserting an event with delta=1:
      ```[1] -> [2] -> [3] -> [4] -> [5]```

    Note: The deltas are adjusted during insertion to maintain relative time differences.

    Key characteristics of DTQ:

    - Each node stores the time difference from its previous node
    - Total time to an event = sum of deltas from start to that event
    - Efficient for time-based event scheduling
    - Maintains sorted order automatically
    - Updates are O(n) in worst case but typically much faster in practice

##### tick

The `tick()` method is the core processing function of the event queue, responsible for advancing time and handling events. Each call increments the counter by 1 and processes all due events. 

The processing flow consists of four main steps:

1. Retrieve and Process Historical Events

    ```rust
    /// Perform tick:
    /// 1. get old entries and peform event handlers on each event
    /// 2. insert new generated events into the event queue
    /// 3. handle all events whose counter are zero
    /// 4. insert new generated envets into the event queue
    pub fn tick(&mut self) {
        let counter = self.counter;
        self.dump(counter);
        let mut entries_data = self.get_old_entries(counter);
        let entries_nb = entries_data.len() / E::u64size();
        let mut dataiter = entries_data.iter_mut();
        let mut entries = Vec::with_capacity(entries_nb);
        ....// handle the events
        self.counter += 1;
    }
    ```

    The above code get the historical events data by `get_old_entries`, and calculate the number of the events by using divide the length of the data by the size of the event type in u64, for example:

    ```rust
    struct MyEvent {
        field1: u64,  // takes 1 u64
        field2: u64,  // takes 1 u64
    }

    impl EventHandler for MyEvent {
        fn u64size() -> usize {
            2  // Each MyEvent takes 2 u64s to store
        }
        // ... other implementations
    }

    // If entries_data contains [1, 2, 3, 4, 5, 6] (6 u64 values)
    // And each MyEvent takes 2 u64s
    // Then entries_nb = 6 / 2 = 3 events
    ```

    After getting the number of the events, a entries vector is created to store the historical events.

    ```rust
    let mut entries = Vec::with_capacity(entries_nb);
    for _ in 0..entries_nb {
        entries.push(E::from_data(&mut dataiter));
    }
    ```

    Then, the code will iterate over the historical or existing events and call the `handle` method of each event: 
    ```rust
    for mut e in entries {
        let m = e.handle(counter);
        if let Some(event) = m {
            self.insert(event);
        }
    }
    ```

2. Handle the events in the event queue

    ```rust
    while let Some(head) = self.list.front_mut() {
        if head.get_delta() == 0 {
            let m = head.handle(counter);
            self.list.pop_front();
            if let Some(event) = m {
                self.insert(event);
            }
        } else {
            head.progress(1);
            break;
        }
    }
    ```

    The above code is straightforward and intuitive, it will iterate over the events in the event queue, if the delta of the event is 0, it will call the `handle` method of the event, probably insert a new event into the event queue (if there is a new event generated), and then pop the event from the event queue. If the delta of the event is not 0, it will progress the event by 1 (the delta of the event will be reduced by 1).


You can also notice that we have other implementations of the `EventHandler` trait, such as: 

```rust 
impl<T: EventHandler + Sized> StorageData for EventQueue<T> {
    fn to_data(&self, buf: &mut Vec<u64>)
    fn from_data(u64data: &mut IterMut<u64>) -> Self
}
```

Where `to_data` is used to convert the event queue to a u64 array and store it in the storage, and `from_data` is used to convert the u64 array to an event queue from the storage.

And: 
```rust
impl<T: EventHandler + Sized> EventQueue<T>{
    fn get_old_entries(&self, counter: u64) -> Vec<u64> 
    fn set_entries(&self, entries: &Vec<u64>, counter: u64)
    pub fn store(&mut self)
}
```

Where `get_old_entries` is used to get the historical events data by the given counter, `set_entries` is used to set the existing events data by the given counter, and `store` is used to store the event queue to the storage, which is a merkle key-value pair storage: 

```rust
// In impl<T: EventHandler + Sized> EventQueue<T>{..}
fn set_entries(&self, entries: &Vec<u64>, counter: u64) {
    let kvpair = unsafe { &mut MERKLE_MAP };
    kvpair.set(
        &[counter & 0xeffffff, EVENTS_LEAF_INDEX, 0, EVENTS_LEAF_INDEX],
        entries.as_slice(),
    );
    zkwasm_rust_sdk::dbg!("store {} entries at counter {}", { entries.len() }, counter);
}
```

The above code can store all the events that shall be processed in a same specific counter to the storage. This is particularly useful to save the memory space and improve the performance.

!!!info
    We will cover how to leverage the event queue and time-based events in [Implementing Time-Driven Events](../development-guide/Implementing Time-Driven Events.md) chapter.
