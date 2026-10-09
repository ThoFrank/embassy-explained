# Software Layers

## Machine Code
To be able to run code on a microcontroller we need a compiler to generate machine code compatible with the CPU type. The rust compiler supports a [large number of so called targets](https://doc.rust-lang.org/beta/rustc/platform-support.html).
In our case this means we need to choose the correct target for the processor in our MCU. For a Cortex-M4F this would for example be [thumbv7em-none-eabihf](https://doc.rust-lang.org/rustc/platform-support/thumbv7em-none-eabi.html).

## Microarchitecture Crate
This class of crates contains code which defines the memory layout, initializes static variables before your program is called and makes sure the cpu boots up correctly.
Usually there is no need to interface with this crate directly as it will be built upon by the following abstraction layers. However in the case of the [`cortex-m-rt`](https://docs.rs/cortex-m-rt/latest/cortex_m_rt/) crate you can and often times have to hook into the memory layouting by creating a custom `memory.x` file.

## Peripheral Access Crates (PAC)
Referral access crates are usually auto-generated. They contain code for memory mapped registers of a specific microcontroller. This allows users to directly interface with the hardware in a safer way than dereferencing a bunch of hand coded raw pointers.

## Hardware abstraction Layer Crates (HAL)
Building upon a PAC and a microarchitecture crate the HAL crate for a specific MCU creates a programmer-friendly way to interface with the provided hardware.
Its main job is to implement generic traits for for the available hardware.

Here are some examples of commonly used traits:
- [Wait](https://docs.rs/embedded-hal-async/latest/embedded_hal_async/digital/trait.Wait.html) - Async await for GPIO pins
- [I2c](https://docs.rs/embedded-hal-async/latest/embedded_hal_async/i2c/trait.I2c.html) - I2c bus interface
- [SpiDevice](https://docs.rs/embedded-hal/1.0.0/embedded_hal/spi/trait.SpiDevice.html) - Represents a SpiDevice; should be used when implementing a driver for an Spi device
- [BlockDevice](https://docs.rs/block-device-driver/0.2.0/block_device_driver/trait.BlockDevice.html) - Storage medium which might be connected over an sdmmc interface