# What is a MCU

We will be focusing on microcontrollers (MCUs) in this book.
A typical microcontroller consists of a few components:
- One or more cores (Cortex-M / Cortex-R / ...) for processing
- Memory (Flash, RAM)
- Input/Output (Digital, Analog)
- Communication Interfaces (I2C, SPI, USB, CAN, Ethernet)

## Difference to a Microprocessor (MPU)

All modern microprocessors come with a Memory Management Unit. This adds the capability of virtual memory. Virtual memory makes it possible for programs to have their own address space which helps to isolate programs and allows for abstractions like operating systems.