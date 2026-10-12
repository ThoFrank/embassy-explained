# Bus Overview

An MCU rarely works alone. Sensors, memories, displays, other controllers and whole networks are connected to it through buses.
The bus you use has a large impact on how you structure your software: who owns the peripheral, how many tasks are involved and where data flows.

This chapter sorts the most common buses by how much structure they need in your firmware:

| Complexity | Bus        | Typical use                               | Software structure                                       |
|------------|------------|-------------------------------------------|----------------------------------------------------------|
| simple     | UART, SPI, I2C | sensors, memories, displays, modems   | every driver owns a handle to the (shared) bus           |
| medium     | CAN        | communication between controllers (cars, machines) | one central broker reads frames and distributes them |
| complex    | Ethernet   | IP networks, TCP/UDP, HTTP, MQTT          | background tasks run the stack, every task owns a socket |

## Simple buses: UART, SPI, I2C
These buses are driven by the MCU. Nothing happens on the bus unless your code starts a transfer (I2C and SPI), and the amount of data is small and predictable.
A transfer is just an `async fn` call:

```rust,ignore
let mut buf = [0u8; 2];
i2c.write_read(SENSOR_ADDR, &[REG_TEMP], &mut buf).await?;
```

- **UART** - point-to-point, one device per bus. The driver owns the UART (or its `tx`/`rx` halves after `split()`) and talks to it through [`embedded-io-async`](https://docs.rs/embedded-io-async).
- **I2C** - two wires, up to ~1 MHz, every device has an address. Many devices share the same two wires.
- **SPI** - four wires, tens of MHz, every device has its own chip select (CS) pin. Many devices share SCK/MOSI/MISO.

### Sharing the bus
I2C and SPI are usually shared by several devices. Since the MCU is the one starting every transfer, sharing is easy: put the bus into a `Mutex` and hand every driver its own device handle from [`embassy-embedded-hal`](https://docs.embassy.dev/embassy-embedded-hal/).

```rust,ignore
use embassy_embedded_hal::shared_bus::asynch::spi::SpiDevice;
use embassy_sync::blocking_mutex::raw::CriticalSectionRawMutex;
use embassy_sync::mutex::Mutex;
use static_cell::StaticCell;

type SpiBus = Mutex<CriticalSectionRawMutex, Spi<'static, Async>>;
static SPI_BUS: StaticCell<SpiBus> = StaticCell::new();

// in main
let spi = Spi::new(p.SPI1, p.PA5, p.PA7, p.PA6, p.DMA2_CH3, p.DMA2_CH0, spi_config);
let spi_bus = SPI_BUS.init(Mutex::new(spi));

// one device per chip select pin
let flash = SpiDevice::new(spi_bus, Output::new(p.PA4, Level::High, Speed::VeryHigh));
let display = SpiDevice::new(spi_bus, Output::new(p.PB0, Level::High, Speed::VeryHigh));

spawner.spawn(storage_task(Flash::new(flash))).unwrap();
spawner.spawn(display_task(Display::new(display))).unwrap();
```

`SpiDevice` locks the mutex, pulls CS low, runs the transaction and releases both again. `I2cDevice` works the same way, just without a CS pin.
Both implement the normal `embedded-hal-async` traits, so the driver doesn't know (or care) that the bus is shared (see [Writing a Device Driver](./6_common_tasks/1_writing_a_driver.md#sharing-a-bus)).

The result fits the Rust ownership model nicely:
- **Every driver owns its device handle.** No other code needs to know the driver exists.
- **The driver decides how to talk to its device.** It can poll every second, read a burst on demand, or wait on an interrupt pin - the bus doesn't impose a structure.
- **No central coordinator.** Adding a device means adding a `SpiDevice` and a task, nothing else changes.

> **Tip:** The mutex is held for the whole transaction. A device doing long transfers (e.g. a display flushing a frame buffer) delays all other devices on the bus.
> If one device needs guaranteed latency, give it its own bus.

## Medium: CAN
CAN is different in two ways:
- **Every node can send at any time.** Frames arrive whenever another controller decides to send them, not when your code asks for them.
- **It is a broadcast bus.** Every frame is received by every node. There are no device handles; frames are identified by their *ID*, and all of them end up in the same receive FIFO of your CAN peripheral.

Because of this, the SPI approach of "every driver owns a handle" doesn't work for receiving. Whoever reads from the FIFO takes the frame away from everyone else.
If two tasks read from the same `CanRx`, each one gets a random subset of the frames.

On top of that, a busy CAN bus carries a lot of frames: at 1 Mbit/s a classic frame arrives roughly every 100 µs, with CAN FD even more often.
Waking up a task (or even several tasks) for every single frame wastes a lot of CPU time on context switches.

### Central message broker
The common solution is a single broker task which owns the receive side of the peripheral:

```text
                         ┌────────────────┐
  CAN peripheral ──────▶ │  broker task   │
  (interrupt fills       │ every 1 ms:    │
   a buffer)             │  drain buffer, │
                         │  route by ID   │
                         └───┬────┬────┬──┘
                             │    │    │    Channels
                             ▼    ▼    ▼
                        motor  battery  diagnostics
                        task   task     task
```

1. The interrupt handler of the driver moves incoming frames from the small hardware FIFO into a larger RAM buffer (e.g. a buffered CAN driver of the HAL).
2. The broker wakes up **cyclically** (e.g. with a `Ticker` every millisecond), drains *all* frames received since the last cycle and routes each one by its ID.
   One wakeup handles a whole batch of frames instead of one wakeup per frame.
3. The components only receive the frames they are interested in, through their own `Channel`.

```rust,ignore
use embassy_sync::blocking_mutex::raw::RawMutex;
use embassy_sync::channel::Sender;
use embassy_time::{Duration, Ticker};
use embedded_can::{Frame, Id};

pub trait CanReceiver {
    type Frame: Frame;
    /// returns the next buffered frame without waiting
    fn try_read(&mut self) -> Option<Self::Frame>;
}

pub async fn can_broker<R, M, const N: usize>(
    rx: &mut R,
    motor: Sender<'_, M, R::Frame, N>,
    battery: Sender<'_, M, R::Frame, N>,
) -> !
where
    R: CanReceiver,
    M: RawMutex,
{
    let mut ticker = Ticker::every(Duration::from_millis(1));
    loop {
        ticker.next().await;
        while let Some(frame) = rx.try_read() {
            let target = match frame.id() {
                Id::Standard(id) if (0x100..0x110).contains(&id.as_raw()) => &motor,
                Id::Standard(id) if (0x200..0x210).contains(&id.as_raw()) => &battery,
                _ => continue, // nobody is interested
            };
            // never block the broker because one component is slow
            if target.try_send(frame).is_err() {
                // count / log the dropped frame
            }
        }
    }
}
```

Some notes on this design:
- **Size the buffers for one cycle.** The RAM buffer must hold all frames that can arrive within one broker period, otherwise frames are lost.
  A shorter period means less buffer, more wakeups - pick it based on your bus load and latency requirements.
- **Use the hardware filters.** Most CAN peripherals can drop frames with uninteresting IDs in hardware, so they never cost CPU time.
- **Use `try_send`.** A component that doesn't keep up should lose its own frames, not stall the routing for everyone.
- **Sending is simpler.** Transmitting doesn't have the "who takes the frame" problem. Components can share the `CanTx` behind a `Mutex`, or send their frames to a single TX task through a `Channel`.
- **Keep the routing table in the library.** The broker above only depends on `embedded-can` and a small trait, so it can be tested on the host or run on Linux with `socketcan` (see [Common Tasks](./6_common_tasks.md)).

## Complex: Ethernet
Ethernet (and WiFi, which looks the same to the software) is the most complex bus:
- frames arrive at any time and at high data rates
- the protocol stack has work to do even if your application is idle: answering ARP and ping, renewing DHCP leases, retransmitting TCP segments, handling timeouts
- data is not sent in individual frames but as streams (TCP) or datagrams (UDP) which the stack splits into and assembles from frames

No application task should deal with all of this. That's why [`embassy-net`](./4_embassy_ecosystem.md#embassy-net) needs **background tasks**:
- the stack's `Runner` - runs the TCP/IP stack
- depending on the hardware, a runner for the driver itself (e.g. `cyw43` for WiFi or an SPI Ethernet chip)

```rust,ignore
#[embassy_executor::task]
async fn net_task(mut runner: embassy_net::Runner<'static, Ethernet<'static, ETH, GenericPhy>>) -> ! {
    runner.run().await
}

// in main
static RESOURCES: StaticCell<StackResources<4>> = StaticCell::new();
let (stack, runner) = embassy_net::new(device, config, RESOURCES.init(StackResources::new()), seed);
spawner.spawn(net_task(runner)).unwrap();

spawner.spawn(http_server_task(stack)).unwrap();
spawner.spawn(mqtt_task(stack)).unwrap();
```

### Sockets bring back simple ownership
Once the stack is running, things become simple again. Each task creates its **own socket** with its **own buffers** and owns it, just like a driver owns its `SpiDevice`:

```rust,ignore
#[embassy_executor::task]
async fn http_server_task(stack: Stack<'static>) -> ! {
    let mut rx_buffer = [0; 1024];
    let mut tx_buffer = [0; 1024];

    loop {
        let mut socket = TcpSocket::new(stack, &mut rx_buffer, &mut tx_buffer);
        if socket.accept(80).await.is_err() {
            continue;
        }
        // `socket` implements embedded-io-async Read + Write
        app::serve_http(&mut socket).await.ok();
    }
}
```

The stack takes care of demultiplexing: incoming packets are delivered to the socket with the matching port, so there is no need for a broker like with CAN.
Compared to the other buses:
- **Ownership is per socket.** The HTTP server and the MQTT client don't know about each other, and a slow task only fills up its own buffers.
- **Sockets implement [`embedded-io-async`](https://docs.rs/embedded-io-async).** Your application logic can be written against `Read + Write` and tested on the host with a plain byte buffer or a `std` TCP stream.
- **Resources are static.** `StackResources<N>` limits how many sockets can exist at once, and every socket brings its own RX/TX buffers. Plan your RAM accordingly.

> **Tip:** If the `net_task` is starved (by a task that doesn't `.await`), the whole network stops - including TCP retransmissions and DHCP renewal.
> Run long computations on a lower priority executor or split them up with `yield_now().await`.

## Summary
The more autonomous the traffic on a bus, the more infrastructure you need:
- **SPI, I2C, UART** - the MCU drives every transfer. Share the bus with `SpiDevice`/`I2cDevice` and let every driver own its handle.
- **CAN** - frames arrive at any time and are broadcast to all. A central broker reads them in batches and routes them to the components.
- **Ethernet** - the protocol stack runs in background tasks. Sockets give every task its own, independent connection again.
