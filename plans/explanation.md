🎓 SOCRATIC SESSION: Page 1, The Description — Sentence by Sentence
A. Calibration Check
Reading method first, because it transfers: datasheet descriptions are not marketing. They are written by the applications engineer, and each sentence exists because it constrains something. As we go, keep asking "what does this forbid me from doing?" rather than "what does this let me do?"

Also note the verbs. Twice the paragraph says the module "allows" something. It never says the module does it. That's precise, honest phrasing, and it's telling you where the work lives — in your firmware.

B. The Nine Sentences
1. "The K-LC7 is a small and low cost radar module which allows angle measurements."
Two things hiding here.

"low cost" is a real spec. It tells you there is no per-unit calibration data, no factory characterization sheet, and loose tolerances — which you can confirm on page 2, where I/Q phase shift is spec'd 80 | 90 | 100°. A ±10° manufacturing spread on the quantity you're trying to measure to a degree. You will have to calibrate this yourself. Your plan's last Known Risk already anticipates this; the Description is where the datasheet first admits it.

"allows angle measurements" — and here is the distinction that trips up nearly every beginner:

What sets it	K-LC7 value
Angular resolution — can I tell two targets apart?	beamwidth	80° (awful)
Angular precision — where is this one target?	SNR + phase measurement	~1° achievable
A 2-element interferometer gives you precision, not resolution. If two people stand in the beam at once, the two Rx channels see the vector sum of both echoes, and the phase difference you compute points at neither of them — it points somewhere in between, weighted by their amplitudes.

This has a direct consequence for your plan file: Phase 5's "multi-target handling" cannot be solved in the angle domain. You separate targets by range bin or Doppler bin first, then compute an angle within each bin. That's why Phase 4 in your plan correctly says "for a detected target at a given range bin" — that ordering isn't a stylistic choice, it's forced by this sentence.

2. "...operating in the 24.0 GHz to 24.25 GHz ISM band..."
ISM = Industrial, Scientific, Medical — license-free, worldwide. You may operate this on your desk legally. Good.

But read it as a constraint: the band is 250 MHz wide, and that is a legal ceiling on your FMCW sweep bandwidth. Range resolution in FMCW is

$$\Delta R = \frac{c}{2B} = \frac{3\times10^8}{2 \times 250\times10^6} = 0.6\ \text{m}$$

That is the best case, at the legal limit. It is not a limitation of your STM32, your FFT, or your DAC — no firmware cleverness beats it. Two targets closer than ~60 cm apart in range merge into one peak.

Now cross-reference page 2: Frequency tuning range: 200 | 250 | 350 MHz. The VCO can be pushed to 350 MHz of swing — wider than the band you're allowed to occupy. The datasheet is quietly telling you that you can drive this thing out of compliance. That's also why Spurious emissions: −30 dBm, According to ETSI 300 440 appears in the table: staying legal is your responsibility once you take control of VCO_In.

Set expectations for Phase 3 now: you are building a ~0.6–0.8 m resolution ranger. Excellent for "person at 3 m vs person at 8 m." Useless for "is the box 12 cm or 15 cm thick."

3. "...it has a built in low phase noise VCO..."
VCO = Voltage Controlled Oscillator. It's the source of everything — the thing that generates 24 GHz, whose frequency you steer with VCO_In.

Phase noise is the part worth understanding. A real oscillator isn't a perfect tone; its phase jitters, smearing the tone into a peak with skirts. Why a radar cares: your mixer compares the echo against the current LO. Noise that's slow compared to the round-trip time is common to both and cancels (the "range correlation" effect) — which is why short-range radar tolerates cheap oscillators at all. Noise faster than that does not cancel, and it lands right on top of weak returns.

You can see this claim quantified on page 2, and this is my favorite cross-reference in the document:


IF noise power   fIF = 500 Hz   →  -134 dBm/Hz
                 fIF =  1 MHz   →  -164 dBm/Hz
Thirty decibels worse at 500 Hz than at 1 MHz. That's a factor of 1000 in noise power, and it is largely the phase-noise skirt plus 1/f flicker noise.

Now recall from last session that a person walking at 1 m/s produces a ~160 Hz Doppler tone. Your entire Phase 2 signal of interest sits deep inside the noisiest part of the spectrum this module has. That's not a defect — it's physics, and it's why CW Doppler motion detectors need real gain and tight filtering rather than a naive threshold. It also means: slower target → lower frequency → worse noise floor. Detecting a slowly-drifting hand is genuinely harder than detecting a fast one, for a reason you can now point at in the table.

4. "...which makes the module suitable for FSK or FMCW applications."
"Suitable for" = because the VCO is externally tunable. A fixed-frequency module could only do CW Doppler. This sentence is the entire justification for pin X2,6 existing.

