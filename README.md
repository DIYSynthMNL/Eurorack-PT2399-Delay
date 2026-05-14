# Eurorack-PT2399-Delay

![PT2399 Delay assembled](photos/front.jpg)

A Eurorack delay module based on the [PT2399](https://www.princeton.com.tw/en-us/Product/IC-Voice-Sound-IC/Echo-Processor-IC/PT2399) — Princeton Technology's classic karaoke-chip digital delay. ~30 ms to ~340 ms delay range with audible degradation as time stretches (the "lo-fi delay" character that made this chip famous in guitar pedals and DIY modules).

This module adds **voltage-controllable delay time** (via a Music-Thing-style BJT current source on pin 6), **2 wet outputs + 1 dry output**, an **Aux input** that's normalled to Wet Out 2 for self-feedback (or for routing external effects into the feedback loop), and a **feedback overdrive jumper** that hard-clips the feedback for chaotic / lo-fi sounds.

## Features

- **Voltage-controllable delay time** — accepts 10 Vpp Eurorack CV, scales internally to the 0–650 mV range the PT2399 wants at pin 6
- **3 outputs:** 2 wet (delayed) + 1 dry (buffered copy of input)
- **Aux input** normalled to Wet Out 2 — patches external signals into the feedback loop
- **Feedback overdrive jumper (JP1)** — closes a hard-clipping path (1N4148 diodes) on the feedback for distorted echoes
- **Anti-latch-up protection** on pin 6 — keeps the chip from getting stuck during power-on (PT2399 latches if pin 6 sees <2 kΩ during the first ~400 ms)
- **±5V regulators** on-board (U2 78M05, U3 79M05) for the PT2399's +5V supply and the local −5V reference
- **TL431 precision reference** (U7) generates a stable −5V_ref for CV scaling math
- Eurorack ±12V power via either 16-pin IDC or 3-pin JST connector

## Inputs and outputs

| Jack | Purpose | Notes |
|---|---|---|
| Main In | Audio in (10 Vpp) | Through RV8 (100K atten) → 100K + 220nF AC couple → input op-amp |
| Aux In | External signal to feedback loop | Normalled to Wet Out 2 when unplugged; mixed into the wet path with RV1 (100K) level |
| Time CV | Delay time CV (10 Vpp) | Through RV2 (100K, "CV amt") + RV3 (100K, "CV level") summing → scaling → BJT current source on PT2399 pin 6 |
| Wet Out 1 | Delayed signal | Aux input is mixed in here |
| Wet Out 2 | Delayed signal (alt) | Normalled to Aux In for feedback loop; JP1 jumper inserts hard-clipping diodes |
| Dry Out | Buffered copy of Main In | Pre-delay, just the input signal |

Front-panel pots:

| Pot | Value | Function |
|---|---|---|
| RV1 | 100K | **Aux input level** |
| RV2 | 100K | **CV amount** (depth of patched Time CV) |
| RV3 | 100K | **Manual delay time** (sets time when no CV is patched) |
| RV8 | 100K | **Main input attenuation** (added in Rev 0.1.1) |

Internal trims (set once during build):

| Trim | Value | Function |
|---|---|---|
| RV4 | 5K | **CV level trim** — scales the CV-to-pin-6-current relationship |
| RV5 | 100K | **Max time trim** — sets the maximum delay (Q1's max base bias) |
| RV6 | 200K | **Output gain trim** (changed from earlier value in Rev 0.1.1) |
| RV7 | 500R | **−5V reference trim** (via TL431LP) |

## Block diagram

```
                          ╔═══════════════ AUDIO PATH ════════════════╗
                          ║                                              ║
Main In ──[RV8 100K atten]──[R10 100K]──┬── [C18 220nF AC couple]──┐    ║
                                         │                          ▼    ║
Aux In  ──[RV1 100K level]──[R8 100K]───┤             [U4A TL074 inverting amp + C22 47pF + R9 39K]
                                         │                          │    ║
                                        (sums)                       ▼    ║
                                                              [U4B TL074 buffer]
                                                                     │    ║
                                                              [C20 1µF AC couple]
                                                                     │    ║
                                                              [R13 10K series]
                                                                     │    ║
                                                                     ▼    ║
                                                              PT2399 pin 16 LPF1-IN
                                         ║                                ║
                                         ║   PT2399 internal:             ║
                                         ║   LPF1 → ADC → digital RAM     ║
                                         ║   → DAC → LPF2 → OP1 → OP2     ║
                                         ║                                ║
                                                              PT2399 pin 9 OP1-OUT
                                                                     │    ║
                                                              [C26 15nF + C27 100nF + R26 1M + RV6 200K]
                                                                     │    ║
                                                                     ▼    ║
                                                              [U4C TL074 output buffer with gain trim]
                                                                     │    ║
                                                                     ▼    ║
                                                              [R27 1K series]
                                                                     │    ║
                                          ┌───── wet out ─────────────┘   ║
                                          │                               ║
Wet Out 1 ◄──[U6A TL072]──[R29 1K]────────┤                               ║
Wet Out 2 ◄──[U5B TL072]──[R30 1K]────────┤                               ║
                                          │                               ║
                          ╚═══════════════ ║ ════════════════════════════╝
                                          ▼
                          Feedback (normalled from Wet Out 2):
                          ──[R28 1M]──┬── JP1 jumper (open = clean fb, closed = hard-clip via D6/D7 1N4148)
                                      │
                                      └── back to U4A input summing

Dry Out ◄──[U4D TL074 buffer]──[R34 1K series]── from input

                          ╔═══════════════ CV PATH ═══════════════════╗
                          ║                                              ║
Time CV in ──[RV2 100K "CV amt"]──[R14 200K]──┐                          ║
                                              │                          ║
−5V_REF ──[RV3 100K "CV level"]──[R15 100K]──┴── U5A TL072 inverting amp + R16 100K fb
                                                            │            ║
                                                            ▼            ║
                                                  [R18 100K series]      ║
                                                            │            ║
                                                            ▼            ║
                                              [U5B TL072 inverter, R17 100K + C24 100pF]
                                                            │            ║
                                                            ▼            ║
                                                  [R20 1K series]        ║
                                                            │            ║
                                                            ▼            ║
                                              [D3 / D4 BAT42 clamps to ±5V]
                                                            │            ║
                                                            ▼            ║
                                                  [RV4 5K "CV level trim"]
                                                            │            ║
                                                            ▼            ║
                                              [Q1 BC547 base bias]
                                                            │            ║
                                                  [RV5 100K "Max time trim"]
                                                            │            ║
                                                  Q1 collector ──► [R22 100R "Anti latch-up"]──► PT2399 pin 6 VCO
                                                                                                          │
                                                                                                          ▼
                                                                                            (Q1 acts as variable resistor
                                                                                             to GND; current sets clock)
                          ║                                                                                ║
                          ║   Q2 BC547 + D5 1N4148 + R23 680K — anti-latch-up timer:                       ║
                          ║   forces pin 6 voltage high during first ~400 ms after power-on,               ║
                          ║   preventing the chip from latching                                            ║
                          ╚════════════════════════════════════════════════════════════════════════════════╝
```

## Power

- Eurorack ±12V via **J1** (3-pin JST) or **J2** (16-pin IDC) — populate one
- D1 / D2: reverse-polarity protection
- C4 / C8 (22 µF): bulk rail decoupling on ±12V
- **U2 (78M05): +5V regulator** from +12V — supplies PT2399 (pins 1, 18), reference circuits
- **U3 (79M05): −5V regulator** from −12V — supplies the −5V reference rail used by the TL431 + RV7 network
- **U7 (TL431LP) + RV7 (500R) + R1 / R2 (10K* "optional load resistor")** — precision negative-5V reference for CV scaling math
- Each op-amp has 100 nF decoupling near its V+ / V− pins (C29 / C30 / C31)

The PT2399 itself is a +5V single-supply chip — only the Eurorack op-amps need ±12V. The local ±5V regulators isolate the chip from any rail noise.

## Calibration

Four internal trims, in this order. With the module warmed up ~5 minutes:

**RV7 — −5V reference trim.** Adjust until the −5V_REF net reads **−5.00 V** on a DMM. This is the foundation for the CV scaling math.

**RV6 — Output gain trim.** With a known-amplitude signal at the Main In, scope Wet Out 1 / 2. Adjust RV6 until wet output level matches dry input level (unity gain).

**RV5 — Max time trim.** With no Time CV patched and the front-panel CV-level pot (RV3) at its maximum delay setting (fully CW or CCW depending on polarity), adjust RV5 to set the maximum delay time you want at the upper limit. Going beyond ~340 ms aggravates aliasing and clock noise — choose a max that sounds good for your patch.

**RV4 — CV level trim.** Patch a known CV (e.g., 0–5 V LFO) into Time CV. Adjust RV4 so the CV's full swing produces a musically useful delay range without forcing pin 6 voltage outside ~0 to 650 mV (the chip's optimal range — silkscreen label "0 to 650mV optimal").

If the chip latches up at power-on (no audio passes through despite all rails being good), the anti-latch-up circuit (Q2 + R23 + D5) isn't holding pin 6 voltage high long enough — see [issue #2](../../issues).

## Design notes

### PT2399 application-circuit values (from [Electrosmash analysis](https://www.electrosmash.com/pt2399-analysis))

| Network | Datasheet typical | This design |
|---|---|---|
| LPF1 (input filter, MFB) | R = 15K / 10K / 15K, C = 3.9 nF / 0.56 nF (fc ≈ 8.8 kHz, "delay" variant) | R3 = 10K, R5 = 47K, C16 = 1 nF (different topology — verify on bench) |
| LPF2 (output filter, MFB) | R = 15K / 10K / 15K, C = 5.6 nF / 0.56 nF (fc ≈ 8.9 kHz, "echo" variant) | R4 = 10K, R7 = 47K, C15 = 15 nF (similar pattern) |
| OP1 / OP2 compensation | 100 nF on pins 9 / 10 / 11 / 12 | C16 = 1 nF (pins 9/10), C14 = 100 nF, C13 = 100 nF, C11 = 100 nF, C12 = 100 nF |
| REF pin (2) | Cap to GND for noise filtering | C17 = 470 pF |
| AGND / DGND | Tied together with short heavy trace | Both → GND (verify the layout has thick traces between pins 3 and 4) |
| VCC decoupling | 100 nF + bulk | C10 = 100 nF + C9 = 1 µF on +5V |
| Pin 6 (VCO) external R | Fixed resistor to GND, e.g., 27.6K → 342 ms delay | **Q1 BC547 BJT acting as voltage-controlled variable resistor** + R22 (100 Ω) series + RV5 (100K) max-time trim |

### Cross-cutting features Rex added on top of cited sources

The README cites Echobase pedal, Benjiaomodular Mini Delay, and Electrosmash's analysis. Features here that go beyond those sources:

- **2 wet outputs + 1 dry output** (most PT2399 designs have one wet, sometimes one dry — two wets enables parallel processing)
- **Aux input normalled to Wet Out 2** — a clever performer feature. Default: feedback loop. Patch in: external signal replaces feedback (creating crazy chained-effect feedback paths)
- **CV-controlled delay time** via Q1 BJT current source — the Music Thing Modular pattern, but with a full ±12V Eurorack CV input chain on the front end (CV amt + CV level + RV4 trim + bipolar reference)
- **Feedback overdrive jumper (JP1)** — Echobase-derived hard-clipping feedback path that adds chaotic / lo-fi character

### Component-value oddity to verify (Rev 0.1.1 note)

The Rev 0.1.1 schematic title-block note says *"Change feedback resistor R24 to 220K."* — but the schematic component value for R24 still reads **100K**. Likely an undocumented mismatch between the revision note and the actual value applied. **Verify on the bench whether R24 is 100K (as drawn) or 220K (as noted), and update the schematic to match.**

## References

Local archived copies live in [`references/`](references/) so this repo stays useful if the upstream links die.

- **PT2399 datasheet (Princeton Technology Corp)** — [upstream](http://www.princeton.com.tw/Portals/0/Product/PT2399_1.pdf) (Princeton's CDN times out from many networks; save manually if you want a local archive)
- **Electrosmash PT2399 analysis** — [upstream](https://www.electrosmash.com/pt2399-analysis) — the definitive online deep-dive on this chip; has the chip's pin assignments, internal block diagram, and typical application circuit values. Worth saving the page to PDF for offline use.
- Design inspiration: **Echobase pedal** schematic (commercial; check [Build Your Own Clone forum](https://buildyourownclone.com) for derivatives)
- Design inspiration: **Benjiaomodular Mini Delay** Eurorack module (commercial)
- Design inspiration: **Music Thing Modular Spring Reverb** (uses similar BJT-current-source delay-time control: [www.musicthing.co.uk/Spring-Reverb](https://www.musicthing.co.uk/Spring-Reverb/))

## Build status

What's ready for builders today, and what's still on the TODO list:

**Production assets** (what you need to actually fabricate and assemble a final unit)

- [x] Schematic — Rev 0.1.1 ([Eurorack-PT2399-Delay-Rev0.1.1.pdf](schematic%20pdfs/Eurorack-PT2399-Delay-Rev0.1.1.pdf))
- [ ] PCB layout — in progress — single working layout in `kicad/`, not yet separated for fab
- [ ] Gerber files for fabrication — none yet
- [ ] BOM — none yet
- [ ] Final front panel (SVG/PDF for fab) — none yet
- [ ] License — none yet

**Prototype assets** (for breadboard / perfboard / 3D-printed-panel builds before final PCB)

- [x] 3D-printed prototype panel STL — [2399_delay.stl](3D%20printed%20front%20panel/2399_delay.stl)
- [x] Falstad simulations — [falstad/](falstad/)

**Documentation**

- [x] Photos of the assembled module — see [photos/](photos/)
- [ ] Demo video — none yet
- [ ] Build / assembly instructions — none yet
- [x] Calibration procedure — see Calibration section above

Want to help fill a gap (build photos, gerbers, an assembly guide)? Open an issue or PR.
