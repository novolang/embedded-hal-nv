# embedded-hal-nv

A **hardware abstraction layer (HAL)** is the set of operations a
program uses to drive a microcontroller's peripherals, such as setting
a pin, writing a byte to a serial port or waiting for a millisecond,
written so that the program does not depend on which chip it runs on.
This package is that set for novo-lang, as traits. A board's package
implements the traits for its own peripherals, so a program written
against them builds for every board that does. The design follows
Rust's [`embedded-hal`](https://docs.rs/embedded-hal) and
[`embedded-hal-async`](https://docs.rs/embedded-hal-async).

## What it is

A **trait** is a named set of method signatures. A type implements a
trait by providing a body for each method, and a function that takes a
parameter of the trait's type accepts any type that implements it. Each
call site of such a function is compiled for the type it passes, so the
call reaches that type's method directly.

A **peripheral** is a hardware block beside the processor core: a
general-purpose input/output (GPIO) port, a universal asynchronous
receiver-transmitter (UART), a timer, a serial peripheral interface
(SPI) or inter-integrated circuit (I2C) bus, an analogue-to-digital
converter (ADC), a pulse-width modulation (PWM) channel, or a block of
storage. Most peripheral kinds have two traits here:

- a **sync** trait, whose methods return when the operation is done,
  waiting by spinning the core if they must;
- an **async** trait, whose methods let the calling task wait while
  others run.

A peripheral that can only do one of the two implements that trait
alone. A software UART that toggles a pin has no way to run in the
background, so it implements `UartTx` and not `UartTxAsync`.

The async traits come in two forms. The ones spelled `async fn`, such as
`UartTxAsync`, run on a host's scheduler. On a microcontroller at
`@tier(embedded)` an `async fn` is refused (error E4002), so the
**awaitable** traits there take its place: each `*_arm` method starts
the operation and returns a `Future`, which the peripheral's interrupt
completes and the caller awaits.

A **board** is a particular circuit around a chip, such as the
nRF52840-DK. The package declares three **handles** that name a board's
peripherals without naming the board: `BoardGpio`, `BoardUart` and
`BoardTimer`. A handle is a struct with one field, `id`, which selects
the instance of the peripheral (0 for the first). It holds no other
state, since the state is in the peripheral's registers, so building
one allocates nothing. The board's package implements the traits for
the handles.

## Install

```
novo pkg add embedded-hal-nv
```

A program built for a board whose package implements the traits needs
no install: the board's package depends on this one, and the build adds
it when the program writes `use embedded_hal`.

## Example

The program below blinks a board's first LED. It names no board: built
with `novo build --target=nrf52840-dk` it blinks LED1 on the
nRF52840-DK, with `--target=stm32f3discovery` it blinks LD3 on the
STM32F3DISCOVERY, and with `--target=nrf52-qemu` it runs on the Arm
MPS2 AN386 that QEMU emulates, which prints each toggle.

```novo norun:needs-pkg
use embedded_hal

// Any GPIO and any timer: the call sites below pass the board's.
fn blink(led: GpioOut, clock: Timer, pin: Int) [hw]
    led.toggle(pin)
    clock.delay_ms(500)

@tier(embedded)
fn main() [hw]
    let gpio = BoardGpio { id: 0 }
    let timer = BoardTimer { id: 0 }
    let led = bsp.board.led1()     // the board's LED1 pin
    gpio.mode(led, 1)              // 1: output
    while true
        blink(gpio, timer, led)
```

## What the package contains

| Module | What is in it |
| --- | --- |
| `embedded_hal` | The three board handles; the sync and async traits for GPIO, UART, timers, SPI, I2C, ADC, PWM and block storage; and the awaitable traits `AsyncRead`, `AsyncDeadline`, `AsyncEdge` and `AsyncBlock`. |

| Peripheral | Sync trait | Async trait | Awaitable trait |
| --- | --- | --- | --- |
| GPIO | `GpioOut` | `GpioInAsync` | `AsyncEdge` |
| UART transmit | `UartTx` | `UartTxAsync` | — |
| UART receive | `UartRx` | `UartRxAsync` | `AsyncRead` |
| Timer | `Timer` | `TimerAsync` | `AsyncDeadline` |
| SPI | `SpiBus` | `SpiBusAsync` | — |
| I2C | `I2cBus` | `I2cBusAsync` | — |
| ADC | `Adc` | `AdcAsync` | — |
| PWM | `Pwm` | — | — |
| Block storage | `BlockDevice` | `BlockDeviceAsync` | `AsyncBlock` |

PWM has no async trait: a register write sets the duty cycle and there
is no completion to wait for.

## How to choose an entry point

- A program for any board uses the handles and the sync traits, which
  every board implements first.
- A driver for a device on a bus, such as a sensor on I2C, takes the
  bus as a trait-typed parameter (`bus: I2cBus`), so it works on any
  board and with a test double.
- A program that waits for a peripheral at `@tier(embedded)` uses the
  awaitable traits. The `async fn` traits are for a host's scheduler.

## The rules a user needs

1. A method's `pin`, `channel`, `block` and `addr` are the hardware's own
   numbers. `GpioOut.mode`'s `dir` is 0 for input, 1 for output, 2 for
   input with pull-up and 3 for input with pull-down.
2. `Timer.delay_ms` and `delay_us` spin the core for the interval.
   `TimerAsync.sleep_ms` and `AsyncDeadline.after_us_arm` let it sleep.
3. `UartRx.read_byte` waits until a byte arrives. `try_read_byte`
   answers `None` at once when no byte is waiting.
4. `I2cBus.read` and `write_read` answer `None` when the device did not
   acknowledge, which is not the same as a device that sent no bytes.
5. An awaitable source holds one operation at a time, because the
   hardware has one. A second `*_arm` is refused: the Future resolves
   with a negative value.
6. Every `*_arm` has an inverse, `*_disarm`. It answers `true` when the
   operation was still armed and the source has let go of the Future,
   which the caller may then release. It answers `false` when nothing
   was armed or the interrupt already resolved the Future, which the
   caller should read. `core.rt.future_cancel` alone does not tell the
   peripheral, which would later resolve a released Future.
7. `*_irqs` counts the interrupts a source has taken. A board whose
   implementation completes the operation before the arm returns
   reports 0.
8. Five methods have a default body, so an implementation provides only
   the primitives: `UartTx.write_bytes` writes each byte with
   `write_byte`, `GpioOut.toggle` writes the other level with `get` and
   `set`, `Adc.read_mv` converts `read_raw` for a 3.3 V reference and a
   12-bit count, and `I2cBus.write_reg` and `read_reg` are one `write`
   and one `write_read`. An implementation may override any of them.

## Running on a microcontroller

The traits are written for a microcontroller with no heap: no method
signature needs an allocation, and a handle is an `Int` held inline.
A method that takes or returns a `[u8]`, such as `UartTx.write_bytes`,
needs a list, which a program at `@tier(embedded)` builds only into
storage it owns.

Five boards implement `GpioOut`, `UartTx`, `UartRx` and `Timer` for the
handles, and the awaitable `AsyncRead` for `BoardUart` and
`AsyncDeadline` for `BoardTimer`:

| Board | `--target=` | GPIO | UART | Timer | Awaitable |
| --- | --- | --- | --- | --- | --- |
| Nordic nRF52840-DK | `nrf52840-dk` | port 0's registers, pulls included | RTT, the debug probe's log channel, or UART0 with `--hci-uart=physical` | the core's cycle counter at 64 MHz; the tick count is the time the delays have spent | complete before the arm returns (`*_irqs` reads 0) |
| ST STM32F3DISCOVERY | `stm32f3discovery` | the GPIO ports' registers, pulls included | RTT, or USART1 with `--hci-uart=physical` | a microsecond clock on TIM2 | complete before the arm returns |
| Arm MPS2 AN386 under QEMU | `nrf52-qemu` | a pin state in memory, each write printed | writes through semihosting, reads from the machine's UART | the machine's 25 MHz counter | complete from the UART's receive interrupt and TIMER1's |
| Arm MPS2 AN505 under QEMU | `nrf53-qemu` | as the AN386 | as the AN386 | the machine's 20 MHz counter | as the AN386 |
| Netduino Plus 2 under QEMU | `stm32f3-qemu` | a pin state in memory, each write printed | writes through semihosting, reads from USART1 | a microsecond clock on TIM2 | the read from USART1's interrupt, the deadline before the arm returns |

`UartRx` reads the board's console: RTT's down channel on a board that
logs over RTT, the UART under `--hci-uart=physical`, and the machine's
UART on an emulator.  A board that logs through semihosting has no
console input to poll, so `try_read_byte` answers `None` there.

The image carries only the board methods the program calls. Against
the same program written with the standard library's `hal.*` calls, a
program written with the traits uses the same RAM, and its code is
under 1% larger: the methods it calls are kept as functions of their
own beside the copies inlined at their calls.

## What is not included

- An implementation for any board. Each board's package carries its
  own, because the registers differ from chip to chip.
- A trait for interrupts. An interrupt handler is a function marked
  `@isr(<line>)`, whose signature is fixed by the hardware.
- A trait for DMA descriptors. An async implementation uses whatever
  DMA the chip has, and the trait shows only the result.
- USB, Ethernet and radio traits. Each is large enough to need its own
  design.

## Related packages

The standard library's `hal.*` functions (`hal.gpio.set`,
`hal.uart.write`) drive the same peripherals of the board a program is
built for, without traits. A board's implementation of these traits
calls them.

## Tests

`tests/defaults_tests.nv` runs the five default methods against test
doubles on a host: which primitive each one calls and with what value,
and what `read_reg` and `read_mv` answer. Each board's implementation
is tested in the board's package, by a program that calls every method
through the traits: under QEMU for an emulated board, and on the board
through its debug probe for the nRF52840-DK and the STM32F3DISCOVERY.
Each measures the implementation's line coverage on the device.

```
novo pkg build
novo test tests
```

## Licence

Apache-2.0. See `LICENSE`.
