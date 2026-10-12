# Common Tasks

This chapter walks through tasks you will face again and again when building firmware with Embassy:
- [Writing a Device Driver](./5_common_tasks/1_writing_a_driver.md) for a chip or device connected to your MCU
- [Implementing Application Logic](./5_common_tasks/2_application_logic.md) such as connecting tasks or serving HTTP
- [Porting to a new MCU](./5_common_tasks/3_porting_to_new_mcu.md) when the hardware changes

All three share one idea: **keep hardware specific code small and at the edge, and put everything else into platform independent library crates.**

## Project layout
A project following this idea is usually a cargo workspace with several crates:

```text
my-project/
├── Cargo.toml              # workspace
├── drivers/
│   └── temp-sensor/        # driver, only depends on generic traits
├── app/                    # application logic, depends on drivers and traits
├── firmware-stm32f4/       # binary: HAL init, pin mapping, spawning tasks
└── sim-linux/              # binary: same app, running on a PC
```

The dependencies only point in one direction:

```text
 firmware-stm32f4      sim-linux
   │      │              │     │
   │      ▼              ▼     │
   │   ┌─────────────────┐     │
   │   │       app       │     │
   │   └────────┬────────┘     │
   │            ▼              │
   │   ┌─────────────────┐     │
   │   │     drivers     │     │
   │   └────────┬────────┘     │
   ▼            │              ▼
embassy-stm32   │       linux-embedded-hal / socketcan
                ▼
               generic traits
               (embedded-hal, embedded-can, embedded-io, ...)
```

Only the binary crates know which hardware they run on. `app` and `drivers` never depend on a HAL.
This gives you:
- **Testability** - libraries can be tested with `cargo test` on your PC, using mocks instead of real hardware.
- **Portability** - the same code runs on different MCUs, or on Linux.
- **Low coupling** - a change of hardware does not ripple through the whole code base.

> **Tip:** `cargo test` builds for your host, while the firmware needs the embedded target.
> A common setup is to exclude firmware crates from `default-members` in the workspace `Cargo.toml`
> (or to put them into a separate workspace) and build them with `cargo build --target ...` explicitly.
