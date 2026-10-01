# Changelog

All notable changes to embedded-hal-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.2.0 — 2026-10-01

A board no longer implements the traits; this package does.

### Changed

- **Every trait method that forwards to a `hal.*` hook has that forward
  as its body**: `GpioOut.mode`, `set` and `get`; `UartTx.init` and
  `write_byte`; `UartRx.read_byte` and `try_read_byte`, which read the
  board's console input; the four `Timer` methods; and the arms,
  disarms and interrupt counts of `AsyncRead` and `AsyncDeadline`.  The
  hook is the board's, linked by the build for the board a program
  targets.
- **The package implements the traits for the board's handles**:
  `GpioOut` for `BoardGpio`, `UartTx`, `UartRx` and `AsyncRead[hw]` for
  `BoardUart`, `Timer` and `AsyncDeadline[hw]` for `BoardTimer`.  Each
  takes its methods from the trait, except `BoardGpio`'s `toggle`,
  which forwards to `hal.gpio.toggle`.  A board package that carried
  its own `board_hal.nv` deletes it; what its silicon does better it
  writes as a `@bsp_impl` hook.  Breaking: a board that keeps an
  `impl` of one of these traits for one of these handles now has two,
  and the build refuses the second.
- **Each `*_arm` method takes the caller's Future and answers at
  once**: `read_arm(self, f: Future) -> Int`,
  `after_us_arm(self, us: Int, f: Future) -> Int`,
  `edge_arm(self, pin: Int, edge: Int, f: Future) -> Int`, and
  `AsyncBlock`'s `read_arm`, `write_arm` and `erase_arm` with the block
  first.  The answer is 0 when the operation is armed and negative when
  it is refused.  An arm allocates nothing, so a `@no_alloc` task arms
  through a trait, with Futures made during initialisation and set
  back to pending with `core.rt.future_reset`.  Breaking: a caller
  writes `let f = core.rt.future_new()` once and `src.read_arm(f)`
  where it wrote `let f = src.read_arm()`; an implementation resolves
  the Future it is given where it made one.
- **`BlockDevice.read_block` and `write_block` move a block through a
  buffer the caller owns**, a `Vec[u8; 512]` passed by address:
  `read_block(self, block: Int, var dst: Vec[u8; 512]) -> Bool`
  replaces the buffer's contents with the block's bytes, and
  `write_block(self, block: Int, var src: Vec[u8; 512]) -> Bool` writes
  the bytes it holds.  An implementation allocates nothing.  Breaking:
  `read_block` answered `?[u8]` and `write_block` took a `[u8]`.

The release needs novo 0.18.0 or newer, the first that reads an impl
written as its header alone and has `core.rt.future_reset`.

## 0.1.0 — 2026-09-30

The first release. The module `embedded_hal` declares the sync and
async traits for GPIO, UART, timers, SPI, I2C, ADC, PWM and block
storage, the awaitable traits `AsyncRead`, `AsyncDeadline`, `AsyncEdge`
and `AsyncBlock`, and the board handles `BoardGpio`, `BoardUart` and
`BoardTimer`. The traits were previously the unpublished package `hal`
in the novo-lang source tree, one module per peripheral; a program that
wrote `use hal.awaitable` or `use block` writes `use embedded_hal`.

The release needs novo 0.17.0 or newer. The board packages of the
nRF52840-DK, the STM32F3DISCOVERY, the MPS2 AN386, the MPS2 AN505 and
the Netduino Plus 2 implement the traits for the three handles.