Two modulation schemes, and the datasheet lists them in increasing order of difficulty for you:

FMCW (your Phase 3): sweep the frequency linearly and continuously. Beat frequency ∝ range. Works on stationary targets. Requires a linear ramp — and your plan's Known Risks correctly flags that a non-linear sweep smears the range FFT.
FSK: hop between just two discrete frequencies, $f_1$ and $f_2$. Measure the Doppler tone's phase at each. The phase difference between them gives range, because $\varphi = 4\pi R/\lambda$ and you just changed λ by a known amount.
Look at the Applications bullet on the same page: "Ranging detection of moving objects using FSK." That qualifier is the catch — FSK needs a Doppler signal to measure the phase of, so it only ranges things that move.

Here's why I'm dwelling on it: FSK is dramatically easier on an STM32 than FMCW. Two DAC output levels versus a calibrated linear ramp. No sweep linearity problem. No ramp-rate-vs-VCO-settling tradeoff. It reuses your entire Phase 2 Doppler pipeline and just adds a phase comparison. If Phase 3 stalls on ramp linearity, this is your fallback path to a range number — and it's sitting right there in the Description as a hint from the manufacturer.

5. "The internal temperature compensating circuit keeps the output frequency stable over a wide temperature range."
Quantified on page 2 as ∆fTX = 0.1 MHz/°C, with Note 2: "Transmit frequency stays within 24.000 to 24.250 GHz over the specified temperature range when the VCO pin is left open."

Ask the constraint question — what does drift actually break?

CW Doppler (Phase 2): nothing. You measure a difference frequency; the absolute carrier cancels out.
FMCW (Phase 3): almost nothing. Range depends on the slope of the sweep, not where it started.
AoA (Phase 4): λ appears in your angle formula. Full band edge to edge is 250 MHz on 24.125 GHz — about 1%. Your 0.2257 constant is good to 1%. Ignorable next to your 3 dB / ±10° I/Q imbalance.
Legal compliance: ← this is what it protects. Note 2 is a regulatory statement: leave the VCO pin open and we guarantee you stay inside the ISM band from −20 to +85 °C.
The lesson generalizes: when a datasheet brags about stability, work out which of your measurements is actually sensitive to it. Often the answer is "none of them, it's there for someone else's use case."

6. "IF outputs I and Q allow movement direction detection and high performance signal processing."
We covered the mechanism last session — $\cos$ alone can't sign a phase, so a 90° hybrid gives you $\sin$ too, and $I + jQ = e^{j\varphi}$ rotates one way for approach and the other for recession.

What this sentence adds: it's the reason there are four analog output pins rather than two. Two receivers × (I, Q) = 4. That's your whole X2 connector minus power and VCO_In, and it's why your Phase 4 needs four simultaneously-sampled ADC channels.

"High performance signal processing" is doing quiet work too — a complex baseband stream is what lets you run a complex FFT with a signed frequency axis, rather than a real FFT whose spectrum is symmetric and therefore direction-blind.

7. "The sensor has a 3 x 4 patch radar frontend with an asymmetrical beam."
A patch antenna is just a metal rectangle on the PCB, roughly λ/2 on a side — at 24 GHz, about 6 mm. Cheap, flat, printed. "3 × 4" means an array of 12 of them, and you can see them in the outline drawing on page 6 as the Tx, Rx1, Rx2 blocks.

"Asymmetrical" is the load-bearing word, and it's the direct cause of the two beamwidth numbers on page 2. See section C below — this is worth doing properly.

Why you'd want asymmetry: picture the module on a wall. Horizontally you want to cover the whole room. Vertically you want to avoid the floor and ceiling, because every reflective surface in the beam contributes clutter. A wide-flat beam is the right shape for a room. It's also why your Phase 5 mechanical-scanning idea should scan in azimuth — the axis where the beam is worst.

8. "The built-in voltage regulator covers a wide power supply range from 3.2 to 5.5V."
Convenient — feed it 3.3 V or 5 V, no precision rail required. But three consequences:

There's an LDO inside, and an LDO has finite power-supply rejection. That's the origin of Supply rejection: 25 dB on page 2. Only 25 dB between your Vcc and your microvolt-level IF outputs. Feed it from a Nucleo's switching-regulator-derived rail and that ripple walks straight into your signal.
The 3.2 V floor is a hard floor, not a suggestion — below it the internal regulator drops out and the VCO's frequency stops being trustworthy. Batteries and long thin breadboard jumpers at 90 mA both threaten this.
An LDO burns the difference as heat. At 5 V and 90 mA, some of that 450 mW is dissipated as waste in a small module. Not dangerous, but it's why the module runs warm, and — given sentence 5 — warm means slightly drifted. Prefer 3.3 V if you have it.
9. "The module provides a frequency divided output which can be used to measure the output frequency of the VCO."
$f_{TX} / 8192$, so $24.125\ \text{GHz} / 8192 = 2.945\ \text{MHz}$ — a digital signal your STM32 can count directly.

