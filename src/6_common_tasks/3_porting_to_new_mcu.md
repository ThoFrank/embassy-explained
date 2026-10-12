# Porting to a new MCU

Sooner or later the hardware changes: the chip you use is out of stock, you need more RAM, or a new board revision uses a different MCU.

If drivers and application logic were written as described in the previous sections, porting is mostly **configuration and wiring**.
Only the binary crate has to change. If you find yourself editing a driver or the `app` crate during a port, that is a sign that hardware specific code leaked into it.

## What changes and what doesn't

| Part                                   | Changes when porting? |
|----------------------------------------|-----------------------|
| Drivers (generic over traits)          | No                    |
| Application logic (`app` crate)        | No                    |
| Rust target, probe/runner config       | Maybe                 |
| HAL crate and its features             | Yes                   |
| Memory layout (`memory.x`)             | Yes                   |
| Clock configuration                    | Yes                   |
| Pin mapping, peripheral instances, interrupts | Yes            |
| Adapters for your own traits           | Maybe                 |
| Bootloader partitions                  | Maybe                 |

## Step by step

### 1. Target and tooling
Check which core the new MCU has and choose the matching [target](../3_layers_of_software.md#machine-code):

```sh
rustup target add thumbv8m.main-none-eabihf   # e.g. Cortex-M33 with FPU
```

Update `.cargo/config.toml` with the new target and the chip name for your flashing tool:

```toml
[build]
target = "thumbv8m.main-none-eabihf"

[target.'cfg(all(target_arch = "arm", target_os = "none"))']
runner = "probe-rs run --chip STM32H563ZITx"
```

### 2. HAL
Within the same vendor family this is often just a feature change:

```toml
embassy-stm32 = { version = "*", features = ["stm32h563zi", "memory-x", "time-driver-any", "exti"] }
```

When switching vendors (e.g. STM32 to nRF) the HAL crate changes, e.g. to `embassy-nrf`.

### 3. Memory layout
Flash and RAM sizes differ between chips. `embassy-stm32` can generate `memory.x` for you with the `memory-x` feature.
If you maintain your own `memory.x` (e.g. because of a bootloader), update the addresses and sizes from the datasheet.

### 4. Clocks
`embassy_stm32::init` takes a `Config` whose `rcc` field describes the clock tree. Clock trees differ a lot between families, so this usually has to be rewritten.
Check that peripherals which depend on exact frequencies (CAN bit timing, UART baud rate, USB's 48 MHz) still get what they need.

### 5. Pins, peripherals and interrupts
Map every signal to the new pins and peripheral instances according to the schematic, and bind the interrupts the drivers need with `bind_interrupts!`.
Peripheral names, DMA channels and pin alternate functions differ between chips. The compiler helps here: if a pin can't be used for a peripheral, the code does not compile.

### 6. Adapters
Sometimes the new MCU has a different *kind* of peripheral. A typical example on STM32: older chips like the F4 have a bxCAN controller, newer ones like G4 or H5 have FDCAN, and `embassy-stm32` provides different drivers for them.
Because your CAN drivers only depend on your `CanBus` trait, you just write a new [adapter](./1_writing_a_driver.md#implementing-the-trait-for-real-hardware) for the new peripheral. Nothing else changes.

### 7. Resources and timing
Check the assumptions made for the old chip:
- Is there enough RAM for buffers, `StackResources` and task pools?
- Do flash page sizes still match bootloader partitions and any flash based storage?
- Is the new chip fast enough (or now so fast that timeouts need adjusting)?

## Verify
1. Run the host tests of drivers and `app` - they must still pass without changes, since they didn't change.
2. Build and flash the new firmware; check clocks and pin mapping first (blink an LED, print over `defmt`).
3. Run your integration tests on the real hardware.
