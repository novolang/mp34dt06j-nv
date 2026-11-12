# mp34dt06j-nv

A driver for ST's MP34DT06J, a MEMS microphone with a pulse-density-modulated
(PDM) output and no control bus, over a board's PDM input. The microphone gets
a clock and sends one bit per clock; the board's PDM block filters and
decimates the bits to 16-bit PCM samples. The driver implements the
`Microphone` and `MicrophoneAsync` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv) over the `PdmIn` and
`PdmInAsync` traits of
[embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv): it
configures the clock and the channel through the handle and passes the samples
on. It does not decimate.

**Hardware verification: NOT VERIFIED.** No board on the bench carries this
chip, and no board package has a PDM input: embedded-hal-nv has `PdmIn` and no
board implements it, and there is no `hal.pdm` hook. The driver has run only
against a double of the handle, in the host suite. The device program below
builds for the nRF52840-DK and the nRF54LM20-DK against a stand-in handle and
has not been flashed.

**The data sheet could not be read in full.** The source is ST's data sheet
DS12698, Rev 2, April 2019 ("MEMS audio sensor omnidirectional digital
microphone"). ST's site would not serve the file and the mirrors answered with
an empty page or a bot check, so the facts below are the ones ST's product page
and the data sheet's search excerpts state: the pins, a supply of 1.6 to 3.6 V,
a clock of 1.2 to 3.25 MHz, a sensitivity of -26 dBFS, a signal to noise ratio
of 64 dB(A), an acoustic overload point of 122.5 dBSPL and a current of 650 uA.
Three things that the timing table would settle are the driver's own choice and
are named in `src/mp34dt06j.nv` and below: the clock edge each setting of the
L/R pin selects, the turn-on time, and the low-power mode's use.

## Install

```sh
novo pkg add mp34dt06j-nv
```

The device program is `examples/record.nv`. Make it the `main` of a package that
depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
```

## The device program

`examples/record.nv` starts the microphone, waits out its start-up time,
records one second of sound at 16 kHz into a buffer of 16,000 samples and
prints the largest magnitude in it (the peak) and the root of the mean of the
squares (the RMS), about once a second:

```
peak 4210 rms 812 of 32768
```

```novo norun:needs-pkg
use embedded_hal.{ PdmIn, PdmConfig }
use dev_audio.{ Microphone }
use mp34dt06j

// The board's PDM block goes here; this stand-in refuses to be configured.
@value
struct NoPdm
    id: Int

impl PdmIn for NoPdm
    fn configure(self, c: PdmConfig) -> Bool [hw]
        false
    fn sample_rate(self) -> Int [hw]
        0
    fn enable(self) -> Bool [hw]
        false
    fn disable(self) [hw]
        pass
    fn receive(self, var buf: Vec[i32; 256], n: Int) -> Int [hw]
        -1
    fn xruns(self) -> Int [hw]
        0

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    // L/R to ground: the left channel, read on the falling edge of a
    // clock of 1.28 MHz, decimated by 80 to 16 kHz.
    let mic = mp34dt06j.attach(NoPdm { id: 0 }, mp34dt06j.config_left())
    let settle_us = mic.mic_start()
    if settle_us < 0
        hal.uart.write("refused\n")
    else
        hal.timer.delay_ms(settle_us / 1000 + 1)
        var buf: Vec[i32; 256] = Vec.new()
        let n = mic.mic_read(var buf, 128)
        if n > 0
            hal.uart.write("read\n")
```

The full program, which fills the buffer of 16,000 samples and prints the peak
and the RMS, is `examples/record.nv`. A board package that depends on this
package builds it as its `main`. `tests/device_build.sh` does so from a scratch
package for the two boards below and for the emulated board, where it runs the
image and checks that the program reports that the PDM block refuses:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 8,280 B | 1,964 B | 34,648 B |
| `nrf54lm20-dk` | 9,632 B | 2,290 B | 34,648 B |

The numbers are from the build's memory report of this program, 2026-11-11,
toolchain 0.24.1. Flash also holds the runtime, the compiler's support library
and the program's tables. The RAM is the second of sound, 32,000 bytes of `i16`
samples in `.bss`, and the arena the runtime reserves whether or not the
program uses it.

**The limit: no board has a PDM input.** The nRF52840 and the nRF54LM20 have a
PDM peripheral, and the nRF52840 DK's and nRF54LM20 DK's board packages do not
drive it: embedded-hal-nv gives `PdmIn` and no board handle, so no board
package implements the trait, and the hal serves no board at all. The program is
written against the `PdmIn` trait and is given `NoPdm`, so on both images
`mic_start` answers -1 and the program prints `refused`. The nRF PDM block
offers PDM clocks of 1.032, 1.067, 1.231, 1.280 and 1.333 MHz and decimation
ratios of 64 and 80, which a board package that gives a handle will map onto
`PdmConfig`. The one argument of `attach` is what changes then.

### The effects

Every function of the driver that reaches the handle declares `[hw]`, and
nothing else. The driver performs no input or output of its own. `attach`
allocates one cell of the arena, because a struct with a generic field cannot be
a `@value`. The module checks at every tier, `embedded` included; the async form
(`host_of` and the `MicrophoneAsync` impl) is `@tier(app)`, because an `async
fn` is refused at the embedded tier.

## What it is

The MP34DT06J is a top-port MEMS microphone with a digital PDM output in a
4 x 3 x 1 mm package. Its pins are VDD, GND, CLK, DOUT and LR. The board drives
CLK; the part drives DOUT for half of the clock period and leaves it high
impedance for the other half, so that two microphones can share one data line,
one with LR low and one with LR high. A PDM signal carries the sound as the
density of ones; a filter that averages and decimates it gives PCM. In the nRF
parts that filter is the PDM peripheral, which hands back signed 16-bit samples.

| Fact | Value | Source |
|---|---|---|
| Supply | 1.6 to 3.6 V | ST's search excerpt of Table 2 |
| Clock, normal mode | 1.2 to 3.25 MHz | the task statement and the excerpt of the timing table |
| Clock below 1.2 MHz | the low-power mode; the driver refuses it | the task statement |
| Sensitivity | -26 dBFS at 94 dB SPL, +-1 dB (ST's page; a distributor gives +-3 dB) | ST's page |
| Signal to noise ratio | 64 dB(A) at 94 dB SPL | the excerpt |
| Acoustic overload point | 122.5 dBSPL | the excerpt |
| Current | 650 uA typical | the excerpt |
| Turn-on time | not read; the driver assumes 10 ms, a field `startup_ms` | the family's order |
| LR low | the left channel, data on the falling clock edge | the family's convention, NOT CHECKED |

The nRF PDM block's default clock of 1.032 MHz is below the part's range. That
is the pitfall of putting this part on an nRF board: the clock has to be 1.231
MHz or more, and the default of this package is 1.28 MHz with a ratio of 80 for
16 kHz, which the block has. Zephyr's `dmic_nrfx_pdm` driver searches the block's
clocks inside a minimum and a maximum from the devicetree for the same reason.

## What the package contains

| Module | What is in it |
|---|---|
| `mp34dt06j` | `Mp34Config` and its constructors, the clock and edge arithmetic, `pdm_start` and `pdm_read`, the type `Mp34dt06j` and its trait, and the async form `Mp34Host` |

## How to choose an entry point

A program on a board builds an `Mp34Config` with `config_left()`,
`config_right()` or `config_pair()`, passes it with the board's `PdmIn` handle
to `attach`, and uses the trait on the result.

| Function | What it does |
|---|---|
| `config_left()` | one microphone, LR low, 16 kHz from a 1.28 MHz clock |
| `config_right()` | one microphone, LR high, read on the rising edge |
| `config_pair()` | two microphones on one line, left with LR low and right with LR high |
| `attach(pdm, cfg)` | the microphone on the handle; nothing is sent |
| `mic_start()` | configures the handle, starts the clock, and answers the microseconds until the output is sound, or -1 |
| `mic_read(buf, n)` | `n` samples, 16-bit and sign-extended, or -1 |
| `mic_stop()`, `mic_lost()` | stops the clock; the handle's overruns since the start |
| `mic_format()` | 16 bits at the configured rate and channels |
| `actual_rate()` | the rate the handle runs at, which differs from the request by its rounding |
| `host_of(pdm, cfg)` | the async form: `mic_capture(n)` parks until the handle has the samples |
| `config_valid(c)`, `decimation(c)`, `clock_ok(hz)`, `edge_of(c)` | the arithmetic as functions |

## The rules a user needs

1. The samples are not sound until `mic_start`'s time has passed. The part
   needs its clock running for its turn-on time and the board's decimation
   filter needs to fill, so the samples before it are settling. ST's examples
   discard the first buffers for the same reason. `mic_start` answers the time;
   the program waits it out before it reads.
2. The clock has to be in the part's range, 1.2 to 3.25 MHz, and at least twice
   the sample rate; it need not be a whole multiple, and the handle runs at the
   rate its decimation gives (`actual_rate`). A configuration outside it is
   refused with -1 and the handle is not touched.
3. A single microphone with LR low is the left channel and is read on the falling
   edge. A single microphone with LR high is read, as the only channel, on the
   rising edge. A pair is read in stereo with the left channel on the falling
   edge; the left sample of a frame is the microphone with LR low. A stereo read
   of an odd count would split a frame and is refused.
4. The handle answers 16-bit samples. `mic_read` refuses `n` above 256, and
   leaves the buffer empty on a refusal.
5. The part's sensitivity is fixed. `mic_set_gain` passes the request to the
   handle's `PdmIn.set_gain`: the nRF PDM block has a digital gain of -20 to
   +20 dB in 0.5 dB steps that Zephyr sets, and the answer is the step at or
   below the request. A handle with no gain stage answers `None`.
6. A sample at full scale means the microphone is past its acoustic overload
   point, 122.5 dB SPL, at the sensitivity above.

## What is not included

- A software decimator from PDM bits to PCM: NOT IMPLEMENTED. A chip with no PDM
  block (the RP2350 does it with a PIO program) needs one in the board's
  handle, behind `PdmIn`.
- The use of the low-power mode and the sleep behaviour of the part.
- The sampling-edge check against the timing table, and the turn-on time of the
  data sheet: both could not be read.

## Where the traits fall short

- `PdmConfig` has no decimation filter choice, so the nRF block's ratio 64
  against 80 is the handle's own.
- `PdmIn.receive` answers 16-bit samples as `Vec[i32; 256]`, one `Int` per
  sample; a second of sound is 62.5 buffers.
- A trait method gets a plain `self`, so the driver keeps no state between calls;
  "not started" is the handle's refusal of a read before `enable`.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the `Microphone`
  and `MicrophoneAsync` traits this driver implements.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names the
  `PdmIn` and `PdmInAsync` traits.
- [inmp441-nv](https://novo-lang.org/packages/inmp441-nv) reads an I2S
  microphone, where the frame is the interface and the samples are 24-bit.

## Tests

`novo test` runs the module over a double of the `PdmIn` and `PdmInAsync` traits
that records what the driver configured and answers a ramp of samples: the
clock range at its edges, the low-power clock, the edge for each strapping of
the L/R pin and for a pair, the settings handed to the handle, the validity of a
configuration, the decimation, the format, the start-up time, the start with
every refusal, the read with its limits (not started, past 256, an odd stereo
count, a short answer), the type and the async form. The suite covers every line
of `src/`; `bash tests/coverage.sh` prints the number.

**Hardware: NOT VERIFIED.** See the device program section.

The sources read for this driver, for their facts and not ported, are the
product page and the data sheet excerpts above, Zephyr's `dmic_nrfx_pdm.c`
(`drivers/audio`), and the descriptions of ST's X-CUBE-MEMSMIC1 examples. The
fixes and differences kept:

- The clock is checked against the part's range before the handle is touched.
  The nRF block's own default, 1.032 MHz, is under it.
- The channel is the clock edge, as in Zephyr: a mono read returns the left
  channel only, and the left and right are exchanged by the edge of the
  sampling, which here comes from the L/R pin.
- The samples are 16 bits only, as Zephyr's driver requires.
- The clock starts at `enable` and the data is not valid until the turn-on time
  has passed, so `mic_start` answers the time and the first samples are not
  used (ST's examples discard them).

## Licence

Apache-2.0.
