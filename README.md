# embedded-hal-nv

A **hardware abstraction layer (HAL)** is the set of operations a
program uses to drive a microcontroller's peripherals, such as setting
a pin, writing a byte to a serial port or waiting for a millisecond,
written so that the program does not depend on which chip it runs on.
This package is that set for novo-lang, as traits, and their
implementation for the board a program is built for: each method
forwards to the standard library's `hal.*` hook of the same operation,
which the board supplies. A program written against the traits builds
for every board. The design follows
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
(SPI) or inter-integrated circuit (I2C) bus, a controller area network
(CAN) controller, an analogue-to-digital converter (ADC), a pulse-width
modulation (PWM) channel, a block of storage, or a watchdog, a counter
that resets the chip unless the program feeds it in time. Most
peripheral kinds have two traits here:

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
the operation on a `Future` the caller owns, which the peripheral's
interrupt completes and the caller awaits.

A **board** is a particular circuit around a chip, such as the
nRF52840-DK. The package declares seven **handles** that name a board's
peripherals without naming the board: `BoardGpio`, `BoardUart`,
`BoardTimer`, `BoardI2c`, `BoardSpi`, `BoardCan` and `BoardWatchdog`. A handle is a struct with one field, `id`, which selects
the instance of the peripheral (0 for the first). It holds no other
state, since the state is in the peripheral's registers, so building
one allocates nothing.

The package implements the traits for the handles. A trait method's
body calls the `hal.*` hook of the same operation (`GpioOut.set` calls
`hal.gpio.set`), and the build links the hooks of the board it is for:
the board package's own, written in novo-lang over the chip's
registers, or the runtime's default for its chip where the board writes
none. A board never implements a trait; what its silicon does better it
writes as a hook.

## Install

```
novo pkg add embedded-hal-nv
```

A program built for a board needs no install: the board's package
depends on this one, and the build adds it when the program writes
`use embedded_hal`.

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
| `embedded_hal` | The seven board handles; the sync and async traits for GPIO, UART, timers, SPI, I2C, ADC, PWM and block storage, the sync trait for CAN with its frame, `CanBusFrame`, and its counters, `CanBusErrors`, and the sync trait for the watchdog, `Watchdog`, with `reset_reason()` and its `RESET_*` values; the awaitable traits `AsyncRead`, `AsyncDeadline`, `AsyncEdge` and `AsyncBlock`; and the implementation of `GpioOut`, `UartTx`, `UartRx`, `AsyncRead`, `Timer`, `AsyncDeadline`, `I2cBus`, `SpiBus`, `CanBus` and `Watchdog` for the handles. |

| Peripheral | Handle | Sync trait | Async trait | Awaitable trait |
| --- | --- | --- | --- | --- |
| GPIO | `BoardGpio` | `GpioOut` | `GpioInAsync` | `AsyncEdge` |
| UART transmit | `BoardUart` | `UartTx` | `UartTxAsync` | — |
| UART receive | `BoardUart` | `UartRx` | `UartRxAsync` | `AsyncRead` |
| Timer | `BoardTimer` | `Timer` | `TimerAsync` | `AsyncDeadline` |
| SPI | `BoardSpi` | `SpiBus` | `SpiBusAsync` | — |
| I2C | `BoardI2c` | `I2cBus` | `I2cBusAsync` | — |
| CAN | `BoardCan` | `CanBus` | — | — |
| Watchdog | `BoardWatchdog` | `Watchdog` | — | — |
| ADC | — | `Adc` | `AdcAsync` | — |
| PWM | — | `Pwm` | — | — |
| Block storage | — | `BlockDevice` | `BlockDeviceAsync` | `AsyncBlock` |

A handle implements the sync trait and, where there is one, the
awaitable trait of its row; the async traits are for types of a
program's own on a host, since a handle is a `@value` struct and an
`async fn` takes none. PWM has no async trait: a register write sets
the duty cycle and there is no completion to wait for. CAN has no
`async fn` trait either, since its frame is a `@value` struct; a program
polls `receive` and `tx_done`. The watchdog has no async trait: feeding
it is a register write that completes at once.

