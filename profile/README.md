# RTIC Scope

RTIC Scope is an non-intrusive auxillary toolset for tracing [RTIC](https://rtic.rs) programs executing on ARMv7-M targets by exploiting the *Instrumentation Trace Macrocell* (ITM) and *Data Watchpoint and Trace* (DWT) units.
The target-generated ITM trace packet stream (henceforth referred to as the "trace stream") is recorded on a host system by reading from the *Embedded Trace Buffer* (ETM) via [probe-rs](https://probe.rs) or a serial device connected to the target's *Trace Port Interface Unit* (TPIU).

Recorded trace streams can be analyzed in real-time or replayed offline for post-mortem purposes.

## Toolset components

RTIC Scope is a collection of three crates:
- [`cargo-rtic-scope`](https://github.com/rtic-scope/cargo-rtic-scope/tree/master/cargo-rtic-scope): the host-side daemon that records the trace stream, recovers RTIC app metadata, serializes the replay file to disk, and forwards the trace to a frontend;
- [`rtic-scope-api`](https://github.com/rtic-scope/cargo-rtic-scope/tree/master/rtic-scope-api): the `serde` JSON API implemented by `cargo-rtic-scope` and any frontend; and
- [`cortex-m-rtic-trace`](https://github.com/rtic-scope/cargo-rtic-scope/tree/master/cortex-m-rtic-trace): an auxilliary target-side crate that propely configures the ITM/DWT/TPIU units. [^1]

[^1]: This crate is a crutch and will be deprecated on v1.0.0 release: see https://github.com/rtic-scope/cargo-rtic-scope/issues/90.

A [dummy frontend, `rtic-scope-frontend-dummy`](https://github.com/rtic-scope/frontend-dummy) [^2] is also available for reference purposes, but can be substituted by any frontend that implements the API: a graphical user interface, a database client, etc.

[^2]: The dummy only prints received trace information to `stderr` with absolute microsecond (and relative nanosecond) timestamps. These messages are echoed by `cargo-rtic-scope`.

The other repositories listed below are dependencies with patches that are due (or already have been) pushed upstream.

## Performance impact of ITM/DWT tracing

## Limitations

## Roadmap

## Getting started

## How RTIC Scope works

## Publications

TBA

## License and contact
See the respective repositories for non-commercial licenses.

This project is maintained in cooperation with @GrepitAB and Luleå Technical University.
For commercial support and alternative licensing, inquire via <contact@grepit.se>, or contact me directly at <viktor.sonesten@grepit.se>.

---

This `README.md` is a work in progress until [the v0.3.0 milestone](https://github.com/rtic-scope/cargo-rtic-scope/milestone/3) closes.
