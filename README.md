# pitch

Monophonic pitch detection for [sysl](https://github.com/sysl-lang/sysl): YIN, in plain time-domain
arithmetic, generic over the float width — and the high-pass filter and note arithmetic a tuner wraps
around it.

```
sh/sysl/pitch/
    detector.sysl         the detector: the difference function, its normalisation, the dip and its vertex
    highpass.sysl         a streaming high-pass filter to put ahead of it: cascaded cookbook biquads
    note.sysl             frequency to the nearest equal-tempered note and back, and cents between two
    tests.sysl            what the detector and the note helpers claim, run by `sysl test .`
    highpass_tests.sysl   what the filter claims, and that it is what lets a low E through rumble
package.hocon             who this package is, and what it needs of the machine
```

The module is **`sh.sysl.pitch`**. It depends on nothing outside the standard library's `sysl.math`.

## Using it

Name it in your project's `package.hocon` and `sysl build` fetches it:

```
dependencies {
  pitch { git = "github.com/sysl-lang/pitch", version = "0.1.1" }
}
```

```
import sh.sysl.pitch.{detector, highpass, note_of}

var hp = highpass(f32(48000.0), f32(65.0)).expect("a cutoff below the low E")
var d = detector(f32(48000.0), f32(70.0), f32(1400.0)).expect("a guitar's range")

// each buffer from the audio device, whatever its size, before it joins the sliding window
hp.process(buffer)

// frame: the latest 2048 filtered samples -- at least d.frame_len()
d.detect(frame).expect("a long enough frame") match
    Some(e) ->
        val n = note_of(e.frequency).expect("a positive frequency")
        print(f"${n.label()} ${n.cents}%.1f cents (clarity ${e.clarity}%.2f)")
    None -> print("--")
```

| | |
|---|---|
| `detector(rate, lowest, highest) -> Result[Detector[F], Fault]` | one per stream; refuses a rate or range it cannot measure |
| `d.detect(samples) -> Result[Option[Estimate[F]], Fault]` | `Err(TooShort(n))` below `d.frame_len()`; `None` for silence, noise, or a pitch outside the range |
| `d.threshold`, `d.gate` | the dip a period must reach (default 0.12) and the RMS level below which a frame is silence (default 0.001) |
| `Estimate { frequency, clarity }` | Hz, read to a fraction of a sample; and `1 −` the depth of the dip, 1 for a perfectly periodic frame |
| `note_of(frequency, a4 = 440.0) -> Option[Note[F]]` | the nearest equal-tempered note: `midi`, `cents` in −50..+50, `name()`, `octave()`, `label()` |
| `frequency_of(midi, a4 = 440.0) -> F` | the other way |
| `cents_from(frequency, reference) -> F` | how far a string is from its target, which is what a tuner's needle shows |
| `rms(samples) -> F` | the level the gate is compared against |
| `highpass(rate, cutoff, sections = 2) -> Result[HighPass[F], Fault]` | `BadCutoff` unless 0 < cutoff < rate / 2; `NoSections` for zero |
| `hp.process(samples)` | filters in place, carrying the state on to the next call |
| `hp.reset()`, `hp.sections()` | forget the stream so far; how many second-order sections are in cascade |

## Filtering ahead of the detector

A microphone hears more than the string: handling noise, a room's rumble, a resonance below the
lowest note. YIN measures the period of everything in the frame, so a loud component below the range
drags the difference function with it and the dip at the note's own period never gets deep enough to
count. **For an instrument, high-pass a little below its lowest note: 65 Hz for a guitar's E2.** The
note at 82.4 Hz loses 2.8 dB, rumble at 40 Hz loses 18.

`highpass` is a cascade of identical second-order sections, each the high-pass biquad of Robert
Bristow-Johnson's *Audio EQ Cookbook* with Q = 1/√2, in transposed direct form II. Each section is
−3 dB at the cutoff and falls at 12 dB an octave below it, so the default two are −6 dB at the cutoff
and 24 dB an octave down — a Linkwitz-Riley response rather than a fourth-order Butterworth, and the
same filter as `sox … highpass 65 highpass 65`.

**It streams.** The state carries from one `process` call to the next, so feeding it the buffers an
audio device hands over, of whatever size, gives the same samples bit for bit as one call over the
whole recording. Filter each buffer as it arrives, before it joins the detector's sliding window; one
filter per stream, and `reset` when the stream starts again.

Measured on sines at 48 kHz with a 65 Hz cutoff, against the gain the analog prototype and the
bilinear transform's frequency warping say it should have:

| frequency | expected | measured |
|---|---|---|
| 32.5 Hz, an octave below | −24.6091 dB | agrees to 1e-11 dB |
| 40 Hz | −18.0324 dB | agrees to 1e-11 dB |
| 65 Hz, the cutoff | −6.0206 dB | agrees to 1e-12 dB |
| 1 kHz | −0.00015 dB | agrees to 1e-12 dB |

In `highpass_tests.sysl`, a synthetic low E — five partials under a 40 Hz rumble louder than any of
them and an 80 Hz resonance — gives no reading at all in any of eight frames unfiltered, and reads E2
in all eight through the filter, the worst 3.1 cents out.

## Why YIN, and why not through the FFT

A plucked string's strongest partial is often its second harmonic, so the loudest bin of a spectrum is
an octave above the note. YIN does not ask which frequency is loudest — it asks at what lag the
waveform repeats, by measuring how much the frame differs from itself shifted by each lag, normalising
that by its running mean, and taking the first dip below a threshold. The fundamental is the period
whatever the balance of the partials, and `tests.sysl` checks it on a tone whose second harmonic is
two and a half times its fundamental.

The difference function is summed directly — for a 70 Hz floor at 48 kHz and a 2048-sample frame,
about 940 thousand multiply-adds a frame, which a Cortex-M33 does well inside a tuner's frame period.
It can be computed through an FFT in O(n log n) instead; that would make this package depend on `fft`
and on a power-of-two scratch buffer, and is left out until a consumer needs the speed.

## How close it reads

Measured at 48 kHz over 2048 samples, 70–1400 Hz, against the frequency each tone was built from:

| tone | worst error at the six open strings |
|---|---|
| sine | 0.002 cents |
| a decaying pluck, second harmonic dominant | 0.07 cents |
| a band-limited sawtooth, every harmonic to Nyquist | 0.65 cents |

The sub-sample period comes from a parabola through the raw difference at the dip and its two
neighbours, which is what the YIN paper prescribes. A full-band sawtooth high in the range has a dip
only a few samples wide, so the parabola fits it less well — 1.9 cents at 1318 Hz. Noise 36 dB under a
110 Hz tone costs 0.02 cents; at 16 dB, the three-lag parabola is pulled by the noise and the reading
wanders by tens of cents, which is the point to average frames rather than trust one.

## What allocates and what does not

`detector` sizes the buffer the difference function is written into from the lowest frequency, and
`highpass` its state from the number of sections, so the package declares `heap = true`. **`detect`
and `process` allocate nothing**, frame after frame. There are no
module-level bindings outside the test file, so nothing here needs an initializer — which is what lets
it go into a `sysl build-c` archive for a freestanding target.

## What is not here, and why

- **MPM (McLeod's normalised square difference)** — a close cousin of YIN with a different
  normalisation and peak-picking rule. One detector that is well tested beats two that are half
  tested; it is the natural second one if a consumer finds YIN's octave behaviour wrong for them.
- **The FFT-accelerated difference function** — see above.
- **Polyphony** — this is a monophonic detector; a chord has no single period.
- **Flats in note names** — `name()` spells every accidental as a sharp. A tuner shows one name per
  note and a key signature is not something a frequency carries.

## What it checks itself against

Every tone in `tests.sysl` is synthesized from a frequency chosen in advance — the equal-tempered
open strings, a period of exactly 200.5 samples, frequencies above and below the range — so every
estimate is checked against the number the signal was built from. The note helpers are checked against
the equal-tempered table (A4 = 440 Hz = MIDI 69, middle C = 261.6256 Hz = MIDI 60). Each bound was set
at about ten times the measured error, and each was broken on purpose before release: removing the
interpolation, the cumulative normalisation, the threshold, the gate or the range check turns tests
red. The filter is checked against its transfer function rather than its coefficients, and flipping
a coefficient's sign, dropping the state between calls or emptying `reset` each turn tests red too.

## License

ISC — see `LICENSE`.
