# Writing a Device Driver

A driver is the code that talks to a device connected to your MCU: a sensor on an I2C bus, a display on SPI, ...

The golden rule: **a driver only depends on generic traits, never on a specific HAL.**

If the driver takes an `embassy_stm32::i2c::I2c` it can only ever run on an STM32 with Embassy.
If it takes "anything that implements `embedded_hal_async::i2c::I2c`" it runs on every MCU with a HAL implementing that trait, on Linux, and in your unit tests.

> **Tip:** Before writing a driver yourself, search [crates.io](https://crates.io) for the name of your device (e.g. the chip name like `tmp117` or `bme280`).
> Many common sensors, displays and controllers already have a driver built on `embedded-hal`.
> Prefer one that supports `embedded-hal-async`, so it fits nicely into Embassy.

## A simple I2C driver
Let's start with the common case where a standard trait already exists.

```rust,ignore
#![no_std]

use embedded_hal_async::i2c::I2c;

const ADDRESS: u8 = 0x48;
const REG_TEMP: u8 = 0x00;

pub struct TempSensor<I> {
    i2c: I,
}

impl<I: I2c> TempSensor<I> {
    pub fn new(i2c: I) -> Self {
        Self { i2c }
    }

    /// Returns the temperature in degree Celsius.
    pub async fn read_temperature(&mut self) -> Result<f32, I::Error> {
        let mut buf = [0u8; 2];
        self.i2c.write_read(ADDRESS, &[REG_TEMP], &mut buf).await?;
        let raw = i16::from_be_bytes(buf) >> 4;
        Ok(raw as f32 * 0.0625)
    }

    /// Gives the bus back, e.g. to use it for something else.
    pub fn release(self) -> I {
        self.i2c
    }
}
```

Things to note:
- The crate is `#![no_std]` so it can be used on an MCU. It still works on Linux, since `std` is a superset.
- There is no dependency on a HAL or on `embassy-executor`. The driver is just a bunch of `async fn`s; whoever calls them decides how they are executed.
- Errors of the bus are passed through (`I::Error`), so the caller can decide what to do.

### Sharing a bus
Often several devices sit on the same I2C or SPI bus. Don't build bus sharing into your driver.
Instead let the application wrap the bus with [`embassy-embedded-hal`](https://docs.embassy.dev/embassy-embedded-hal/)'s `I2cDevice` / `SpiDevice` (or [`embedded-hal-bus`](https://docs.rs/embedded-hal-bus) for blocking code). These wrappers implement the same traits, so the driver does not notice the difference.

## Testing a driver
Since the driver only needs a trait, tests can provide a fake implementation and run on your PC with `cargo test`.
For standard traits like I2C or SPI you don't even have to write the fake yourself: [`embedded-hal-mock`](https://docs.rs/embedded-hal-mock) provides mocks which check that the expected bus transactions happen (enable its `embedded-hal-async` feature for the async traits):

```rust,ignore
#[cfg(test)]
mod tests {
    use super::*;
    use embassy_futures::block_on;
    use embedded_hal_mock::eh1::i2c::{Mock, Transaction};

    #[test]
    fn reads_temperature() {
        let expectations = [Transaction::write_read(ADDRESS, vec![REG_TEMP], vec![0x19, 0x00])];
        let mut i2c = Mock::new(&expectations);
        let mut sensor = TempSensor::new(i2c.clone());

        let temp = block_on(sensor.read_temperature()).unwrap();
        assert_eq!(temp, 25.0);

        i2c.done(); // fails if not all expected transactions happened
    }
}
```

## Checklist
- [ ] `#![no_std]`, no HAL and no executor dependency
- [ ] generic over traits from `embedded-hal(-async)`, `embedded-io(-async)` or a small trait of your own
- [ ] bus errors are passed to the caller instead of panicking
- [ ] optional features (e.g. `defmt` logging) are behind cargo features
- [ ] unit tests run on the host with mocks