Your plan file currently lists this pin as "for monitoring only." I'd push back on that phrasing. This is the module's calibration and feedback port, and it is the answer to your own Known Risk about ramp linearity: it lets you measure what frequency the VCO actually reached for a given DAC code, instead of trusting the typical 80 MHz/V figure. That converts an open-loop assumption into a measured lookup table.

One gotcha to note now, from page 2: Divider output voltage: 1.5 Vpp. Think about what that means meeting a 3.3 V logic input's threshold.

C. Illustrative Example: Where "3 × 4" Becomes "80° × 34°"
Here's the payoff for reading sentence 7 carefully — a page-1 word and two page-2 numbers are the same fact.

Toy setup, nothing to do with the K-LC7. Imagine a row of $N$ speakers, spaced distance $d$ apart, all playing the same tone in phase. Stand directly in front: every speaker is equidistant, all signals arrive in phase, they add — loud. Now walk sideways. Each speaker is now slightly farther than its neighbor, so their signals arrive progressively delayed, and they start cancelling. The angle at which they cancel is roughly where the end-to-end path difference across the whole row reaches one wavelength:

$$L \sin\theta \approx \lambda, \qquad L = N d$$

The physical intuition: a longer array cancels sooner, so a longer array has a narrower beam. The rule of thumb is

$$\theta_{3dB} \approx \frac{51°}{L/\lambda}$$

Apply it to a 3-wide, 4-tall array with roughly one-wavelength spacing:


Horizontal:  3 elements  →  L/λ ≈ 3   →  51/3  ≈  17°  per element-row... 
                              short array  →  WIDE beam   →  datasheet says 80°

Vertical:    4 elements  →  L/λ ≈ 4   →  longer array
                              →  NARROWER beam            →  datasheet says 34°
The absolute numbers won't match exactly — real patch spacing isn't a full λ, patches have their own element pattern, and there's amplitude tapering — but the ratio does the work: more elements along an axis ⇒ narrower beam along that axis. 4 > 3 vertically, so 34° < 80°. The asymmetry in the word on page 1 and the asymmetry in the numbers on page 2 are one physical fact, stated twice.

The transferable skill: whenever a datasheet gives you an array geometry and a beamwidth, check that they're consistent. If they aren't, you've misunderstood which axis is which — and on this module, mixing up azimuth and elevation would silently ruin your Phase 4 angle measurements by mounting the thing rotated 90°.

D. Check for Understanding
Sentence 2 caps you at 250 MHz of sweep, giving $\Delta R \approx 0.6$ m. Your STM32 DAC tops out at 3.3 V, and VCO sensitivity is 80 MHz/V. Are you actually limited by the law here, or by your DAC? Compute both and tell me which binds first — and what a cheap op-amp could change about that answer.

Sentence 3 says the noise floor is 30 dB worse at 500 Hz than at 1 MHz, and sentence 4 offers FMCW as an alternative to CW Doppler. In FMCW, roughly where in the spectrum does a target's beat tone land? Given that, does switching from CW to FMCW help you with the phase-noise problem, hurt, or neither?

Sentence 1 says angular resolution is set by the 80° beamwidth, not by the two receivers. So what does moving to a 4-receiver module actually buy you — better resolution, better precision, or something else entirely? (This one is the crux of whether your Phase 5 upgrade path is worth it.)

E. Your Turn
Before we move on to page 2, one small exercise on the page we just read:

Go back through the nine sentences and write down, for each one, the single row on page 2's Characteristics table that quantifies it. Some map cleanly to one row; at least two map to multiple rows; and I'd claim one sentence has a consequence in the table that isn't obviously connected to it at all (sentence 8 → supply rejection is the example I already gave you — find whether there are others).

The point isn't bookkeeping. It's that once you've done this, page 2 stops being an intimidating wall of symbols and becomes nine familiar ideas with numbers attached. That's the reading strategy for every datasheet you'll ever open.

When you bring that back, we'll go straight into the Block Diagram (Fig. 1) and then the Characteristics table proper — the Block Diagram is only ten boxes, and you can now predict almost all of them from what you just read.

Professor's Query: That's the concept — over to you. Try applying it, and bring me what you get (working or broken, both are useful). When you're ready to move from understanding to building, just say so and we'll switch back to developer mode.