## How to choose an entry point

- A program for any board uses the handles and the sync traits.
- A driver for a device on a bus, such as a sensor on I2C, writes its
  register reads and writes as functions generic over the bus
  (`<B: I2cBus>`), so they work on any board and with a test double,
  and holds the board's bus, a `BoardI2c` or a `BoardSpi`, in the type
  a program uses.
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
   An I2C `addr` is the device's 7-bit address. `SpiBus.init`'s `mode`
   is the SPI mode, 0 to 3, and chip select is a GPIO pin the caller
   drives around a transfer.
5. A CAN frame is classic: an identifier of 11 bits, or 29 with
   `extended`, up to 8 data bytes held in one `Int` with byte 0 in bits 0
   to 7, and the remote flag. `CanBus.send` refuses a frame whose
   identifier does not fit or whose length is past 8, and a frame while
   the last one still waits for the bus. `CanBus.receive` answers `None`
   at once when no frame waits. `CanBus.open` refuses a bit rate the
   controller's clock cannot give exactly. `CAN_MODE_LOOPBACK` receives
   every frame sent and drives nothing onto the bus.
6. `Watchdog.start` refuses a timeout of 0 or less and one past the
   longest the watchdog holds, and `timeout_ms` reads back what the
   hardware was set to, which its steps round. A watchdog that cannot be
   stopped once started answers `false` from `can_stop` and refuses
   `stop`; nothing but a reset stops it. `reset_reason()` answers why the
   chip last reset, `RESET_WATCHDOG` after the watchdog's reset, and
   `RESET_OTHER` on a chip that cannot tell.
7. An awaitable arm takes the caller's Future and answers 0 when the
   operation is armed and a negative value when it is refused. The
   caller makes its Futures once, with `core.rt.future_new()`, and sets
   one back to pending with `core.rt.future_reset(f)` before the next
   arm, so the arm allocates nothing and a `@no_alloc` task can call it.
   An awaitable source holds one operation at a time, because the
   hardware has one, and refuses a second arm.
8. Every `*_arm` has an inverse, `*_disarm`. It answers `true` when the
   operation was still armed and the source has let go of the Future,
   which the caller may then release. It answers `false` when nothing
   was armed or the interrupt already resolved the Future, which the
   caller should read. `core.rt.future_cancel` alone does not tell the
   peripheral, which would later resolve a released Future.
9. `*_irqs` counts the interrupts a source has taken. A board whose
   implementation completes the operation before the arm returns
   reports 0.
10. Two kinds of method have a body. Those of `GpioOut`, `UartTx`,
   `UartRx`, `Timer`, `AsyncRead`, `AsyncDeadline`, `I2cBus`, `SpiBus`,
   `CanBus` and `Watchdog` that name one operation forward to the board's `hal.*`
   hook, which is how the handles are implemented; `CanBus.send` and
   `receive` also pack and unpack the frame for `hal.can`. Five more are written over the trait's own
   methods: `UartTx.write_bytes` writes each byte with `write_byte`,
   `GpioOut.toggle` writes the other level with `get` and `set`,
   `Adc.read_mv` converts `read_raw` for a 3.3 V reference and a 12-bit
   count, and `I2cBus.write_reg` and `read_reg` are one `write` and one
   `write_read`.
11. A type of a program's own, such as a driver for a device on a bus or
   a test double, writes every method of a trait it implements. A
   method it leaves out takes the trait's body, which for the forwarding
   methods drives the board's peripheral and not the type's.
12. `BlockDevice.read_block` and `write_block` move a block through a
   `Vec[u8; 512]` the caller owns and passes as a `var` argument:
   `read_block` replaces its contents with the block's bytes. Neither
   side allocates.

## Running on a microcontroller

The traits are written for a microcontroller with no heap: no method
signature needs an allocation, and a handle is an `Int` held inline.
A method that takes or returns a `[u8]`, such as `UartTx.write_bytes`,
needs a list, which a program at `@tier(embedded)` builds only into
storage it owns.

