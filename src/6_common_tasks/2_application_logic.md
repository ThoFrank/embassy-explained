# Implementing Application Logic

Drivers talk to devices. Application logic decides what to do with them: read a sensor every second, raise an alarm if a value is too high, answer HTTP requests, forward messages between CAN and Ethernet, ...

The rule here is the same as for drivers: **put the logic into a library crate that doesn't know which hardware it runs on.**
The firmware binary then only creates the concrete drivers and hands them to the library.

## Logic as generic async functions
Embassy tasks (`#[embassy_executor::task]`) can't be generic, because every task is allocated statically and the compiler needs to know its exact size.
So the library doesn't define tasks. It defines generic `async fn`s, and the binary wraps them in small concrete tasks.

First define what the logic needs from the outside world, for example:

```rust,ignore
// in the `app` crate
#![no_std]

#[allow(async_fn_in_trait)]
pub trait TemperatureSource {
    type Error;
    async fn read_temperature(&mut self) -> Result<f32, Self::Error>;
}
```

The drivers from the [previous section](./1_writing_a_driver.md) can implement this trait (in the driver crate behind a feature, in the `app` crate, or via a newtype in the binary).
Then the logic itself:

```rust,ignore
use embassy_sync::blocking_mutex::raw::RawMutex;
use embassy_sync::channel::Sender;
use embassy_time::{Duration, Ticker};

pub enum Event {
    Temperature(f32),
    Overheated(f32),
    SensorError,
}

pub async fn monitor_temperature<S, M, const N: usize>(
    sensor: &mut S,
    events: Sender<'_, M, Event, N>,
    limit: f32,
) -> !
where
    S: TemperatureSource,
    M: RawMutex,
{
    let mut ticker = Ticker::every(Duration::from_secs(1));
    loop {
        let event = match sensor.read_temperature().await {
            Ok(t) if t > limit => Event::Overheated(t),
            Ok(t) => Event::Temperature(t),
            Err(_) => Event::SensorError,
        };
        events.send(event).await;
        ticker.next().await;
    }
}
```

The binary crate wires it together:

```rust,ignore
// in firmware-stm32f4
static EVENTS: Channel<CriticalSectionRawMutex, Event, 8> = Channel::new();

#[embassy_executor::task]
async fn monitor_task(mut sensor: TempSensor<I2c<'static, Async>>) {
    app::monitor_temperature(&mut sensor, EVENTS.sender(), 80.0).await
}

#[embassy_executor::task]
async fn alarm_task(led: Output<'static>) {
    app::handle_events(EVENTS.receiver(), led).await
}
```

The tasks are three lines each - all the interesting code lives in `app`.

## Connecting tasks
Tasks communicate through the primitives from [`embassy-sync`](https://docs.embassy.dev/embassy-sync/):

| You want to ...                                 | Use                     |
|-------------------------------------------------|-------------------------|
| send a stream of messages from A to B           | `Channel`               |
| tell B that something changed (latest value wins) | `Signal`              |
| let many tasks see the current value            | `Watch`                 |
| broadcast messages to many subscribers          | `PubSubChannel`         |
| share a resource (bus, buffer) between tasks    | `Mutex`                 |

Pass `Sender`s, `Receiver`s or references to these primitives into your library functions instead of using global `static`s inside the library.
That way the library doesn't decide how many instances exist or which mutex type is used, and a test can create its own channel.

The `RawMutex` type parameter decides how the primitive is protected:
- `CriticalSectionRawMutex` - safe everywhere, including between [different executors](../4_embassy_ecosystem.md#multiple-executors) and interrupts
- `ThreadModeRawMutex` - cheaper, but only for tasks on the thread mode executor
- `NoopRawMutex` - no protection, only within a single task or executor; handy in tests

Keeping the library generic over `M: RawMutex` lets the binary choose.

## Takeaways
We now have defined a way which allows us to separate:
- Driver code - code specific to a given sensor
- Sensor data provider - Specific sensor data (e.g. temperature) can be provided via any device on any bus
- business logic - takes input data from any source and does logic manipulation

=> This aligns very well with rust ownership model.

## Checklist
- [ ] logic lives in a `#![no_std]` library crate without HAL dependency
- [ ] logic is written as generic `async fn`s; tasks are thin wrappers in the binary
- [ ] dependencies (sensors, channels, sockets) are passed in, not created inside the library
- [ ] code is generic over `embedded-io-async` traits
- [ ] logic is tested on the host, ideally also as a whole in a simulation binary
