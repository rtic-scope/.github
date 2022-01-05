# RTIC Scope

RTIC Scope is an non-intrusive auxillary toolset for tracing [RTIC](https://rtic.rs) programs executing on ARMv7-M targets by exploiting the *Instrumentation Trace Macrocell* (ITM) and *Data Watchpoint and Trace* (DWT) units.
The target-generated ITM trace packet stream (henceforth referred to as the "trace stream") is recorded on a host system by reading from the *Embedded Trace Buffer* (ETB) via [probe-rs](https://probe.rs) or a serial device connected to the target's *Trace Port Interface Unit* (TPIU).

Recorded trace streams can be analyzed in real-time or replayed offline for post-mortem purposes.

## Toolset components

RTIC Scope is a collection of three crates:
- [`cargo-rtic-scope`](https://github.com/rtic-scope/cargo-rtic-scope/tree/master/cargo-rtic-scope): the host-side daemon that records the trace stream, recovers RTIC app metadata, serializes the replay file to disk, and forwards the trace to a frontend;
- [`rtic-scope-api`](https://github.com/rtic-scope/cargo-rtic-scope/tree/master/rtic-scope-api): the `serde` JSON API implemented by `cargo-rtic-scope` and any frontend; and
- [`cortex-m-rtic-trace`](https://github.com/rtic-scope/cargo-rtic-scope/tree/master/cortex-m-rtic-trace): an auxilliary target-side crate that propely configures the ITM/DWT/TPIU units. [^1]

[^1]: This crate is a crutch and will be deprecated on v1.0.0 release: see https://github.com/rtic-scope/cargo-rtic-scope/issues/90.

A [dummy frontend, `rtic-scope-frontend-dummy`](https://github.com/rtic-scope/frontend-dummy) [^2] is also available for reference purposes, but can be substituted by any frontend that implements the API: a graphical user interface, a database client, etc.

[^2]: The dummy only prints received trace information to `stderr` with absolute microsecond (and relative nanosecond) timestamps. These messages are echoed by `cargo-rtic-scope`.

The other repositories listed below (except [`itm`](https://github.com/rtic-scope/itm) [^3]) are dependencies with patches that are due (or already have been) pushed upstream.

[^3]: `itm` is a library for decoding the ITM packet protocol. Because of its general nature and detachment from the implemention of RTIC Scope, it is not a part of the project itself, but hosted here for convenience.

## Performance/resource impact of ITM/DWT tracing

[ARM's *Understanding Trace*, §7](https://developer.arm.com/documentation/102119/0200/Can-trace-capture-affect-a-system-) states that:
> Except for the power that is consumed by the system trace components, trace is almost entirely non-invasive. This means that performing trace generation and collection does not influence the wider system.

The target-side code of RTIC Scope itself has a negligible performance impact during execution:
- the ITM/DWT/TPIU units need only be configured once in `#[init]` or during a later stage; and
- when software tasks are traced, a `u8` variable write must be done when entering and exiting the task.

Resource-wise, however, two DWT comparators currently need to be consumed in order to trace software tasks.

The performance of the host-side `cargo-rtic-scope` daemon has yet been measured.

## Limitations
As of v0.3.0, RTIC Scope supports RTIC v1.0.0.
Support for other real-time operating systems is not planned.
For a more general tool, see [orbuculum](https://github.com/orbcode/orbuculum).

## Roadmap
See [the project milestones](https://github.com/rtic-scope/cargo-rtic-scope/milestones).

## Changelog
See [`cargo-rtic-scope/CHANGELOG.md`](https://github.com/rtic-scope/cargo-rtic-scope/blob/master/CHANGELOG.md).

## Getting started
The purpose of RTIC Scope is to enable instant insight into your firmware if it is written in RTIC with ideally zero end-user overhead, except for setting some flags in the RTIC application declaration and potentially writing some configuration in a checked-in file. [^4]

[^4]: RTIC Scope is not quite there yet. See [#100](https://github.com/rtic-scope/cargo-rtic-scope/issues/100).

Start by installing the toolset: [^5]
```shell
$ # Prepare by installing system libraries:
$ # if you have Nix available:
$ nix develop github:rtic-scope/cargo-rtic-scope?dir=contrib
$ # otherwise:
$ sudo apt install -y libusb-1.0-0-dev libftdi1-dev libudev-dev # or equivalent; see <https://github.com/probe-rs/probe-rs#building>
$ # then install RTIC Scope:
$ cargo install --git https://github.com/rtic-scope/cargo-rtic-scope cargo-rtic-scope # install the cargo subcommand
$ cargo install --git https://github.com/rtic-scope/frontend-dummy # install the dummy frontend
```

[^5]: Installation cannot yet be done against the crate registery. See [#101](https://github.com/rtic-scope/cargo-rtic-scope/issues/101).

## How RTIC Scope works, or: the RTIC metadata recovery step

## Publications

- [*RTIC Scope — Real-Time Tracing Support for the RTIC RTOS Framework*](https://github.com/tmplt/masters-thesis): a master's thesis on the development and design of RTIC Scope. Will eventually include an application example on a complex control system.

## License and contact
See the respective repositories for non-commercial licenses.

This project is maintained in cooperation with [@GrepitAB](https://github.com/GrepitAB) and Luleå Technical University.
For commercial support and alternative licensing, inquire via <contact@grepit.se>, or contact me directly at <viktor.sonesten@grepit.se>.

---

This `README.md` is a work in progress until [the v0.3.0 milestone](https://github.com/rtic-scope/cargo-rtic-scope/milestone/3) closes.
