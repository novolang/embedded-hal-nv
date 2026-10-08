# Changelog

All notable changes to embedded-hal-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.3.2 — 2026-10-08

The toolchain floor is 0.19.4, which carries `hal.watchdog`; 0.3.1 named
0.19.3, whose toolchain has no such module, so its Watchdog bodies could
not build there, and 0.3.1 is withdrawn.  No signature changed.

## 0.3.1 — 2026-10-08

The chip's watchdog, and why the chip last reset.

### Added

- **`Watchdog`**, a trait for a watchdog: `start(timeout_ms)` arms it
  and answers whether it took the timeout; `feed()` restarts the count;
  `timeout_ms()` reads back the timeout the hardware was set to, which
  its counter's steps round; `can_stop()` answers whether the watchdog
  can be stopped once started, and `stop()` stops it, refused where it
  cannot.  Every method forwards to the standard library's
  `hal.watchdog`, which the build generates over the watchdog a board
  names with `@hal(watchdog)`; on a board that names none, a start is
  refused.
- **`BoardWatchdog`**, the handle for the chip's watchdog, and the
  implementation of `Watchdog` for it.
- **`reset_reason()`**, why the chip last reset, as one of
  `RESET_OTHER`, `RESET_POWER_ON`, `RESET_PIN`, `RESET_WATCHDOG` and
  `RESET_SOFTWARE`, from `hal.watchdog.reset_reason()`, which the
  board's family runtime answers.
- `tests/watchdog_tests.nv`, the trait's bodies on the host and a
  watchdog of a program's own.

No signature of 0.3.0 changed.  The toolchain floor stays 0.19.3; the
bodies of the additions need `hal.watchdog`, which the release after
0.19.3 carries.

## 0.3.0 — 2026-10-08

A CAN controller, classic frames only.

### Added

- **`CanBus`**, a trait for a CAN controller: `open(bitrate, mode)`
  starts it at a bit rate in `CAN_MODE_NORMAL` or `CAN_MODE_LOOPBACK`,
  where the controller receives its own frames and drives nothing onto
  the bus; `send(frame)` hands it a frame; `receive()` answers the next
  frame, or `None` when none waits; `tx_done()` answers whether no frame
  still waits for the bus; `bitrate()` reads the rate back from the
  controller; `errors()` reads the error counters and the fault
  confinement state.  Every method forwards to the standard library's
  `hal.can`, which the build generates over the controller a board
  names with `@hal(can)`; on a board that names none an open is refused.
- **`CanBusFrame`**, a `@value` frame: an 11-bit or 29-bit identifier,
  the remote flag, a length of 0 to 8 and the data bytes in one `Int`,
  byte 0 in bits 0 to 7.  `can_data`, `can_data_ext` and `can_remote`
  build one; `can_byte`, `can_payload`, `can_frame_valid`, `can_flags`,
  `can_header` and `can_frame_of` read and pack it.
- **`CanBusErrors`**, the transmit and receive error counters and the
  state, `CAN_ERROR_ACTIVE`, `CAN_ERROR_PASSIVE` or `CAN_BUS_OFF`, and
  `can_errors_of`, which unpacks `hal.can.errors`' word.
- **`BoardCan`**, the handle for a board's CAN controller, and the
  implementation of `CanBus` for it.

CAN FD is not here; its frames carry up to 64 bytes and would be a trait
of its own.  No `async fn` form is declared: an `async fn` takes no
`@value` struct, and a frame is one.

### Changed

- `tests/board_tests.nv` runs the GPIO, UART and timer handles'
  forwarding bodies on the host, and `tests/coverage.sh` reports the
  line coverage of `src/` merged across the suites; the lines only a
  board runs carry a `cov: skip` marker with the reason.
- The toolchain floor is 0.19.3, the first release with `hal.can`.  The
  minor number moves because a toolchain older than the floor cannot
  build the module at all, so a build that took this release under a
  `^0.2` requirement would fail.  No signature of 0.2.1 changed.

## 0.2.1 — 2026-10-06

A device driver has a concrete board bus to hold.

### Added

- **`BoardI2c` and `BoardSpi`**, handles for the board's I2C and SPI
  buses, `@value` structs with one field, `id`, like the other three.
  The package implements `I2cBus` for `BoardI2c` and `SpiBus` for
  `BoardSpi`.
- **`I2cBus` and `SpiBus` forward to the board's buses**: `init`,
  `write`, `read` and `write_read` call `hal.i2c`'s operation of the
  same name, and `init`, `transfer`, `write` and `read` call `hal.spi`'s.
  The build generates those operations over the driver the board names
  with `@hal(i2c)` and `@hal(spi)`; on a board that names none, no I2C
  device acknowledges and an SPI read answers no bytes.  A type of a
  program's own that writes all four methods, as a test double does, is
  unchanged.

### Changed

- The toolchain floor is 0.18.4, the first release with `hal.i2c` and
  `hal.spi` as layers over the board's drivers.

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