Every board in novo-lang's catalogue supplies the hooks the traits
forward to. Five of them, as an example of what the hooks do:

| Board | `--target=` | GPIO | UART | Timer | I2C, SPI and CAN | Awaitable |
| --- | --- | --- | --- | --- | --- | --- |
| Nordic nRF52840-DK | `nrf52840-dk` | port 0's registers, pulls included, through the nRF GPIO driver written in novo-lang | RTT, the debug probe's log channel, or UARTE0 with `--hci-uart=physical`, whose receive completes from its interrupt | the core's cycle counter at 64 MHz; the tick count is the time the delays have spent | no driver yet: no device acknowledges, and an SPI read answers no bytes | the read from UARTE0's interrupt under `--hci-uart=physical`, else before the arm returns; the deadline before the arm returns |
| NXP FRDM-MCXN947 | `frdm-mcxn947` | the PORT and GPIO blocks' registers, pulls included | RTT, or LP_FLEXCOMM4's LPUART with `--hci-uart=physical` | a microsecond clock on CTIMER0 | I2C on LP_FLEXCOMM5's LPI2C, the bus of the board's P3T1755DP temperature sensor, at address 0x48; SPI on LP_FLEXCOMM1's LPSPI; CAN on CAN0, a FlexCAN, in loopback or on the board's transceiver | the read from LP_FLEXCOMM4's interrupt under `--hci-uart=physical`, else before the arm returns; the deadline before the arm returns |
| ST STM32F3DISCOVERY | `stm32f3discovery` | the GPIO ports' registers, pulls included | RTT, or USART1 with `--hci-uart=physical` | a microsecond clock on TIM2 | no driver yet | complete before the arm returns |
| Arm MPS2 AN386 under QEMU | `nrf52-qemu` | a pin state in memory, each write printed | writes through semihosting, reads from the machine's UART | the machine's 25 MHz counter | SPI on the machine's first PL022 controller, in its loopback mode, so each byte sent is the byte read; no I2C driver | complete from the UART's receive interrupt and TIMER1's |
| Arm MPS2 AN505 under QEMU | `nrf53-qemu` | as the AN386 | as the AN386 | the machine's 20 MHz counter | no driver yet | as the AN386 |

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

- The hooks. Each board's package, or the runtime for its chip,
  carries them, because the registers differ from chip to chip.
- A trait for interrupts. An interrupt handler is a function marked
  `@isr(<line>)`, whose signature is fixed by the hardware.
- A trait for DMA descriptors. An async implementation uses whatever
  DMA the chip has, and the trait shows only the result.
- USB, Ethernet and radio traits. Each is large enough to need its own
  design.

## Related packages

The standard library's `hal.*` functions (`hal.gpio.set`,
`hal.uart.write`) drive the same peripherals of the board a program is
built for, without traits. The traits' forwarding bodies call them.

## Tests

`tests/defaults_tests.nv` runs the five bodies written over a trait's
own methods against test doubles on a host: which method each one
calls and with what value, and what `read_reg` and `read_mv` answer.
`tests/board_tests.nv` calls the handles on a host, whose hooks answer
a pin as 0, refuse an arm with -1 and acknowledge no I2C write, so the
answers show each call reached the hook; arms one Future again and again with
`core.rt.future_reset`; and reads blocks into a caller's buffer.
`tests/can_tests.nv` checks the CAN frame and its words, the board's
controller on a host, which refuses the open and receives nothing, and
`CanBus` over a bus of a program's own. Each
board's hooks are tested in the board's package, by a program that
calls every method through the traits: under QEMU for an emulated
board, and on the board through its debug probe. Each measures the
line coverage of this module's bodies and the board's hooks on the
device.

```
novo pkg build
novo test tests
bash tests/coverage.sh
```

`tests/coverage.sh` merges the suites' line coverage of `src/`. The
lines a host test cannot run, the console's reads and a CAN frame
received, are marked with their reason and run in the boards' suites.

## Licence

Apache-2.0. See `LICENSE`.
