# Changelog

All notable changes to embedded-hal-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
