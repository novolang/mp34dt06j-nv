# Changelog

## 0.1.1 — 2026-11-12

Moves off its stand-ins onto devices-nv 0.4.0 and embedded-hal-nv 0.4.0. The release depends on both.

- `mic_set_gain` passes the request to `PdmIn.set_gain` and answers the step the
  handle took, where 0.1.0 answered `None`.
- `mic_format` reports the framing `AudioPdm` and `slot_bits: 0`, where 0.1.0
  reported `AudioPhilips`.
- A clock that is not a whole multiple of the rate is accepted so long as it is
  at least twice the rate; `decimation` rounds down. 0.1.0 refused it.

## 0.1.0

The first release: a driver for ST's MP34DT06J, a MEMS microphone with a PDM
output and no control bus, over a board's PDM input.

- `Mp34dt06j`, the type `attach` builds over a board's `PdmIn` handle,
  implementing devices-nv's `Microphone`: `mic_start` configures the handle with
  the clock, the rate, the channels and the clock edge, starts the clock and
  answers the start-up time in microseconds; `mic_read` answers 16-bit samples;
  `mic_stop`, `mic_lost` and `mic_format` complete the trait. `mic_set_gain`
  keeps the trait's body, `None`, because `PdmIn` exposes no gain.
- `Mp34Config` and its constructors `config_left`, `config_right` and
  `config_pair`: the clock (1.28 MHz by default, a ratio of 80 to 16 kHz), the
  rate, one or two channels, the L/R strapping of a single microphone, and the
  start-up time (10 ms by default).
- The link arithmetic as functions over `Mp34Config`: the clock range of 1.2 to
  3.25 MHz and the low-power clock below it, the edge each strapping of the L/R
  pin is read on, the settings handed to the handle, the validity of a
  configuration, the decimation ratio, the format and the start-up time.
- `pdm_start` and `pdm_read`, the start and the read over any `PdmIn` handle: a
  clock outside the range is refused before the handle is touched, a read past
  256 samples, or of an odd count from a pair, is refused, and the buffer is
  empty on a refusal.
- The async form `host_of` and `Mp34Host`, implementing `MicrophoneAsync` over
  `PdmInAsync` for the host scheduler; they are `@tier(app)`.
- The suite runs over a double of the `PdmIn` and `PdmInAsync` handle and covers
  every line of `src/`.
- The device program `examples/record.nv`, which records a second of sound into a
  buffer and prints its peak and RMS. Built for the nRF52840-DK it is 8,280 B of
  flash and 34,648 B of RAM, and for the nRF54LM20-DK 9,632 B and 34,648 B.

### Known

- Hardware verification: NOT VERIFIED. No board carries an MP34DT06J, and no
  board package implements `PdmIn`; the nRF52840 DK and the nRF54LM20 DK have a
  PDM peripheral that no board package drives. The device program is given a
  stand-in handle that refuses to be configured, and prints `refused`.
- The data sheet, DS12698 Rev 2, April 2019, could not be fetched in full. The
  clock range, the supply, the sensitivity, the signal to noise ratio, the
  overload point and the current are from ST's page and the data sheet's search
  excerpts; the clock edge of each L/R strapping and the turn-on time are the
  driver's own choice and are not checked against the timing table.
- The manifest names devices-nv and embedded-hal-nv by the registry range
  `^0.3.3`, the first releases with `Microphone` and `PdmIn`.
- Not implemented: a software decimator from PDM bits to PCM, the gain, the
  low-power mode of the part, and the checks against the timing table.
