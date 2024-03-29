# RTIC Scope

RTIC Scope is an non-intrusive auxillary toolset for tracing [RTIC](https://rtic.rs) programs executing on ARMv7-M targets by exploiting the *Instrumentation Trace Macrocell* (ITM) and *Data Watchpoint and Trace* (DWT) units.
The target-generated ITM trace packet stream (henceforth referred to as the "trace stream") is recorded on a host system by reading from the *Embedded Trace Buffer* (ETB) via [probe-rs](https://probe.rs) or a serial device connected to the target's *Trace Port Interface Unit* (TPIU).

Recorded trace streams can be analyzed in real-time or replayed offline for post-mortem purposes.

## Toolset components

RTIC Scope is a collection of three crates:
- [`cargo-rtic-scope`](https://github.com/rtic-scope/cargo-rtic-scope/tree/master/cargo-rtic-scope): the host-side daemon that records the trace stream, recovers RTIC app metadata, serializes the replay file to disk, and forwards the trace to a frontend;
- [`rtic-scope-api`](https://github.com/rtic-scope/cargo-rtic-scope/tree/master/rtic-scope-api): the `serde` JSON API implemented by `cargo-rtic-scope` and any frontend; and
- [`cortex-m-rtic-trace`](https://github.com/rtic-scope/cargo-rtic-scope/tree/master/cortex-m-rtic-trace): an auxilliary target-side crate that propely configures the ITM/DWT/TPIU units. [^1]

[^1]: This crate is a crutch and will be deprecated on v1.0.0 release: see [#90](https://github.com/rtic-scope/cargo-rtic-scope/issues/90).

A [dummy frontend, `rtic-scope-frontend-dummy`](https://github.com/rtic-scope/cargo-rtic-scope/tree/master/rtic-scope-frontend-dummy) [^2] is also available for reference purposes, but can be substituted by any frontend that implements the API: a graphical user interface, a database client, etc.

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
Support for other real-time operating systems is not planned as implied by the project name; such a project would warrant functionality of RTIC Scope to merge into a larger project.
Interested in adapting RTIC Scope for other RTOSs? Feel free to contact me.

For a more general tool that is not limited to RTIC, see [orbuculum](https://github.com/orbcode/orbuculum).

## Roadmap
See [the project milestones](https://github.com/rtic-scope/cargo-rtic-scope/milestones).

## Changelog
See [`cargo-rtic-scope/CHANGELOG.md`](https://github.com/rtic-scope/cargo-rtic-scope/blob/master/CHANGELOG.md).

## Getting started
The purpose of RTIC Scope is to enable instant insight into your firmware if it is written in RTIC with ideally zero end-user overhead, except for setting some flags in the RTIC application declaration and potentially writing some configuration in a checked-in file. [^4]

[^4]: RTIC Scope is not quite there yet. See [#100](https://github.com/rtic-scope/cargo-rtic-scope/issues/100).

### Installing the toolset
Start by installing the toolset: [^5]
```shell
$ # Prepare by installing system libraries:
$ # if you have Nix available:
$ nix develop github:rtic-scope/cargo-rtic-scope?dir=contrib
$ # otherwise:
$ sudo apt install -y libusb-1.0-0-dev libftdi1-dev libudev-dev # or equivalent; see <https://github.com/probe-rs/probe-rs#building>
$ # then install RTIC Scope: the cargo subcommand and reference frontend:
$ cargo install --git https://github.com/rtic-scope/cargo-rtic-scope cargo-rtic-scope rtic-scope-frontend-dummy
```

[^5]: Installation cannot yet be done against the crate registery. See [#101](https://github.com/rtic-scope/cargo-rtic-scope/issues/101).

### Reading the trace stream from the target
RTIC Scope implements two methods to extract the trace stream from the target.
The recommended method is via `probe-rs` which polls the *Embedded Trace Buffer* (ETB) of the target by specifying `--chip` (e.g. `--chip stm32f401retx`, just like [`cargo-flash`](https://github.com/probe-rs/cargo-flash) [^6]).
Using `probe-rs` also optionally flashes and resets your target with the chosen firmware (specify with `--bin`).
This can be disabled via `--dont-touch-target`.
The second method is by reading the trace stream from a serial device by specifying `--serial /path/to/serial/device`.
A raw ITM trace stream is expected on this device. This trace stream can be read from the SWO or TRACE debug pins. It's left to the end-user to configure hardware such that the expected stream can be read from the serial device.

[^6]: `cargo-rtic-scope` is an extension of `cargo-flash`. All options supported by `cargo-flash` are also supported by `cargo-rtic-scope`, e.g. `--bin` and `--list-chips`.

### Generating the trace stream on the target
When using the `probe-rs` method the target is partially configured to emit ITM packets with timestamps when hardware tasks are entered and exited.
What remains is to flip any device-specific ITM master switches.
Work has begun on `probe-rs` to remove this burden from the end-user, but each platform is different.
No additional target-side configuration must be done unless software task tracing is wanted.
However, for reasons of symmetry, it is recommended that `cortex-m-rtic-trace` is employed.

Starting from a skeleton RTIC application for an `stm32f401retx` we have:
```rust,ignore
//! rtic-scope-example
#![no_main]
#![no_std]

use panic_semihosting as _;
use rtic;

#[rtic::app(device = stm32f4::stm32f401, dispatchers = [EXTI0, EXTI1])]
mod app {
    use cortex_m::peripheral::syst::SystClkSource;

    #[shared]
    struct Shared {}

    #[local]
    struct Local {}

    #[init]
    fn init(mut ctx: init::Context) -> (Shared, Local, init::Monotonics) {
        ctx.core.SYST.set_clock_source(SystClkSource::Core);
        ctx.core.SYST.set_reload(16_000_000); // period = 1s

        // Allow debugger to attach while sleeping (WFI)
        ctx.device.DBGMCU.cr.modify(|_, w| {
            w.dbg_sleep().set_bit();
            w.dbg_standby().set_bit();
            w.dbg_stop().set_bit()
        });

        sw_task::spawn().unwrap();

        (Shared {}, Local {}, init::Monotonics())
    }

    #[task(binds = SysTick)]
    fn hardware(_: hardware::Context) {
        software::spawn().unwrap();
    }

    #[task]
    fn software(_: software::Context) {}
}
```

We employ `cortex-m-rtic-trace` via:
```toml
# Cargo.toml additions
[package.metadata.rtic-scope]
# Required to recover metadata about device-specific exceptions
pac_name = "stm32f4"
pac_features = ["stm32f401"]
pac_version = "0.13"
interrupt_path = "stm32f4::stm32f401::Interrupt"

# required to read the ETB, and to calculate timestamps
tpiu_freq = 16000000
tpiu_baud = 115200
lts_prescaler = 1

# Required for software task tracing
dwt_enter_id = 1
dwt_exit_id = 2

# whether to expect malformed ITM packets; useful to debug the stream extraction method
expect_malformed = true

[dependencies.cortex-m-rtic-trace]
git = "https://github.com/rtic-scope/cortex-m-rtic-trace"

[patch.crates-io]
cortex-m = { version = "0.7", git = "https://github.com/rtic-scope/cortex-m.git", branch = "rtic-scope" }
```
and
```rust,ignore
//! rtic-scope-example
#![no_main]
#![no_std]

use panic_semihosting as _;
use rtic;

#[rtic::app(device = stm32f4::stm32f401, dispatchers = [EXTI0, EXTI1])]
mod app {
    use cortex_m::peripheral::syst::SystClkSource;
    use cortex_m_rtic_trace::{
        self, trace, GlobalTimestampOptions, LocalTimestampOptions, TimestampClkSrc,
        TraceConfiguration, TraceProtocol,
    };

    #[shared]
    struct Shared {}

    #[local]
    struct Local {}

    #[init]
    fn init(mut ctx: init::Context) -> (Shared, Local, init::Monotonics) {
        ctx.core.SYST.set_clock_source(SystClkSource::Core);
        ctx.core.SYST.set_reload(16_000_000); // period = 1s

        // Allow debugger to attach while sleeping (WFI)
        ctx.device.DBGMCU.cr.modify(|_, w| {
            w.dbg_sleep().set_bit();
            w.dbg_standby().set_bit();
            w.dbg_stop().set_bit()
        });

        // flip device-specific master swtich for tracing
        #[rustfmt::skip]
        ctx.device.DBGMCU.cr.modify(
            |_, w| unsafe {
                w.trace_ioen().set_bit() // master enable for tracing
                 .trace_mode().bits(0b00) // TRACE pin assignment for async mode (SWO), feeds into the ETB
            },
        );

        // setup software tracing
        cortex_m_rtic_trace::configure(
            &mut ctx.core.DCB,
            &mut ctx.core.TPIU,
            &mut ctx.core.DWT,
            &mut ctx.core.ITM,
            1, // task enter DWT comparator ID
            2, // task exit DWT comparator ID
            &TraceConfiguration {
                delta_timestamps: LocalTimestampOptions::Enabled, // prescaler = 1
                absolute_timestamps: GlobalTimestampOptions::Disabled,
                timestamp_clk_src: TimestampClkSrc::AsyncTPIU,
                tpiu_freq: 16_000_000, // Hz
                tpiu_baud: 115_200,    // B/s
                protocol: TraceProtocol::AsyncSWONRZ,
            },
        )
        .unwrap();

        sw_task::spawn().unwrap();

        (Shared {}, Local {}, init::Monotonics())
    }

    #[task(binds = SysTick)]
    fn hardware(_: hardware::Context) {
        software::spawn().unwrap();
    }

    #[task]
    #[trace]
    fn software(_: software::Context) {}
}
```
If the target does not support the requested trace configuration, `cortex_m_rtic_trace::configure` will return an `Err`.

### Tracing the RTIC application
With the target appropriately configured, we can now trace the application:
```shell
$ cd /path/to/rtic-scope-example
$ cargo rtic-scope trace --bin rtic-scope-example --chip stm32f401retx
```
or alternatively:
```shell
$ cargo rtic-scope trace --bin rtic-scope-example --serial /path/to/device [--dont-touch-target]
```
While tracing, resolved metadata and recorded ITM packets will be serialized to a `*.trace` file under `target/rtic-traces`.

An example output of this trace command would be
```text
    Building RTIC target application...
  Recovering metadata for trace-example (/path/to/main.rs) and preparing target...
   Recovered 2 task(s) trace-example: 1 hard, 1 soft.
       Frontend /home/tmplt/.cargo/bin/rtic-scope-frontend-dummy: @2084 µs (+2084 ns): [Task { name: "app::hardware", action: Entered }]
       Frontend /home/tmplt/.cargo/bin/rtic-scope-frontend-dummy: @3584 µs (+1500 ns): [Task { name: "app::software", action: Entered }]
       Frontend /home/tmplt/.cargo/bin/rtic-scope-frontend-dummy: @4001 µs (+417 ns): [Task { name: "app::software", action: Exited }]
       Frontend /home/tmplt/.cargo/bin/rtic-scope-frontend-dummy: @4085 µs (+84 ns): [Task { name: "app::hardware", action: Returned }]
       Frontend /home/tmplt/.cargo/bin/rtic-scope-frontend-dummy: @8252 µs (+4167 ns): [Task { name: "app::hardware", action: Exited }]
      Traced trace-example: 13 packets processed in 9s (~1.4 packets/s; 0 malformed, 0 non-mappable); 2/2 sinks operational.
```
for an RTIC application that defines one software task and one hardware task, where the software task has higher priority.

### Replaying a trace
To list all recorded trace files, execute:
```shell
$ cargo rtic-scope replay --list [--trace-dir]
```
A trace file can then be replayed via
```shell
$ cargo rtic-scope replay <idx> [--trace-dir]
```
or via
```shell
$ cargo rtic-scope replay --trace-file /path/to/trace/file
```

## FAQ
### Why am I getting an erroneous trace?
A current fundamental flag of `cargo rtic-scope trace` is that it expects the *Serial Wire Out* (SWO) pin to be configured before the device boots.
This issue is not necessarily common to all Cortex-M platforms but the pin with SWO functionality must usually be configured before it emits a trace.
This configuration causes a transient period of noise on the pin, incorrectly interpreted as ITM packets, ultimately thrashing the state of the ITM packet decoder.
A solution to this (albeit hacky) is to
1. insert a breakpoint, delay, or a wait-for-pin-pulled-high/low after SWO configuration;
2. start your trace as usual; and
3. continue from the breakpoint.

For example, the PC27 for `atsame51n` must be configured into the alternative mode *M*, after first configuring the trace clock:
```rust,ignore
#[init]
fn init(mut ctx: init::Context) -> (SharedResources, LocalResources, init::Monotonics()) {
    // configure trace clock
    let mut gcc = GenericClockController::with_internal_32kosc(
        ctx.device.GCLK,
        &mut ctx.device.MCLK,
        &mut ctx.device.OSC32KCTRL,
        &mut ctx.device.OSCCTRL,
        &mut ctx.device.NVMCTRL,
    );
    let gclk0 = gcc.gclk0();
    let trace_clk = gcc.cm4_trace(&gclk0).unwrap();

    let freq = trace_clk.freq().0;

    // configure SWO pin; this causes transient noise on the pin.
    let pins = hal::gpio::v2::Pins::new(ctx.device.PORT);
    let _pc27 = pins.pc27.into_mode::<Alternate<M>>();

    // start `cargo rtic-scope trace` before continuing from this point
    cortex_m::asm::bkpt();

    // configure tracing
    cortex_m_rtic_trace::configure(
        &mut ctx.core.DCB,
        &mut ctx.core.TPIU,
        &mut ctx.core.DWT,
        &mut ctx.core.ITM,
        1, // task enter DWT comparator ID
        2, // task exit DWT comparator ID
        &TraceConfiguration {
            delta_timestamps: LocalTimestampOptions::Enabled,
            absolute_timestamps: GlobalTimestampOptions::Disabled,
            timestamp_clk_src: TimestampClkSrc::AsyncTPIU,
            tpiu_freq: freq,
            tpiu_baud: 38400,
            protocol: TraceProtocol::AsyncSWONRZ, // use SWO pin
        },
    )
    .unwrap();

    // remainder of init...
}
```

An alternative is to use the *Embedded Trace Buffer* (ETB) instead of the TPIU (which serializes to the SWO pin).
This peripheral instead writes the trace packets to RAM which a debugger polls.
See the investigation in [#128](https://github.com/rtic-scope/cargo-rtic-scope/issues/128).

## Publications

- [*RTIC Scope — Real-Time Tracing Support for the RTIC RTOS Framework*](https://github.com/tmplt/masters-thesis): a master's thesis on the development and design of RTIC Scope. Will eventually include an application example on a complex control system.

## License and contact
See the respective repositories for non-commercial licenses.

For commercial support and alternative licensing, inquire via <v@tmplt.dev>.
