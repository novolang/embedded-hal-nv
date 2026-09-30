---
package: embedded-hal-nv
title: An awaitable arm makes its own Future, so a @no_alloc function cannot call it
slug: an-awaitable-arm-allocates-a-future-so-no-alloc-code-cannot-call-it
status: open
discovered: 2026-09-29
fixed:
version: 0.1.0
toolchain: 0.15.0-94-g8902d15e7
platform: linux/x86_64
fixed_in:
class: debt
refs: []
upstream:
related: []
area:
discovered_during: w56-e9-dogfood
duplicate_of:
tracker:
---

# An awaitable arm makes its own Future, so a @no_alloc function cannot call it

## Symptoms

Every `*_arm` method of the awaitable traits (`AsyncRead.read_arm`,
`AsyncDeadline.after_us_arm`, `AsyncEdge.edge_arm`, and
`AsyncBlock.read_arm`, `write_arm` and `erase_arm`) answers a `Future`,
so an implementation calls `core.rt.future_new()` inside the arm.  On a
device a Future is a 24-byte block of the arena.  A task body is
`@no_alloc` on the embedded tier, so a task that arms a peripheral
through a trait is refused with E4005, naming `future_new`.

The module's own comment says the opposite: "A method here does not
allocate.  The Future is the caller's to create and release, one per
event."  The signatures do not let the caller supply it.

## Repro

```novo
use core.rt
use embedded_hal

@tier(embedded)
fn reader() [hw, async]
    let uart = BoardUart { id: 0 }
    loop
        let f = uart.read_arm()
        let _b = core.rt.future_await(f)
        core.rt.future_drop(f)

@tier(embedded)
fn main() [hw, async]
    core.rt.spawn(reader)
    core.rt.run()
```

```
$ novo build --target=nrf52840-dk repro.nv
expected: builds; the read is allocation-free
actual:   E4005: the task body 'reader' reaches the arena through
          'read_arm' and core.rt.future_new
```

## Cause

The arm returns the Future rather than resolving one it is given.  The
board's implementation has nowhere to take a Future from but the arena.

## Workaround

Call the board's `hal.*` hook directly with a Future made during
initialisation, and reuse it for every operation while the device
completes each operation before the arm returns.  orbit/sensorhub does
this for the block device.

## Fix

A signature change for the next minor version: each arm takes the
caller's Future, `fn read_arm(self, f: Future) -> Int [e]`, answering 0
when armed and a negative value when refused, as the `hal.*` hooks
already do.  A caller makes its Futures during initialisation, or takes
them from a pool.  Reusing one Future for several operations also needs
a way to set a resolved Future back to pending, which `core.rt` does
not have.

## History

- 2026-09-29 — filed while putting orbit/sensorhub on the embedded
  memory model: its block device calls go through `hal.block` rather
  than `AsyncBlock` for this reason.
