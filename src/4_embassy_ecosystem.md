# Embassy Ecosystem

Embassy is an umbrella project for multiple pieces. Its name comes from EMBedded ASYnc.
All crates live in a single [repository](https://github.com/embassy-rs/embassy) and are designed to work together, but most of them can also be used on their own.
This chapter introduces the crates you will run into first.

## embassy-executor
In Rust an `async fn` does nothing by itself. It returns a `Future` which has to be polled by an *executor* until it completes.
On a desktop this job is done by a runtime like tokio. On a microcontroller [`embassy-executor`](https://docs.embassy.dev/embassy-executor/) takes this role.

Some properties make it a good fit for MCUs:
- **No heap required** - every task is allocated statically at compile time. You decide how many instances of a task may exist with `#[embassy_executor::task(pool_size = N)]`.
- **Low power by default** - when no task has work to do, the executor puts the core to sleep (e.g. `WFE` on Cortex-M) until an interrupt wakes it up again. There is no busy polling.
- **Cooperative scheduling** - a task runs until it hits an `.await` which cannot make progress yet. Then the executor switches to the next task that is ready.

A minimal program looks like this:

```rust,ignore
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::Timer;

#[embassy_executor::task]
async fn blink(mut led: Output<'static>) {
    loop {
        led.toggle();
        Timer::after_millis(500).await;
    }
}

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());
    let led = Output::new(p.PA5, Level::Low, Speed::Low);

    spawner.spawn(blink(led)).unwrap();
}
```

The `#[embassy_executor::main]` macro creates the executor, starts it and runs `main` as the first task. Further tasks are started with the `Spawner`.

### Multiple executors
Because scheduling is cooperative, a task that never awaits (or does a long computation between two `.await`s) blocks every other task on the same executor.
If some work needs a guaranteed reaction time, you can run more than one executor:

- **Thread mode executor** - the regular executor created by `#[embassy_executor::main]`. It runs at the lowest priority.
- **[`InterruptExecutor`](https://docs.embassy.dev/embassy-executor/git/cortex-m/struct.InterruptExecutor.html)** - an executor which runs inside an interrupt handler. You pick an otherwise unused interrupt and give it a priority. When a task on this executor becomes ready, the interrupt is pended and preempts whatever lower priority code is currently running.

This way you can for example run a motor control loop on a high priority executor while networking and logging run on the low priority thread mode executor.
Tasks on different executors communicate through the primitives in `embassy-sync` (channels, signals, mutexes), which are safe to use across priority levels.

> **Tip:** Check out the [From Zero to Async in Embedded Rust - The Rusty Bits](https://www.youtube.com/watch?v=wni5h5vIPhU) video for a deep dive into concurrency in microcontrollers and how async rust works under the hood.

## embassy-time
Almost every program needs to wait for something. [`embassy-time`](https://docs.embassy.dev/embassy-time/) provides `Timer`, `Instant`, `Duration` and `Ticker` together with async delays like `Timer::after_millis(100).await`.
The time source is a hardware timer provided by the HAL. For `embassy-stm32` you select it with a cargo feature such as `time-driver-any`.

## embassy-sync
[`embassy-sync`](https://docs.embassy.dev/embassy-sync/) contains synchronization primitives which work without an operating system:
- `Channel` - a bounded queue to send messages between tasks
- `Signal` - notifies a task that a value changed, only keeps the latest value
- `Mutex` - an async mutex for sharing a resource (e.g. a SPI bus) between tasks
- `Watch` / `PubSubChannel` - one producer, many consumers

## embassy-stm32
[`embassy-stm32`](https://docs.embassy.dev/embassy-stm32/) is a HAL (see [Software Layers](./3_layers_of_software.md#hardware-abstraction-layer-crates-hal)) for the whole STM32 family.
Instead of having one crate per chip family, a single crate covers all of them. You select your exact chip with a cargo feature:

```toml
[dependencies]
embassy-stm32 = { version = "*", features = ["stm32f407vg", "time-driver-any", "exti"] }
```

The register definitions come from [`stm32-metapac`](https://docs.rs/stm32-metapac), a PAC that is generated from the [stm32-data](https://github.com/embassy-rs/stm32-data) project.

`embassy_stm32::init` configures the clocks and returns a `Peripherals` struct. It contains exactly one instance of every peripheral (`p.PA5`, `p.SPI1`, `p.USART2`, ...). Because each peripheral exists only once, the compiler prevents two drivers from using the same pin or peripheral by accident.

Most importantly, the drivers implement the generic traits from the ecosystem:
- [`embedded-hal`](https://docs.rs/embedded-hal) 1.0 (and 0.2) - blocking GPIO, SPI, I2C, delays
- [`embedded-hal-async`](https://docs.rs/embedded-hal-async) - the async versions of the above
- [`embedded-io-async`](https://docs.rs/embedded-io-async) - byte streams such as UART

Thanks to these traits, a driver crate for a sensor only has to be written once and works on every HAL that implements them.
Async drivers use DMA and interrupts under the hood, so while a transfer is running the CPU is free to run other tasks or sleep.

Embassy has sibling HALs with the same design for other vendors, for example [`embassy-nrf`](https://docs.embassy.dev/embassy-nrf/) (Nordic) and [`embassy-rp`](https://docs.embassy.dev/embassy-rp/) (Raspberry Pi RP2040/RP235x).
Some vendor HALs outside the Embassy project, like [`esp-hal`](https://docs.rs/esp-hal), also work with the Embassy executor.

## embassy-net
Networking is hard. A TCP/IP stack has to answer ARP requests, retransmit lost packets, renew DHCP leases and handle timeouts, even while your application is not sending or receiving anything.
[`embassy-net`](https://docs.embassy.dev/embassy-net/) wraps the [`smoltcp`](https://docs.rs/smoltcp)(will be replaced soon by [`xarxa`](https://github.com/embassy-rs/xarxa)) stack and gives it an async interface.

The important part to understand is that the stack needs its own background task which keeps it running:

```rust,ignore
#[embassy_executor::task]
async fn net_task(mut runner: embassy_net::Runner<'static, MyDriver>) -> ! {
    runner.run().await
}

// in main
let (stack, runner) = embassy_net::new(driver, config, resources, seed);
spawner.spawn(net_task(runner)).unwrap();

// wait until we have an IP address (e.g. from DHCP)
stack.wait_config_up().await;
```

If this task is not spawned, or is starved by other tasks that never await, nothing will happen on the network.
After that you can create `TcpSocket`s, `UdpSocket`s or use DNS on the `stack` from any task.

`embassy-net` itself is hardware independent. The actual network interface is provided by a driver implementing the [`embassy-net-driver`](https://docs.embassy.dev/embassy-net-driver/) trait, for example:
- the built-in Ethernet MAC of an STM32 (`embassy_stm32::eth`)
- [`cyw43`](https://docs.embassy.dev/cyw43/) - the WiFi chip on the Raspberry Pi Pico W
- [`embassy-net-wiznet`](https://docs.embassy.dev/embassy-net-wiznet/) - SPI Ethernet chips like the W5500

Note that some of these drivers need a background task of their own in addition to the stack's runner.
Memory for sockets is reserved statically with `StackResources<N>`, so you have to decide up front how many sockets can be open at once.

## embassy-boot
When devices are deployed in the field you want to update their firmware without a debugger attached.
[`embassy-boot`](https://docs.embassy.dev/embassy-boot/) is a bootloader that makes this possible. Chip specific integrations exist as `embassy-boot-stm32`, `embassy-boot-nrf` and `embassy-boot-rp`.

It splits the flash into four partitions:
- **BOOTLOADER** - the bootloader itself
- **ACTIVE** - the firmware that is currently running
- **DFU** - where the application downloads the new firmware to (must be at least one page larger than ACTIVE)
- **STATE** - remembers the progress of an update

An update works like this:
1. The running application receives the new image (over network, USB, UART, ...) and writes it to the DFU partition using the `FirmwareUpdater`.
2. It marks the update as ready and resets.
3. The bootloader swaps ACTIVE and DFU page by page. This is power fail safe: if power is lost, the swap continues at the next boot.
4. The new firmware has to confirm that it works by calling `mark_booted`. If it does not (e.g. because it crashes), the bootloader swaps back to the old firmware on the next reset.

Optionally images can be signed with ed25519 so that only firmware from you is accepted.

### When you might need a custom bootloader
`embassy-boot` is a good starting point, but it does not fit every project. You may have to write your own, or extend it, if:
- the device already ships with a different bootloader and the update format must stay compatible
- the new firmware is fetched by the bootloader itself (e.g. from an SD card or over CAN) instead of by the application
- you need encrypted images, secure boot with hardware key storage, or a specific certification
- your flash layout does not allow for a DFU partition of the required size, or you want to use the dual bank feature of your MCU instead of swapping

Since the update logic is split into reusable building blocks (`embassy-boot` for the logic, `embedded-storage` traits for the flash), a custom bootloader can often still reuse large parts of it.
