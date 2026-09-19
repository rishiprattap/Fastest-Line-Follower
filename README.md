# FLF-01 — High-Speed Fast Line Follower
## Product Research & Engineering Development Paper

**Prepared for:** Rishi PS
**Subject hardware:** FLF-01 schematic (rev 1.0, 2026‑08‑26) and Fabrication BOM
**Date of analysis:** 2026‑09‑19
**Author:** Claude (Anthropic), acting as product/EE/ME/embedded research assistant

---

## HOW TO READ THIS DOCUMENT

Every technical claim below is tagged:

- **[VERIFIED]** — confirmed from a manufacturer datasheet, distributor spec sheet, or official product page (link given).
- **[CALC]** — a value I calculated from verified inputs; the formula is shown.
- **[ASSUMPTION]** — an engineering assumption made because no authoritative spec exists; flagged explicitly.
- **[OPINION]** — community/forum-level consensus, included only where it materially affects a decision and clearly marked as such.
- **UNVERIFIED** — used verbatim where a spec could not be confirmed from any reliable source.

I did **not** have access to your competition's specific rulebook (you did not name the competition/venue), so all rules-dependent numbers (max size, weight limits, track geometry) are marked UNVERIFIED and must be checked against your actual event's regulations before you finalize dimensions.

---

# EXECUTIVE SUMMARY

Your current FLF-01 design is a functionally reasonable **v1 "get it moving" platform**: Arduino Nano + TB6612FNG + bare N20 motors + a 16-channel analog line array + an OLED/4-button menu system, running from a 2S LiPo through an XT60 and a Mini-360 buck converter. It will drive, and it will follow a line at low-to-moderate speed. It is **not yet specified precisely enough to be fabricated, and it is not yet a high-speed-capable design** in several concrete, fixable ways:

1. The two most speed-critical parts of the whole robot — the **exact motor** (RPM, gear ratio, torque, encoder resolution) and the **exact 16-channel sensor board** — are both left as open placeholders in your own BOM ("N20 Right", "N20 Left", encoder connectors that may or may not be redundant hardware). Nothing about top speed, cornering ability, or control-loop rate can be finalized until these are locked.
2. There is **no reverse-polarity protection, no fuse, and no low-voltage cutoff** on a bare 2S LiPo feed — this is a real safety and hardware-survival gap, not a nice-to-have.
3. Decoupling is present but under-specified for the actual transient currents two stalled N20 motors can pull through a TB6612FNG.
4. The Arduino Nano/ATmega328P is workable for a first PID robot but becomes the limiting factor once you want an interrupt-driven quadrature encoder loop *and* a 16-channel ADC sensor sweep *and* a >500 Hz control loop simultaneously — this is a real, quantifiable bottleneck (Section 10), not a stylistic preference.
5. Your line sensor is 16 analog channels wired straight into an 8-pin header that only exposes 6 analog-capable Nano pins (A0–A6, with A7 used for the button ladder) — **the schematic as drawn cannot actually read 16 analog channels on a Nano.** This is the single most urgent electrical bug in the design (Section 4 and Section 26).

This paper works through the problem the way a real product-development cycle would: requirements → competitive research → component research with real datasheets and calculations → control system design → mechanical/power/PCB design → BOM/cost → manufacturing → test plan → FMEA → roadmap → a final, evidence-based configuration, and a concrete list of changes to your schematic. Every recommendation distinguishes what is *proven fact*, what is *my calculation*, and what is *still an assumption you need to close out* before committing to PCB fabrication.

---

# 1. INTRODUCTION

A Fast Line Follower (FLF) is a small autonomous ground vehicle that uses an optical sensor array to track a contrasting line (usually black on white or white on black) on a track that includes straights, curves, and often sharp turns, S-curves, and intersections. Competition FLF events score primarily on **lap time**, with disqualification or penalty for losing the line. The engineering problem is therefore not "go as fast as possible" but "maximize the speed at which the robot can still reliably sense, decide, and correct" — a closed-loop control problem bounded by sensor bandwidth, actuator bandwidth, tire grip, and chassis rigidity, not by motor RPM alone.

This paper audits your existing FLF-01 design, researches the component space with real sources, and builds a from-scratch, evidence-based specification for a competition-capable version, while being explicit everywhere the evidence runs out.

---

# 2. PRODUCT REQUIREMENTS DOCUMENT (PRD)

**UNVERIFIED — no specific competition rulebook was provided or identified in this conversation.** The values below are therefore *target engineering requirements* derived from the component research in this paper and from widely-used FLF class conventions (line width 15–25 mm black tape on white, or white tape on black, is close to universal — **[OPINION]**, common across Indian/South-Asian college robotics circuits such as those run by IIT technical festivals, Robocon-adjacent events, and hobbyist competitions, but you must confirm your own event's rule PDF before locking chassis width or sensor span).

| Parameter | Target | Basis |
|---|---|---|
| Robot mass (complete) | ≤ 150 g | [ASSUMPTION] — typical for N20-class FLF builds; keep mass low because torque, not top RPM, is what governs acceleration/deceleration authority |
| Footprint | ≤ 100 mm (W) × 130 mm (L) | [ASSUMPTION] — must be checked against your event's max-dimension rule |
| Target top speed (straight) | 1.2–2.0 m/s | [CALC] — see Section 6, bounded by motor/wheel combination chosen |
| Minimum controllable creep speed | ~0.15 m/s | [ASSUMPTION] — used for calibration/search behavior |
| Target acceleration (0→top speed) | ≥ 3 m/s² | [ASSUMPTION] — needed to actually reach top speed within a typical 0.3–0.6 m straight segment |
| Braking distance (top speed → line-loss stop) | ≤ 150 mm | [ASSUMPTION] — chassis/tire dependent, verify empirically |
| Minimum practical curve radius handled at speed | ≥ 60 mm at reduced speed, tighter at creep speed | [ASSUMPTION] |
| Line width | 15–20 mm | [ASSUMPTION] — common FLF track convention; confirm with your rulebook |
| Sensor mounting height above track | 5–10 mm | [VERIFIED, generic] — Pololu's QTR-family sensors specify **optimal sensing distance 5 mm, max recommended 50 mm** ([Pololu QTR‑MD‑16A](https://core-electronics.com.au/qtr-md-16a-reflectance-sensor-array-16-channel-8mm-pitch-analog-output.html)) |
| Sensor sampling rate | ≥ 500 Hz full-array read | [CALC], see Section 8 |
| Control-loop frequency | 200–1000 Hz | [ASSUMPTION], see Section 10/11 |
| Encoder resolution (at wheel) | ≥ 200 counts/rev post-gear, quadrature | [ASSUMPTION], balances resolution vs. ISR load, see Section 9 |
| Wheel diameter | 32–42 mm | [ASSUMPTION], candidate range evaluated in Section 14 |
| Battery voltage | 7.4 V nominal (2S LiPo) | Matches your current BOM; **[VERIFIED]** as a real, common part class, see Section 15 |
| Battery capacity | 300–500 mAh | [ASSUMPTION] — enough for practice + multiple runs without excess mass |
| Peak motor current (both motors, stall) | see Section 6 calc | [CALC] |
| Average operating current | see Section 15 calc | [CALC] |
| Operating time per charge | ≥ 15 min of running | [ASSUMPTION] |
| CG location | Low, centered between wheels, slightly toward drive axle | [ASSUMPTION], standard FLF practice — **[OPINION]** |
| Allowable mechanical flex | Chassis deflection under load should not measurably change sensor height (<0.5 mm) | [ASSUMPTION] |

**INFORMATION STILL REQUIRED (flagged again in Section 27):** your competition's official rulebook — track width, line color/contrast, max robot size, max voltage/battery chemistry restrictions (some events ban LiPo), and whether autonomous start/stop is required via IR gate vs. your current pushbutton.

---

# 3. COMPETITIVE / EXISTING DESIGN RESEARCH

**Honesty note up front:** the instructions asked for 8–15 *fully documented* real robots with a complete spec table for each (motors, MCU, sensor, encoder, speed, etc.). In the time available for this research pass, I was not able to find 8–15 individually well-documented, source-linked FLF *robots* (as opposed to generic components) — most public FLF build logs online document components but not complete verified spec tables, and many "fastest line follower" claims on YouTube/forums have no accompanying datasheet-level documentation. Rather than inventing specifics to fill a table, I am stating this gap directly per your own instruction not to hide uncertainty, and logging it in Section 27.

What I *can* state reliably, because it comes directly from the component ecosystem research in Sections 6–10 (all individually sourced there), is the set of **recurring architectural patterns** that the FLF-building community converges on, which is the useful signal your requirements PRD should be built from:

- **Motor class:** N20/N30-family brushed micro-gearmotors with a low gear ratio (1:10 to 1:50) are the dominant choice for *speed-class* FLF robots, while high-ratio N20s (1:100–1:1000) are used for torque-class or maze-solving robots that prioritize precision over speed — **[OPINION]**, inferred from the fact that vendors explicitly market low-ratio N20/N30 variants (e.g., the 1:10, 1000 RPM N30 in Section 6) as "line-follower" motors while high-ratio variants are marketed for slow, high-torque tasks ([thingbits.in](https://rc.thingbits.in/products/6v-1000-rpm-dc-micro-metal-gear-motor-high-speed)).
- **Sensor class:** 8-, 16-, and increasingly higher-count IR reflectance arrays (Pololu QTR family, RoboJunkies/Robokits-style boards, JSumo XLINE) with analog output are the dominant sensing approach for competitive FLFs, over single-photodiode or camera-based approaches, because of low latency and low compute cost — **[VERIFIED]** as products that exist and are explicitly marketed for this purpose (Pololu QTR-MD-16A: [core-electronics.com.au](https://core-electronics.com.au/qtr-md-16a-reflectance-sensor-array-16-channel-8mm-pitch-analog-output.html); JSumo XLINE 16: [robotshop.com](https://www.robotshop.com/es/products/jsumo-xline-16-sensor-array-board)).
- **Driver class:** TB6612FNG and DRV8833 are the two most common small dual-H-bridge drivers in this power class, both **[VERIFIED]** with continuous ratings around 1.0–1.2 A/channel (Section 7).
- **MCU class:** the hobbyist/beginner tier standardizes on ATmega328P (Arduino Nano/Uno), while the speed-focused tier increasingly moves to STM32F4-class ARM Cortex-M4 boards ("Black Pill") for the higher ADC throughput and hardware quadrature-encoder timer inputs — **[VERIFIED]**, both platforms' datasheets are cited in Section 10.

I did **not** rank these against each other with a score, per your instructions; the trade-offs are analyzed in the relevant component sections.

---

# 4. CURRENT DESIGN AUDIT

This section is a direct, critical read of `SCH_FLF-01_1-FLF_Main-Schematic_2026-09-02.png` and `FLF-01_Fabrication_BOM.xlsx`, referencing exact reference designators.

## 4.1 What is correctly designed

- **U1 (Arduino Nano) pin allocation is broadly sane**: motor driver control lines (D06–D12), buttons (A7, D13, D4), buzzer (D5), OLED (I2C via the A4/A5/5V/GND header) are on distinct, non-conflicting pins.
- **TB6612FNG (U2) wiring is textbook-correct**: `VM = VBAT`, `VCC = 5V`, `STBY` tied to D12 (so firmware must explicitly enable it — good, this prevents an undefined power-up state), and both grounds (PGND and small-signal GND) are joined at one point, which matches Toshiba's and Pololu's reference wiring ([Pololu TB6612FNG carrier](https://www.pololu.com/product/713)).
- **Decoupling philosophy is present**: C1/C2 (100 nF) close to VBAT and 5V, plus a C3 470 µF bulk cap on VBAT. This is the right *category* of components for a brushed-motor system, even though the values need re-checking (Section 4.3).
- **Reverse-drive braking / four-quadrant control** is available because AI1/AI2 and BI1/BI2 are both routed from the Nano rather than hard-wired, giving you access to TB6612's short-brake mode — useful for high-speed stopping.
- **Button ladder (KEY1–KEY4) on a single ADC pin (A7)** using R3/R4 is a legitimate, low-pin-count technique, and R4 (10 kΩ) as a defined pull-down is correct practice, avoiding a floating input.
- **Buzzer driven through Q1 (2N2222) rather than directly from a GPIO** is correct — GPIOs can't reliably source/sink the current a buzzer needs, and R2 (10 kΩ) as a base pull-down prevents an undefined buzzer state at power-up/reset before firmware initializes D5.

## 4.2 What is incomplete (the biggest problems)

- **U6/U7 ("N20-Right"/"N20-Left") have zero locked electrical or mechanical specification.** "N20 motor" is a *form factor*, not a part. As your own BOM's Design Checks tab already flags, RPM, gear ratio, voltage rating, stall current, and encoder resolution are all open. This one omission blocks: wheel diameter selection, top-speed calculation, driver current-margin calculation, battery C-rating selection, encoder ISR budget, and PID gain scaling. **Nothing downstream can be finalized until this is locked** — Section 6 gives you real, sourced candidates and the calculations to choose between them.
- **ENC-R / ENC-L in the BOM may be phantom line items.** If you buy N20 motors with an *integrated* magnetic encoder (the common Adafruit/Pololu-style part, Section 9), the encoder is not a separate purchasable component — it's wires coming off the motor. Your BOM currently lists them as if they might be separate parts. This must be resolved before ordering, or you will buy hardware you don't need or, worse, omit hardware you do need.
- **U3 (OLED) is specified only as "0.96" I2C OLED" with a warning that pin order is unverified.** 0.96" I2C OLED modules from different vendors do **not** share a standard pin order (some are GND-VCC-SCL-SDA, others VCC-GND-SCL-SDA, others SDA-SCL-VCC-GND) — this is a well-known gotcha, and your own BOM correctly flags it as OPEN, but the schematic doesn't show a specific verified module.
- **H1 (1×8 header) pitch is not specified.** "1×8 header" without a pitch (2.54 mm / 2.0 mm / 1.27 mm) is not a buildable BOM line — this connects directly to the sensor board wiring gap below.

## 4.3 What is electrically questionable

- **Decoupling capacitor placement vs. value mismatch for motor transients.** C1/C2 are 100 nF (good for high-frequency logic noise) and C3 is a single 470 µF bulk cap on VBAT. Two N20 motors under a hard directional reversal (a real, frequent event in aggressive line-following) can each pull several hundred mA to over 1 A in transient stall current (see Section 6 motor candidates — stall currents of 0.35–1.5 A are typical for candidates in this class). A single 470 µF cap gives limited transient buffering for *two* motors switching simultaneously; **[CALC]**: at a nominal 7.4 V rail, 470 µF stores `E = ½CV² = ½ × 470×10⁻⁶ × 7.4² ≈ 12.9 mJ` — enough to smooth small ripples but not to fully arrest a multi-ms current transient from two motors reversing at once without help from battery internal resistance. This isn't necessarily a failure, but it is *undersized relative to best practice* for a two-motor high-current driver board; Pololu's own TB6612 carrier ships with onboard filtering capacitors for exactly this reason.
- **No confirmed voltage rating on C3.** Your BOM explicitly flags this as OPEN ("Voltage rating above maximum VBAT with margin"). A 2S LiPo can reach ~8.4 V fully charged; an electrolytic rated only for, say, 6.3 V or 10 V-with-no-margin is a real failure risk. This must be ≥16 V rated for adequate margin.
- **No separate logic-side bulk capacitance shown near the Nano's 5 V pin beyond C2 (100 nF).** The Mini-360 (U5) output ripple is spec'd by its manufacturer at **30 mV no-load** ([components101.com](https://components101.com/node/2228)) but ripple rises under load and switching transients from PWM'd motors on the same rail can couple in. A single 100 nF is minimal; adding a 10–47 µF bulk cap at the 5 V rail near U1/U3 is standard practice and currently absent.

## 4.4 What is mechanically questionable

- **No visible motor mounting, wheel, or chassis detail exists anywhere in the provided files** (the uploads are a schematic and an electronics BOM only). This paper's Section 13 gives a from-scratch mechanical design, but as of today literally zero mechanical design has been done — this is the largest overall project gap, larger than any single electrical issue.
- **No CG or weight-distribution plan** is implied by anything in the schematic (this is expected at the schematic stage, but it needs to be resolved before PCB outline is finalized, because a poorly balanced PCB position is a common source of front-caster-hop or rear-wheel-slip on fast FLFs).

## 4.5 What is likely to limit speed

- **Sensor channel count vs. Nano analog pin count is the hard ceiling right now** (detailed in 4.7 below) — you cannot run a true 16-channel analog sweep on a stock Nano without a multiplexer, and no multiplexer appears in the schematic or BOM.
- **ATmega328P's ~15 kSPS single-channel ADC** ([usc.edu ADC reference](https://ece-classes.usc.edu/ee459/library/documents/ADC.pdf)) forces a real trade-off between how many sensor channels you read per loop and how fast your control loop can run — quantified in Section 10.

## 4.6 What is likely to cause noise or instability

- Motor return current and logic ground currently share the same ground net with no explicit star-point or separated pour called out in the schematic (schematics don't usually show this, but it needs to be an explicit PCB-layout requirement, captured in Section 16, because H-bridge switching noise injected into the ADC reference/ground is a classic cause of jittery sensor readings on fast line followers).
- No ferrite bead / additional filtering is present between the motor driver's power stage and the sensor/logic supply beyond the Mini-360's own regulation.

## 4.7 What is missing from the schematic — the most urgent issue

**Your schematic wires 16 sensor channels (labels "A0–A7" implied by the header H1, which only exposes 8 pins: `Gnd A3 A2 A1 A0 A6 5v Gnd`) but the Nano only has 8 analog-capable pins total (A0–A7), one of which (A7) is already consumed by the button ladder.** That leaves at most 6–7 free analog pins on the Nano, not 16. As drawn, **there is no way to read a genuine 16-channel analog sensor array from this schematic.** This is the single highest-priority fix (see Section 26, Change #1): you need either (a) an analog multiplexer (e.g., a 74HC4051-style 8:1 mux ×2, or a 16:1 mux) between the sensor board and the Nano's ADC, (b) a digital (RC-time-based) sensor array instead of analog, which only needs one GPIO per channel or can be read with a shift-register/timing trick, or (c) a move to an MCU with more ADC channels (STM32F411 offers up to 16 ADC-capable channels natively — **[VERIFIED]**, Section 10). This single decision cascades into the MCU decision, so it's treated as the anchor decision in Section 11.

## 4.8 What is unsafe

- **No fuse between the XT60 battery input and the rest of the circuit.** A cell short, a motor winding short, or a driver failure currently has no upstream protection — current will keep flowing until the battery, wiring, or a component fails, which is a fire/burn risk with LiPo chemistry specifically (LiPo cells can vent and ignite under sustained short-circuit heating).
- **No reverse-polarity protection.** A 2S LiPo with a standard XT60 is keyed against reverse insertion at the *connector* level, but if a person ever rewires the XT60 pigtail incorrectly, or a different battery/connector convention is used, there is nothing on the PCB to prevent instant destruction of U2, U5, and potentially U1.
- **No low-voltage (over-discharge) cutoff.** LiPo cells degrade rapidly and can become unsafe if discharged below ~3.0 V/cell; nothing in this schematic monitors or protects against that (a buzzer-based low-battery *warning*, driven by firmware reading VBAT through a divider, is trivial to add and currently absent even as a warning, let alone a hard cutoff).

## 4.9 What is missing from the BOM

- Exact motor part number (already flagged in your own Design Checks tab).
- Exact 16-channel sensor board part number/model — **this is not in your BOM at all**, even though the schematic implies its existence via H1. This is a bigger gap than the motor gap, because the sensor board is a *purchased assembly*, not something you spec loose components for.
- A fuse and its holder/footprint.
- A reverse-polarity protection MOSFET or diode.
- Wheels, tires, chassis material, and fasteners — entirely absent, as expected at this stage, but they must appear before a "Fabrication BOM" is truly complete.
- Wire gauge specification for the VBAT/motor power path.

## 4.10 Ambiguous components, unspecified footprints/connectors

- U2 (TB6612FNG): "Module or bare IC — choose one exact footprint" is explicitly flagged OPEN in your BOM; this materially changes PCB footprint and possibly current-carrying trace requirements.
- U3 (OLED): pin order unverified, as discussed.
- H1: pitch unverified.
- Q1 (2N2222): TO‑92 pinout varies by manufacturer (On Semi vs. other fabs do not always agree on E-B-C ordering for 2N2222 clones) — correctly flagged OPEN in your BOM.

## 4.11 Power distribution / grounding / current capacity / connector ratings / thermal / protection — summary table

| Topic | Current state | Verdict |
|---|---:|---|
| Power distribution | Single VBAT rail + single Mini-360-derived 5V rail | Acceptable topology, under-filtered |
| Decoupling | 100 nF ×2 + 470 µF | Undersized for 2-motor transient load, see 4.3 |
| Grounding | Single implied ground net | Needs explicit star/pour separation in PCB layout, see 16 |
| Motor noise | No snubber/flyback-specific components beyond driver's internal protection | TB6612FNG has internal protection diodes; acceptable but add small motor-terminal caps (Section 26) |
| Encoder wiring | Not present at all in current schematic | Must be added once motor is locked |
| Logic-level compatibility | 5 V Nano, 5 V TB6612 VCC — consistent | OK |
| Current capacity (traces) | Not yet a PCB, so N/A yet | Must follow Section 16 trace-width table |
| Connector ratings | XT60 rated 30 A cont / 60 A burst ([probots.co.in](https://probots.co.in/xt60pw-f-female-connector-right-angle-pcb-mount.html)) — wildly oversized for this robot's actual current draw (a few amps) | Fine electrically, but heavy/large for a lightweight FLF — see Section 15 alternative |
| Capacitor voltage ratings | C3 unspecified | Must be ≥16 V, see 4.3 |
| PCB trace requirements | N/A yet | Section 16 |
| Thermal | Not analyzed yet | Section 7/16 |
| Protection circuitry | None | Section 26 Change list |
| Reverse-polarity protection | None | Section 26 |
| Battery protection (fuse) | None | Section 26 |
| Switch/fuse requirements | No power switch or fuse shown; power is controlled only by physically plugging/unplugging XT60 | Add a power switch or resettable fuse for bench safety |

---

# 5. SYSTEM ARCHITECTURE

```
                      ┌───────────────────────────┐
   2S LiPo ──XT60──►  │  Fuse + reverse-protect    │
                      └─────────────┬──────────────┘
                                    │ VBAT (7.4 V nominal)
                     ┌──────────────┼───────────────┐
                     ▼                              ▼
             TB6612FNG (VM)                  Mini-360 buck → 5V (or 3.3V if MCU changes)
                     │                                │
        ┌────────────┴────────────┐                   ├── MCU (Nano or STM32F411)
        ▼                          ▼                   ├── OLED (I2C)
   Left N20 motor            Right N20 motor            ├── Buzzer driver (Q1)
   (+ integrated             (+ integrated               └── Buttons (ladder or discrete)
    magnetic encoder)         magnetic encoder)
        │                          │
        └──────────► Encoder A/B lines → MCU timer/interrupt pins

  16-ch line sensor array ──(mux if Nano / direct if STM32)──► MCU ADC
```

Control data flow: **Sensor array → weighted line-position error → PID (+ derivative filter) → per-wheel PWM command, cross-checked/trimmed by encoder-based wheel-speed feedback → TB6612FNG → motors.** Full detail in Section 11.

---

# 6. MOTOR RESEARCH

This is the most consequential open item in your current design. Below are **8 real, sourced candidates**, spanning the torque↔speed spectrum, followed by the calculations needed to choose correctly.

## 6.1 Candidates

| # | Motor | Mfr/source | Voltage | Gear ratio | No-load RPM | Rated RPM | Stall torque | Stall current | Encoder | Source |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | BJ‑N20‑12‑1000 | generic/SparkFun-hosted datasheet | 12 V | 1:30 | 1000 | 681 @ 0.16 kg·cm | 0.7 kg·cm | 0.43 A | Integrated, 7 PPR (AB square wave) | [VERIFIED](https://cdn.sparkfun.com/assets/a/3/9/9/9/28633_datasheet_N20_Motor_With_Encoder_and_Cable_pair.pdf) |
| 2 | N20 w/ magnetic encoder, 1:50 | Adafruit #4638-class | 4.5–6 V (6 V nom.) | 1:50 | UNVERIFIED exact RPM (typ. ~200 RPM class per vendor family data) | — | — | No-load ~100 mA, stall ~200 mA | Integrated, 14 CPR pre-gear | [VERIFIED current specs](https://littlebirdelectronics.com.au/products/n20-dc-motor-with-magnetic-encoder-6v-with-1-50-gear-ratio.json) |
| 3 | N20 w/ magnetic encoder, 1:100 | Adafruit #4639 | 6 V nom. | 1:100 | UNVERIFIED exact RPM | — | — | ~100 mA / ~200 mA stall | Integrated, 14 CPR pre-gear | [VERIFIED](https://www.adafruit.com/product/4639) |
| 4 | N20 w/ magnetic encoder, 1:298 | Adafruit-class | 6 V nom. | 1:298 | UNVERIFIED exact RPM | — | — | ~100 mA / ~200 mA stall | Integrated, 14 CPR pre-gear | [VERIFIED](https://littlebirdelectronics.com.au/products/n20-dc-motor-with-magnetic-encoder-6v-with-1-298-gear-ratio.json) |
| 5 | N20‑BT01, 75:1 | Botland (Pololu-compatible line) | 1–6 V (opt. 6 V) | 1:75 | 200 @ 6 V | — | 0.8 kg·cm (0.078 N·m) | 50 mA no-load | **None** — vendor explicitly warns N20 is *not* compatible with Pololu's micro-motor encoder kit | [VERIFIED](https://botland.com.pl/en/micro-n20-mp-series-motors-medium-power/12541-micro-motor-n20-bt01-751-220rpm-6v.html) |
| 6 | N20 1000:1 HPCB | Pololu-class high-power carbon-brush | 6 V | 1:986 (marked 1000:1) | 33 RPM | — | 11 kg·cm (extrapolated, vendor warns theoretical) | 1.5 A stall | Encoder-ready N20 form factor, not integrated | [VERIFIED](https://littlebirdelectronics.com.au/products/1000-1-micro-metal-gearmotor-hpcb-6v) |
| 7 | GA12-N20, 1:1000 | zbotic.in | 6 V | 1:1000 | 15 RPM | — | 10 kg·cm stall | 0.67 A | Not integrated | [VERIFIED](https://zbotic.in/product/high-torque-n20-6v-15rpm-micro-dc-metal-gear-reduction-motor-reduction-ratio-11000/) |
| 8 | **12SG‑N30VA‑10** ("6V 1000RPM High Speed", N30 body) | thingbits.in | 6 V | **1:10** | **1000** | 780 @ 16 g·cm | 56 g·cm (0.056 kg·cm) | 0.35 A | Not integrated (N30 form factor) | [VERIFIED](https://rc.thingbits.in/products/6v-1000-rpm-dc-micro-metal-gear-motor-high-speed) — explicitly marketed "especially well suited for line-following robots" |

**Reading the table:** Candidates 6 and 7 (1000:1-class) are torque-class motors — far too slow for a *fast* FLF (15–33 RPM no-load is walking-pace territory even with a large wheel) and are included specifically to show the other end of the trade space you should *not* pick from. Candidates 2–4 (magnetic-encoder N20 family) are attractive because the encoder is free/integrated, but their exact no-load RPM at each ratio was not printed in the scraped vendor text in this research pass — **UNVERIFIED**, and you should pull the exact ratio-vs-RPM table from the product page before buying (the listings reference an on-page "Ratio Comparison Table" that didn't extract as text). Candidates 1 and 8 are the two strongest *speed-class* contenders with hard numeric specs.

## 6.2 Calculations — from motor spec to robot speed

Formula chain used throughout, **[CALC]**, standard drivetrain kinematics:

```
wheel_circumference = π × wheel_diameter
robot_linear_speed  = (motor_RPM_at_wheel / 60) × wheel_circumference
wheel_RPM = motor_shaft_no-load_RPM_after_gearbox   (already includes gear ratio in vendor specs above)
```

Using **candidate 8** (best-documented speed-class motor) with two wheel diameter options:

| Wheel Ø | No-load surface speed | Rated-load surface speed |
|---|---|---|
| 32 mm | (1000/60)×π×0.032 = **1.675 m/s** [CALC] | (780/60)×π×0.032 = **1.307 m/s** [CALC] |
| 42 mm | (1000/60)×π×0.042 = **2.199 m/s** [CALC] | (780/60)×π×0.042 = **1.715 m/s** [CALC] |

**Critical correction per your own instructions — no-load RPM ≠ real robot speed.** The "rated-load" column above is *still the vendor's rated load point*, not your robot's actual load, and does not yet account for:

- **Battery voltage sag under load.** A 2S LiPo under a few-amp draw commonly sags 0.3–0.6 V from nominal — **[ASSUMPTION]**, typical for small 1S/2S packs under partial C-rate load, not independently sourced here; this proportionally reduces motor RPM since brushed DC motor speed is roughly proportional to applied voltage.
- **Wheel slip and tire deformation** — reduces effective rolling radius/traction, especially under hard acceleration or cornering; magnitude depends on tire compound (Section 14), not calculable without empirical tire data.
- **Gearbox mechanical losses** — typically modeled as 70–85% efficiency for small spur gearboxes at these ratios — **[ASSUMPTION]**, standard order-of-magnitude for plastic/metal micro-gearboxes, not vendor-specified for these specific parts.
- **Robot mass and acceleration demand** — instantaneous RPM is lower than the steady-state number while accelerating; matters most on short straights.

**Practical, defensible estimate [CALC + ASSUMPTION stack]:** applying a combined 65–75% real-world derate to the no-load number (covering sag + slip + gearbox loss + partial loading, consistent with how experienced FLF builders describe the gap between "motor datasheet speed" and "robot track speed" — **[OPINION]**) gives:

- Candidate 8, 32 mm wheel: 1.675 × 0.70 ≈ **1.17 m/s** realistic top straight-line speed.
- Candidate 8, 42 mm wheel: 2.199 × 0.70 ≈ **1.54 m/s** realistic top straight-line speed.
- Candidate 1 (1:30, 12 V — but you're running 2S/7.4 V, not 12 V, so this candidate must be re-derated for voltage too): scale no-load RPM roughly linearly with voltage as a first-order approximation — **[ASSUMPTION]**: 1000 RPM × (7.4/12) ≈ 617 RPM no-load at 7.4 V → 32 mm wheel: (617/60)×π×0.032 ≈ 1.03 m/s no-load → ×0.70 derate ≈ **0.72 m/s** realistic. This shows candidate 1 is meaningfully slower than candidate 8 *when both are run from your actual 2S battery*, despite candidate 1's higher no-load RPM spec (which was measured at 12 V, not 7.4 V) — exactly the kind of nominal-spec trap your instructions warned about.

## 6.3 Tractive force / acceleration check

Stall torque is the ceiling on acceleration force. Using candidate 8 (stall torque 56 g·cm = 0.0056 kg·m ≈ 0.055 N·m) at a 32 mm wheel (radius 0.016 m):

```
F_tractive_max = torque / wheel_radius = 0.055 / 0.016 ≈ 3.4 N  per motor at absolute stall [CALC]
```

For a 150 g (0.15 kg) robot, combined stall tractive force from two motors (≈6.9 N, before considering you never actually run at stall) gives a theoretical max acceleration of `a = F/m ≈ 46 m/s²` — **this number is not usable directly** (motors are never run at stall in normal operation, and tire grip will be the real limiter long before motor torque is), but it confirms candidate 8 has generous torque headroom relative to a 150 g chassis, meaning the design is very likely **traction-limited, not torque-limited** — consistent with general FLF engineering consensus **[OPINION]**.

## 6.4 What motor characteristics are actually required

Given the calculations above, the requirement is not "highest no-load RPM" but:

1. **Low gear ratio (≤1:30, ideally ~1:10–1:20)** run near the motor's actual supply voltage (7.4 V, not the datasheet's 12 V test point where applicable) — because gear ratio and voltage both scale surface speed, and low ratio preserves usable torque margin per Section 6.3.
2. **Stall current comfortably under the driver's continuous rating** (see Section 7) with margin for two-motor simultaneous transients.
3. **An integrated (or field-addable) quadrature encoder**, because Section 9/11 show encoder feedback materially improves high-speed controllability, and you're already building your BOM as if one exists.
4. **Wheel diameter chosen as a system, together with the motor**, not independently (Section 14).

**Recommendation for Section 25 final config:** Candidate 8's part family (N30 1:10, 6 V, ~1000 RPM no-load class) is the strongest evidence-backed speed-class choice found, **but it does not ship with an integrated encoder**, so you must either add a field encoder (e.g., an AS5600-based magnetic encoder disc, Section 9) or source an equivalent low-ratio N20/N30 motor that explicitly ships with one — **UNVERIFIED whether such a part exists in the exact 1:10–1:20 range with integrated encoder; this is a live research gap**, logged in Section 27.

---

# 7. MOTOR DRIVER RESEARCH

| Spec | **TB6612FNG** (your current U2) | **DRV8833** (candidate alternative) |
|---|---|---|
| Continuous current | 1.0–1.2 A/channel (avg 1.2 A rated, 1.0 A continuous per Pololu's real-world note on thermal limits) | 1.2 A/channel continuous |
| Peak current | 3.2 A (single pulse, tw=10 ms) / 2 A (20 ms pulse, ≤20% duty) | 2 A/channel peak |
| Voltage range (motor) | VM 2.5–13.5 V (15 V abs max) | ~2.7–10.8 V |
| Logic voltage | VCC 2.7–5.5 V | Onboard logic, no separate VCC needed |
| PWM frequency | Up to 100 kHz | Comparable range (TI datasheet not pulled in this pass for exact max — UNVERIFIED here) |
| Protection | Thermal shutdown, low-voltage detect, reverse-power protection on Pololu carrier | Reverse-voltage, under-voltage, over-current, over-temperature protection built into IC |
| Package | SSOP24, 0.65 mm pitch (fine-pitch hand-soldering is hard) | Similar SMD-class package on carrier boards |
| Availability | Extremely common, your current BOM | Also widely available (Pololu, Adafruit, SparkFun) |
| Price (carrier board) | ~US$4.95 at Pololu | Similar price class (~US$5, Pololu 2130 series) |

Sources: TB6612FNG — [Toshiba datasheet](https://toshiba.semicon-storage.com/info/docget.jsp?did=10660), [Pololu carrier](https://www.pololu.com/product/713). DRV8833 — [TI product page](https://www.ti.com/product/DRV8833), [Pololu DRV8833 carrier](https://core-electronics.com.au/drv8833-dual-motor-driver-carrier.html).

**Engineering trade-off analysis (not "pick the bigger peak number"):**

- Both drivers land in essentially the **same continuous-current class (~1.0–1.2 A/channel)**. Candidate motor 8's stall current is 0.35 A and candidate 1's is 0.43 A — **both drivers have 2–3× margin over even worst-case simultaneous single-motor stall**, so *driver current capacity is not the differentiator* for either motor choice in Section 6.
- TB6612FNG's advantage is that it's **already correctly wired in your schematic** (Section 4.1) — a genuine switching cost exists if you change parts.
- DRV8833's advantage is **built-in current regulation/limiting and reverse-voltage protection on the IC itself**, which reduces (but does not eliminate — Section 26 still recommends discrete reverse-polarity protection at the battery input) some of the protection-circuit gap noted in Section 4.8.
- **Recommendation: keep TB6612FNG.** The calculated current margins (Section 6.3, Section 15) show it is not the bottleneck; changing it would cost redesign effort for no measurable speed or reliability benefit given your motor candidates. This is a case where "the part you already have is fine," which the instructions specifically asked me not to hide.

---

# 8. LINE SENSOR RESEARCH

Your schematic implies a 16-channel analog sensor via H1's labeling. I could not identify a specific "RoboJunkies 16-channel sensor" datasheet with a distinct part number in this research pass (the RoboJunkies/Robokits-style Indian-market 16-channel arrays are commonly sold but the specific current datasheet for that exact branded board is **UNVERIFIED** here). I instead pulled the closest fully-documented, spec-complete equivalent — **Pololu's QTR‑MD‑16A**, which is the de facto reference part for this exact sensor class and is what most "16-channel line sensor" boards (including most RoboJunkies/Robokits-style boards) are functionally cloned from:

**Pololu QTR-MD-16A [VERIFIED](https://core-electronics.com.au/qtr-md-16a-reflectance-sensor-array-16-channel-8mm-pitch-analog-output.html):**
- 16 sensors, 8 mm pitch → 120 mm total sensing width
- Operating voltage 2.9–5.5 V
- Full-brightness LED current 30 mA per channel, **max board current 250 mA** (all 16 LEDs on)
- Analog voltage output, 0–VCC
- Optimal sensing distance 5 mm, max recommended 50 mm
- Dimensions 125.0 × 16.5 × 2.5 mm, weight 7.5 g

Compare to **JSumo XLINE 16** [VERIFIED](https://www.robotshop.com/es/products/jsumo-xline-16-sensor-array-board): 16 sensors at 7 mm pitch (117.8 mm span), 5 V operation, **240 mA total current for all 16 sensors**, 117.8 × 23 × 3.8 mm.

## 8.1 Analog vs. digital arrays

Analog reflectance arrays (as above) output a continuous voltage proportional to reflectance, giving finer position resolution per sensor than a binary digital threshold output, at the cost of needing an ADC channel (or mux + ADC) per sensor and being more sensitive to ambient light drift, requiring calibration. Digital/RC-time-based arrays (Pololu also sells a QTRX-...RC variant) need only a GPIO per channel and are read via a charge/discharge timing method, trading resolution for pin-count and ADC-time savings. **[VERIFIED]** both variant families exist from the same manufacturer family referenced above.

## 8.2 Does your 16-channel architecture suit high-speed operation?

**Partially — the *sensor* is fine; the *interface to your current MCU* is the actual limitation (already flagged in Section 4.7).** Sixteen channels at 8 mm pitch gives ~6–8% of total array width resolved per sensor, materially better cornering/edge-tracking resolution than an 8-channel array of the same width — **[VERIFIED reasoning, generic industry explanation](https://www.aliexpress.com/s/wiki-ssr/article/16-array-ir-sensor)** (used here only for the general engineering logic, not as a primary spec source). The real question is sampling rate:

**[CALC]** — ADC time budget for a full 16-channel analog sweep:
- On ATmega328P: ADC conversion takes **~13 ADC clock cycles per reading** at its usual prescaled clock, giving the commonly cited **~15 kSPS** maximum single-channel throughput ([usc.edu](https://ece-classes.usc.edu/ee459/library/documents/ADC.pdf)). At 15,000 samples/sec, one 16-channel sweep (ignoring mux settling time) takes `16 / 15000 ≈ 1.07 ms`, i.e., a maximum theoretical full-array sample rate of ~937 Hz *if nothing else were happening* — but a mux add-on (needed regardless per 4.7) adds settling-time overhead per channel switch, typically a few µs, which is small relative to the ADC conversion time itself and doesn't change the order of magnitude.
- On STM32F411: **12-bit ADC at up to 2.4 MSPS** with up to 16 channels natively addressable without an external mux ([Zephyr project STM32F411 board doc](https://docs.zephyrproject.org/latest/boards/weact/blackpill_f411ce/doc/index.html); [mischianti.org](https://mischianti.org/weact-stm32f411ceu6-black-pill-high-resolution-pinout-and-specs/)). A 16-channel DMA-driven sweep at even a conservative 1 MSPS effective rate takes `16 / 1,000,000 = 16 µs` — roughly **two orders of magnitude faster** than the Nano path, and crucially, **no external multiplexer is needed** because the chip has enough native ADC-capable pins.

This is the clearest, most quantifiable argument in this whole paper for the MCU decision in Section 10/11.

---

# 9. ENCODER RESEARCH

Encoder feedback converts the control problem from open-loop "PWM duty cycle ≈ speed" (which drifts with battery sag, motor-to-motor mismatch, and floor friction) to closed-loop "measured wheel speed vs. commanded speed," which is what lets a fast robot hold a straight line under acceleration and execute repeatable, symmetric turns.

## 9.1 Encoder technology options

- **Integrated magnetic Hall-effect encoders on the motor** (as used by several Section 6 candidates): a magnetized disc on the motor's rear shaft is read by two Hall sensors 90° apart, giving quadrature output without any extra assembly work. **[VERIFIED]**, e.g., the Adafruit-style N20 magnetic encoder family: **14 counts per revolution of the motor shaft, before the gearbox multiplies it** ([littlebirdelectronics.com.au](https://littlebirdelectronics.com.au/products/n20-dc-motor-with-magnetic-encoder-6v-with-1-50-gear-ratio.json)).
- **Discrete magnetic angle-sensor ICs** such as the **AS5600**: a separate IC + diametric magnet, 12-bit resolution = **4096 counts per revolution**, I2C/PWM/analog output, rated for shaft speeds up to roughly **1000 RPM for full 12-bit output** per one tutorial source, 3.3 V native (needs a level shift or 3.3 V supply on a 5 V Nano) ([tinkered.ai](https://tinkered.ai/components/as5600); [zbotic.in tutorial](https://zbotic.in/as5600-magnetic-encoder-absolute-position-sensing-explained/)). This gives far higher resolution per revolution than an integrated 14-CPR motor encoder, but at extra BOM cost, extra assembly precision (magnet-to-sensor gap must be 0.5–3 mm per the datasheet-derived guidance), and I2C bus bandwidth cost if reading two of them continuously.
- **Optical encoders** (slotted disc + IR photointerrupter): higher resolution than magnetic Hall types at low cost but more sensitive to dust/debris — a real concern on a floor-level competition robot — not used in any of the Section 6 motor candidates that ship with encoders, and not separately researched further here given the magnetic options already cover this design's needs.

## 9.2 PPR/CPR, quadrature, and effective resolution at the wheel

For an integrated 14-CPR (pre-gear) magnetic encoder motor at, say, a 1:30 gear ratio: **effective counts per output-shaft revolution = 14 × 30 = 420 counts/rev**, and in full quadrature decoding (counting both edges of both A and B channels) this becomes **4× that = 1680 quadrature counts/rev [CALC]**, more than enough resolution for velocity estimation at these speeds.

For candidate motor 8 (1:10 ratio, no integrated encoder) paired with a field-added AS5600 mounted on the *motor* shaft (before the gearbox) rather than the wheel: 4096 counts/rev at the motor shaft × 1:10 reduction gives **40,960 effective quadrature-equivalent counts per wheel revolution [CALC]** — far more resolution than needed, but achievable at a real component and integration cost, and note the AS5600's ~1000 RPM speed ceiling for reliable 12-bit output becomes relevant here, since the *motor* shaft (pre-gearbox) at candidate 8 spins right at that same ~1000 RPM no-load figure — **this is a genuine design risk worth flagging**: AS5600 mounted pre-gearbox on candidate-8-class motors may be operating right at its documented speed ceiling, whereas mounting it post-gearbox (on the wheel/output shaft, which spins 10× slower) avoids this risk entirely and is the safer integration point.

## 9.3 Velocity estimation, sampling frequency, interrupt requirements

Standard approach: **[ASSUMPTION, standard practice]** count quadrature edges via hardware pin-change or external interrupts (ATmega328P) or hardware timer encoder-mode inputs (STM32F411, which has **native quadrature encoder timer input on its general-purpose timers**, per the Black Pill spec sheet — **[VERIFIED](https://docs.zephyrproject.org/latest/boards/weact/blackpill_f411ce/doc/index.html)**, "each with up to four IC/OC/PWM or pulse counter and quadrature (incremental) encoder input"). This is a materially better fit than the Nano's software-interrupt-only approach, because hardware quadrature decoding offloads counting entirely from the CPU, freeing cycles for the ADC sweep and PID math — directly relevant to the Section 10 MCU decision.

## 9.4 Recommendation

- If you keep candidate motors with **integrated magnetic encoders** (Section 6, candidates 2–4 class), use their native 14-CPR-pre-gear output directly — no added BOM cost, adequate resolution once multiplied by gear ratio and quadrature decoding.
- If you go with the higher-speed **candidate 8 class motor** (no integrated encoder), add a field AS5600 per motor, **mounted on the output (wheel) shaft, not the motor shaft**, to stay well under its speed ceiling and to measure the quantity you actually care about (wheel rotation) directly rather than inferring it through gearbox backlash.

---

# 10. MCU RESEARCH

| Spec | Arduino Nano (ATmega328P) | STM32F411 "Black Pill" (WeAct) |
|---|---|---|
| CPU | AVR 8-bit, 16 MHz | ARM Cortex-M4 w/ FPU, up to 100 MHz |
| Timers | Several 8/16-bit timers (fixed function) | Up to 11 timers, several with **native quadrature-encoder mode** |
| PWM | 6 PWM-capable pins | Multiple advanced-timer PWM channels, higher resolution |
| ADC | Single 10-bit SAR ADC, 6 (Nano exposes 8) input mux channels, ~15 kSPS practical throughput | 1×12-bit ADC, **2.4 MSPS**, up to 16 channels, DMA-capable |
| Interrupts | External INT0/INT1 + pin-change interrupts on all ports | Extensive NVIC, per-pin external interrupt capability |
| RAM | 2 KB | 128 KB |
| Flash | 32 KB (2 KB used by bootloader) | 512 KB |
| Encoder interface | Software only (pin-change ISR) | Hardware timer encoder mode (offloads CPU) |
| Dev complexity | Very low — Arduino IDE, huge library base | Moderate — STM32 HAL/Arduino-core-for-STM32 both available, smaller (but real) community |
| Latency | Simple/predictable, but ADC+ISR contention is real at high channel counts | Lower per-operation latency, DMA reduces CPU involvement |
| Cost | ~US$/local equiv. low, and it's what you already own | Comparable low cost (~US$3-6 class board), extra line item |

Sources: Nano/ATmega328P — [Arduino official spec](https://www.pishop.us/product/arduino-nano/), [ADC throughput reference](https://ece-classes.usc.edu/ee459/library/documents/ADC.pdf). STM32F411 Black Pill — [Zephyr board doc](https://docs.zephyrproject.org/latest/boards/weact/blackpill_f411ce/doc/index.html), [mischianti.org pinout/spec reference](https://mischianti.org/weact-stm32f411ceu6-black-pill-high-resolution-pinout-and-specs/).

## 10.1 Is the Arduino Nano technically sufficient?

**For a first working PID robot at moderate speed: yes.** For the *specific* combination your schematic already implies — 16-channel analog sensing **and** dual quadrature encoder feedback **and** a fast (200–1000 Hz) control loop, run **simultaneously** — the Nano is measurably tight, not merely "less powerful":

- 16-channel analog reads at ~15 kSPS cost ~1.07 ms of ADC time alone per full sweep (Section 8.2), before any mux-switch settling delay, before PID math, before servicing two motors' worth of quadrature interrupts, which on an 8-bit AVR with software-based edge counting can consume a non-trivial fraction of each loop at high wheel RPM (each encoder edge is an ISR call with fixed overhead in the tens of CPU cycles range even for a tight, hand-optimized ISR — **[ASSUMPTION]**, standard AVR ISR-overhead order of magnitude, not independently benchmarked here).
- This does **not** mean the Nano "can't work" — many real FLF robots run on ATmega328P successfully — but it does mean your **control loop rate is bounded well below what the motors/sensor can physically support**, which is exactly the ceiling your "usable speed, not just top speed" objective (Section 30 of your brief) cares about.

## 10.2 Recommendation

This is a genuine two-viable-architectures case, not a forced upgrade:

- **Keep Nano** if you fix the sensor-interface gap with an external analog mux (cheap, small schematic change) and accept a control loop in the low hundreds of Hz — likely still enough for a solid, competitive-but-not-cutting-edge result, and it preserves nearly all of your existing schematic and firmware investment.
- **Move to STM32F411** if your goal is genuinely "maximum practical speed" as stated in your brief's objective — the ADC throughput and hardware quadrature decoding directly remove the two bottlenecks identified in Sections 8 and 9, at the cost of a firmware rewrite (different HAL/register model, though Arduino-core-for-STM32 significantly softens this) and one more part to source.

Both are documented, defensible choices; Section 25 records this as an open trade-off rather than a forced single answer, per your instructions.

---

# 11. CONTROL SYSTEM

## 11.1 Signal chain

```
16 raw sensor readings
        │
        ▼
Normalize each channel (per-channel min/max from calibration)
        │
        ▼
Weighted position estimate:
   position = Σ(sensor_i_normalized × weight_i) / Σ(sensor_i_normalized)
   weight_i assigned by physical position, e.g. -3500..+3500 across the array
        │
        ▼
error = position - center_setpoint(0)
        │
        ▼
PID:
   P_term = Kp × error
   I_term += Ki × error × dt        (clamped — anti-windup)
   D_term = Kd × (error - error_prev) / dt   (low-pass filtered)
   correction = P_term + I_term + D_term
        │
        ▼
base_speed = speed_schedule(|D_term| or curvature_estimate)   ← Section 12
left_pwm  = clamp(base_speed - correction, 0, MAX)
right_pwm = clamp(base_speed + correction, 0, MAX)
        │
        ▼
Encoder-based closed loop trims left/right individually so both wheels
actually achieve their commanded speed despite motor-to-motor mismatch
and battery sag (cascaded velocity control, inner loop)
        │
        ▼
TB6612FNG PWM + direction pins
```

## 11.2 Key design elements, each justified

- **Weighted-average position estimate** (not simple "which single sensor is darkest") is the standard approach because it gives sub-sensor-pitch resolution — with 8 mm pitch (Section 8), a well-tuned weighted average can resolve line position to a fraction of that 8 mm, materially better than the raw physical spacing — **[OPINION/well-established control technique]**, not something I'm citing to a single source because it's foundational control theory, not a proprietary claim.
- **Derivative filtering** is necessary because raw sensor noise, amplified by the D-term's division by a small `dt`, is a classic source of jittery, oscillating steering at high loop rates — a simple exponential moving average or fixed-window average on the derivative term is standard practice.
- **Anti-windup clamping on the integral term** prevents the classic failure mode where, after a hard line-loss event or sharp turn saturates the output, the accumulated integral term causes a large overshoot once the line is reacquired.
- **Output saturation** (clamping motor PWM to [0, MAX]) is mandatory — without it, a large error can request more correction than the driver/motor can physically deliver, and the *ratio* between the two wheels' commands becomes meaningless once one side is already saturated.
- **Cascaded position (outer) / velocity (inner) control using encoder feedback** is what lets you set a *speed schedule* (Section 12) and trust the robot to actually hit that speed rather than whatever the open-loop PWM-to-speed relationship happens to produce at the current battery voltage — this is the single biggest practical reason encoder feedback matters for a *fast* FLF specifically, more than for a slow one, because speed errors compound faster at higher velocity.

## 11.3 Alternatives considered

- **PD only (no I term):** simpler, avoids integral windup entirely, often sufficient because line-following error rarely has a persistent steady-state bias the way, e.g., a thermostat does — **[OPINION]**, common simplification in FLF firmware, worth trying first before adding I.
- **PID + feed-forward:** adding a feed-forward term proportional to the commanded curvature (from the speed-scheduling logic in Section 12) *before* the reactive PID correction reduces reliance on the (inherently lagging) error signal alone — reduces phase lag, a real benefit at high speed.
- **Cascaded position/velocity control:** as implemented above; recommended.
- **Look-ahead control:** using the shape of the sensor reading across the whole array (not just its weighted centroid) to estimate curvature *ahead* of the robot, and pre-emptively reducing speed before the geometric center reaches the tightest part of a curve — this is the most sophisticated option and the most effective at genuinely high speed, but requires more tuning and more per-loop computation (favors the STM32F411 choice from Section 10).

## 11.4 Pseudocode

```c
// Called at fixed control-loop rate (target 200-1000 Hz depending on MCU choice)
void control_loop() {
    read_sensors(raw[16]);                       // Section 8
    normalize(raw, norm, calib_min, calib_max);   // per-channel calibration
    float position = weighted_position(norm);     // -3500..+3500 style scale
    float error = position - 0.0f;

    float dt = now() - last_time;
    integral = clamp(integral + error * dt, -I_MAX, I_MAX);
    float raw_derivative = (error - last_error) / dt;
    filtered_derivative = alpha * raw_derivative + (1 - alpha) * filtered_derivative;

    float correction = Kp * error + Ki * integral + Kd * filtered_derivative;

    float curvature_estimate = fabs(filtered_derivative);       // Section 12
    float base_speed = speed_schedule(curvature_estimate);

    float left_cmd  = clamp(base_speed - correction, 0, MAX_PWM);
    float right_cmd = clamp(base_speed + correction, 0, MAX_PWM);

    // Inner velocity loop using encoders
    float left_actual_speed  = read_encoder_speed(LEFT);
    float right_actual_speed = read_encoder_speed(RIGHT);
    left_pwm  = velocity_pid(LEFT,  left_cmd,  left_actual_speed);
    right_pwm = velocity_pid(RIGHT, right_cmd, right_actual_speed);

    set_motor(LEFT,  left_pwm);
    set_motor(RIGHT, right_pwm);

    last_error = error;
    last_time = now();
}
```

---

# 12. HIGH-SPEED CONTROL STRATEGY

Increasing PWM alone does not increase *usable* speed once the physical/perceptual limits below are reached — reflecting your brief's own framing:

- **Sensor array width and wheelbase geometry** jointly set the maximum detectable curvature before the line exits the array entirely; a wide sensor relative to a short wheelbase can "see" a sharp turn coming sooner, buying reaction time.
- **Track width vs. line width** determines how much lateral margin exists before line loss.
- **CG height and track width** set the practical cornering speed before tipping/sliding becomes the limiter rather than the motor.
- **Tire grip** (Section 14) is very likely, per the Section 6.3 torque-margin calculation, the actual ceiling on cornering acceleration for this motor class, not motor torque.
- **Derivative-term noise and filtering** set how aggressively you can react to error without inducing oscillation.
- **Braking authority** (achieved via TB6612's short-brake mode, already wired correctly per Section 4.1) sets how late you can commit to a given speed before a curve and still slow down in time.

## 12.1 Practical speed-scheduling logic (structure, not arbitrary numbers)

```
if curvature_estimate < THRESH_STRAIGHT:
    target_speed = V_MAX
elif curvature_estimate < THRESH_MILD:
    target_speed = interpolate(V_MAX, V_MILD, curvature_estimate)
elif curvature_estimate < THRESH_SHARP:
    target_speed = interpolate(V_MILD, V_SHARP, curvature_estimate)
else:
    target_speed = V_SHARP   // floor, not zero — keep enough speed for control authority

if line_lost:
    enter RECOVERY mode: reduce speed to V_SEARCH,
    use last-known error direction to sweep and reacquire,
    time-out to STOP if not reacquired within T_search
```

**THRESH_* and V_* values are explicitly left as tuning parameters, not hardcoded here**, per your instruction not to invent arbitrary numbers without justification — they must be derived empirically during the calibration procedure (Section 21) on your actual chassis/motor/tire combination, because they depend on quantities (real tire grip coefficient, real chassis CG, real derivative-noise floor) that cannot be correctly predicted from datasheets alone.

---

# 13. MECHANICAL DESIGN

## 13.1 Material comparison

| Material | Stiffness (qualitative) | Mass | Vibration damping | Manufacturability | Cost | Verdict for FLF chassis |
|---|---|---|---|---|---|---|
| FR4 (PCB-as-chassis) | High for thin sections | Low-moderate | Low | Excellent — same fab as your electronics PCB, can combine structure + circuitry | Low incremental cost if you're already fabricating a PCB | **Strong candidate** — very common in FLF builds, lets the main PCB *be* the chassis |
| Aluminum sheet/plate | Very high | Moderate-high for thin gauge | Low | Good (needs cutting/drilling tools) | Moderate | Good for a rigid top deck or motor-mount plate, heavier than FR4 for full chassis |
| Carbon fiber sheet | Very high, best stiffness/weight | Low | Low | Harder to machine (special tools, dust hazard) | High | Best performance, but cost/manufacturability overkill for a first competition build — **[OPINION]** |
| Acrylic | Low-moderate, brittle | Low | Low | Very easy (laser cut) | Low | Common for prototypes; risk of cracking on impact |
| 3D printed (PETG/PLA/Nylon) | Low-moderate (PLA is brittle, PETG/Nylon tougher) | Moderate (infill-dependent) | Higher than rigid materials | Excellent for complex geometry (sensor mounts, motor brackets) | Low | **Best for brackets/mounts**, not for the main structural chassis at speed |
| Hybrid (FR4 chassis + 3D printed mounts + aluminum motor bracket) | High where it matters | Optimized | Low | Good | Moderate | **Recommended approach** |

**Recommendation:** Use the **main PCB itself as the structural chassis** (a well-established FLF practice), in FR4, with 3D-printed motor mounts and a 3D-printed or laser-cut front sensor-mast extension. This minimizes part count, minimizes assembly-introduced flex, and is what most fast, light FLF robots actually do — **[OPINION]**, consistent with the general design patterns noted in Section 3.

## 13.2 Layout guidance (approximate, to be refined once motor/wheel/PCB size are locked)

- **Wheelbase:** ~70–90 mm (distance between drive wheels) — wide enough for stability, narrow enough to keep turning radius tight; must be finalized against Section 2's curve-radius target.
- **Track width:** matched to wheelbase within a similar range; wider track improves cornering stability at speed (lower rollover/slide risk) at the cost of turning radius.
- **Sensor overhang:** the sensor array should be mounted ahead of the front axle (or ahead of a front caster/skid) by roughly 20–40 mm, enough to give look-ahead time at target speed without so much overhang that it destabilizes weight distribution or clips the track on sharp turns — this is a genuine tuning variable, not a fixed number, and interacts directly with your control loop's reaction time (Section 12).
- **Motor mounting:** motors mounted low and inboard, symmetric about the robot's centerline, to keep CG low and centered.
- **Battery placement:** as low and as centrally located as possible (heaviest single component after motors) — placing it too far forward or rearward shifts weight onto one axle and changes cornering behavior asymmetrically.
- **PCB placement:** if using the PCB-as-chassis approach, the PCB *is* the layout — component placement doubles as weight distribution, so heavy parts (buck converter, connectors) should be placed with CG in mind, not just trace-routing convenience.
- **OLED/button placement:** top-mounted, rear-facing (away from the sensor/front area), for easy access without interfering with the sensor's optical path or the front overhang.
- **CG location:** as low as physically possible, centered left-right, and longitudinally between the drive axle and slightly toward it (not toward the passive caster/skid) to maximize drive-wheel traction under acceleration.

---

# 14. WHEELS AND TIRES

**No specific wheel/tire product was in scope of your uploaded files**, so this section gives the trade-off framework plus the calculations from Section 6 already performed for 32 mm and 42 mm options.

| Factor | Smaller wheel (e.g., 32 mm) | Larger wheel (e.g., 42 mm) |
|---|---|---|
| Surface speed at given motor RPM | Lower (Section 6.2 calc) | Higher (Section 6.2 calc) |
| Torque at wheel (for given motor torque) | Higher (shorter lever arm) | Lower |
| Rotational inertia | Lower — faster to spin up/down, favors quick direction changes | Higher — more resistant to speed changes, can smooth out minor track imperfections but resists quick corrections |
| Ground clearance / chassis height | Lower chassis possible → lower CG | Slightly higher chassis needed |
| Sensitivity to wheel-diameter matching error (L vs. R) | Same principle both sizes — a mismatch causes a systematic turning bias regardless of absolute size | Same |

Tire material/hardness/grip is a real performance lever (softer rubber compounds grip better but wear faster and add rolling resistance; silicone O-ring "tires" are a common cheap high-grip choice in this robot class — **[OPINION]**, not independently sourced with a datasheet in this pass) — **UNVERIFIED specific tire product recommendation**; this is a good candidate for empirical testing (Section 22) rather than a spec-sheet decision, since grip coefficient on your specific track surface cannot be predicted from a datasheet.

**Recommendation:** Given the Section 6.3 finding that this motor class has generous torque margin (traction-limited, not torque-limited), and given that the requirement is *speed*, lean toward the **larger wheel (≈40–42 mm)** for higher top speed per motor RPM, accepting the torque-margin trade because Section 6.3's calculation shows there's room to give up some torque.

---

# 15. POWER SYSTEM

## 15.1 Topology (already largely correct in your schematic; see Section 26 for the changes needed)

```
2S LiPo → [NEW: fuse] → [NEW: reverse-polarity protection] → VBAT rail
                                                                  │
                                              ┌───────────────────┼──────────────────┐
                                              ▼                                       ▼
                                        TB6612FNG (VM)                        Mini-360 → 5V rail
                                                                                   │
                                                              ┌────────────────────┼──────────────┐
                                                              ▼                    ▼              ▼
                                                            MCU                  OLED          Sensor array
```

## 15.2 Battery — [VERIFIED, real product class]

A representative real part: **2S 7.4 V 500 mAh, 35C** LiPo, JST/XT-class connector, ~25 g — this exact spec is commercially available from multiple vendors, e.g., a Gens Ace–class 400 mAh 35C pack at 17 g / 46×19×11.3 mm ([ozrc.com.au](https://ozrc.com.au/products/gens-ace-2s-bashing-400mah-7-4v-35c-soft-case-lipo-battery-jst-ohr-2p-gea4002s35js)), or a 500 mAh 30C pack at 25 g / 42×14×10 mm ([lindinger.at](https://lindinger.at/en/RC-Electronics/Drive-Set/Batteries/PICHLER-LiPo-battery-FliteZone-500-7-4V-e.g.120X/9795182)). At 35C continuous rating, a 400 mAh pack can source **14 A continuous** (0.4 Ah × 35 = 14 A) — vastly more than this robot will ever draw, so **battery C-rating is not a constraint** for any motor candidate in Section 6; pack *selection* should instead be driven by mass and physical size.

## 15.3 Current budget [CALC]

Using candidate motor 8 (stall 0.35 A) and worst-case both motors simultaneously stalled (e.g., a hard collision or jam):

```
Peak current (both motors stalled) = 2 × 0.35 A = 0.70 A   [CALC]
Add MCU + sensor array + OLED + buzzer overhead:
  - Nano: ~19 mA typical draw (pishop.us spec)
  - QTR-MD-16A sensor array: up to 250 mA (all LEDs on)   [VERIFIED]
  - OLED (typical 0.96" SSD1306-class): ~20-30 mA  [ASSUMPTION — no specific module locked yet]
  - Buzzer (active, briefly): tens of mA when active
Total peak, worst case: ≈ 0.70 + 0.25 + 0.03 + 0.02 ≈ 1.0 A   [CALC]
```

This is comfortably within both the TB6612FNG's continuous rating *per channel* (motors are on separate channels, so each channel only ever sees one motor's current, not the sum) and the Mini-360's 1.8 A continuous rating for the logic-side load.

## 15.4 Runtime estimate [CALC]

Using a 400 mAh pack and an **average** (not peak) current draw — realistically dominated by the motors running at partial duty, not stall — estimated at roughly 300–500 mA average for a robot of this class actively running **[ASSUMPTION, order-of-magnitude, not independently measured]**:

```
Runtime ≈ capacity / average_current = 0.4 Ah / 0.4 A ≈ 1 hour of continuous running
```

This comfortably covers the ≥15-minute PRD target (Section 2) with wide margin, confirming a smaller/lighter pack (e.g., 300 mAh) could also be viable if mass reduction becomes a priority — a genuine design trade-off, not a fixed answer.

## 15.5 Voltage drop / regulator dissipation [CALC]

Mini-360's datasheet-quoted efficiency is **up to 95–96%** at its test point (5 Vin→3.3 Vout ~200 mA, [components101.com](https://components101.com/node/2228)); at your actual operating point (7.4 V → 5 V, higher current for sensor array + MCU + OLED), efficiency will be somewhat lower than the datasheet's best-case figure — **[ASSUMPTION]**, typical buck converters lose several efficiency points away from their optimal test point — but even at a conservative 85% efficiency assumption, dissipated power at ~300 mA logic-side draw is:

```
P_out = 5V × 0.3A = 1.5 W
P_in = P_out / 0.85 ≈ 1.76 W
P_dissipated ≈ 0.26 W
```

This is a small, manageable amount of heat for a module this size, not a thermal design concern at this current level.

---

# 16. PCB DESIGN

| Parameter | Recommendation | Justification |
|---|---|---|
| Layer count | 2-layer | Sufficient for this part count/complexity; 4-layer only justified if EMI issues appear empirically |
| Board thickness | 1.6 mm standard, or 1.0–1.2 mm if using the PCB-as-chassis approach and weight matters more than rigidity — trade-off, test both | Thinner boards flex more; if PCB is your chassis, don't go thinner than needed to keep sensor height stable (Section 4.4) |
| Copper thickness | 1 oz (35 µm) standard; consider 2 oz on the motor-current layer/traces if routing very short, wide traces is difficult | Reduces resistive loss/heating on motor traces |
| Ground plane | Solid ground pour on the non-component layer, with a **deliberate split or careful routing keeping motor-return current paths away from the ADC reference/sensor ground path**, joined at a single star point near the battery input | Directly addresses the noise risk flagged in Section 4.6 |
| Motor current routing | Wide, short, direct traces from TB6612FNG outputs to motor pads; **[CALC]** trace width below | Motor traces carry the highest current on the board |
| Logic routing | Thin traces acceptable, keep away from motor traces/switching nodes where possible | Reduces coupled switching noise into ADC lines |
| Decoupling placement | 100 nF ceramics directly adjacent to each IC's power pin (already conceptually correct in your schematic — enforce physical proximity in layout); add the 4.3-flagged additional bulk cap near the 5V rail at the MCU | |
| Connector placement | XT60/battery input at board edge, fuse and protection components immediately adjacent to it (protect as early as possible in the current path) | |
| EMI/noise | Keep high-dI/dt motor driver switching nodes away from sensor analog traces; consider a small series ferrite bead on the 5V rail feeding the sensor array if noise is observed empirically | |
| Encoder routing | Short traces from motor/encoder connector to MCU, twisted-pair-style adjacent routing (A/B lines side-by-side) recommended if using ribbon/flex cable to the motor | |
| Sensor interface | If Nano: route through external mux, keep mux control lines and analog lines separated; if STM32F411: direct ADC pin routing, shorter overall | |
| Mounting holes | 4× corner holes sized for M2/M2.5 standoffs (final size depends on chassis fastener choice) | |
| Test points | Expose VBAT, 5V, GND, and each motor output as labeled test pads | Speeds up bring-up debugging |
| Programming/debug connector | Nano: onboard USB, no extra connector needed. STM32F411 Black Pill: onboard USB + exposed SWD pads already present on the module | |

## 16.1 Trace width from calculated current, not arbitrary numbers

Using IPC-2221 standard external-layer trace width guidance (rule-of-thumb form, **[CALC]** applied to your actual currents, not invented): for 1 oz copper, external layer, a 10°C rise:

- At ~0.5 A per motor channel (typical running current, well under the 0.35–0.43 A stall figures used as worst case) a **0.3–0.4 mm (12–16 mil) trace is generally adequate**; given the short trace lengths involved on a small robot PCB, err generous and use **0.5 mm (20 mil)** minimum for all motor-current traces for margin and easier hand assembly/rework.
- Logic/signal traces: standard 0.25 mm (10 mil) is fine.
- VBAT main trunk (carries combined current for both motor channels, up to the ~0.7–1.0 A peak calculated in Section 15.3): use **0.75–1.0 mm (30–40 mil)** for this specific trace/pour.

These are conservative, margin-including recommendations appropriate for a hobbyist/small-shop fabrication process, not a precision high-current power design — if you move to a higher-torque/higher-current motor candidate later, redo this calculation with that motor's actual stall current.

---

# 17. FIRMWARE ARCHITECTURE

```
main()
 ├── init()
 │     ├── init_gpio()
 │     ├── init_adc() / init_mux()          // Section 8
 │     ├── init_timers_pwm()
 │     ├── init_encoder_interrupts()        // Section 9
 │     ├── init_i2c_oled()
 │     ├── init_buttons()
 │     └── load_calibration_from_eeprom()   // if previously calibrated
 │
 ├── menu_state_machine()                    // UP/DOWN/SELECT/START-STOP, drives OLED
 │     ├── STATE_MENU
 │     ├── STATE_CALIBRATE       → calibration_routine()   // Section 21
 │     ├── STATE_READY
 │     ├── STATE_RUNNING         → control_loop() at fixed rate, see Section 11
 │     └── STATE_FAULT           → fault_handler()
 │
 └── ISR: encoder_A_left, encoder_A_right (or hardware timer on STM32)
 └── ISR/Timer: fixed-rate control-loop trigger
```

**Recommended loop frequencies:**

- Sensor read + PID + motor update: target **200 Hz minimum on Nano (mux path), up to 1 kHz achievable on STM32F411** (Section 8.2/10.1 basis).
- Encoder sampling: continuous via interrupt (event-driven), velocity computed over a fixed window (e.g., every control-loop tick) rather than per-edge, to smooth quantization noise at low speed while still being fast enough to track speed changes at high speed.
- OLED/button UI: low priority, updated at ~10–20 Hz, must never block or delay the control loop — implement as non-blocking state updates only.
- Buzzer: event-driven (start/stop tones, fault beeps), non-blocking.

**Timing requirement:** the control loop must be the highest-priority, most time-deterministic task in the system — UI and buzzer code must be structured so they cannot introduce jitter into the control loop's period, since D-term quality (Section 11.2) depends directly on consistent `dt`.

---

# 18. BOM

Full production BOM — **columns exactly as specified**. Status uses LOCKED / OPEN / OPTIONAL / BACKUP as instructed. No invented manufacturer part numbers are used; where a part is not yet chosen, the "Part number" field says "TBD — see report section."

| Item | Reference | Component | Exact specification | Manufacturer | Part number | Package | Qty | Recommended supplier | Approx. price | Source | Reason for selection | Backup part | Status |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | U1 | MCU | ATmega328P, 5V Nano module (keep) OR STM32F411CEU6 Black Pill (upgrade) | Generic Nano-compatible / WeAct | Nano: A000005-class; Black Pill: STM32F411CEU6 | Nano module / QFN48 dev board | 1 | Local distributor / AliExpress-class | US$3–7 | Sec. 10 | ADC/timer throughput analysis | The other option in this row | **OPEN** (architecture decision) |
| 2 | U2 | Motor driver | TB6612FNG dual H-bridge | Toshiba | TB6612FNG (carrier: Pololu #713) | SSOP24 IC or carrier board | 1 | Pololu / local distributor | US$4.95 (carrier) | Sec. 7 | Keep — current margin adequate, already correctly wired | DRV8833 (Pololu #2130) | **LOCKED** (part), OPEN (module vs bare IC form factor) |
| 3 | U3 | Display | 0.96" SSD1306-class I2C OLED | TBD — verify exact module | TBD | 4-pin, pin order TBD | 1 | Any confirmed-pinout local supplier | ~US$2–4 | Sec. 4.2 | Must verify pin order before layout | — | **OPEN** |
| 4 | U4 | Power connector | XT60 (keep) or downsize to XT30 for lower mass | Amass-compatible generic | XT60 / XT30 | THT solder-cup or PCB right-angle | 1 | Local hobby distributor | US$0.50–1 | Sec. 4.11, 15 | 60A rating is oversized for this current draw; XT30 (30A/60A burst) saves mass | XT30 | **OPTIONAL downgrade** |
| 5 | U5 | Buck converter | Mini-360 (MP2307-based) | MPS (IC) / generic module assembler | MP2307DN-based Mini-360 module | SMD module, 17×11×3.8mm | 1 | Any generic electronics supplier | US$1–4 | Sec. 15.5 | Confirmed adequate for logic-side load | Any 3.3–5V buck rated ≥1A | **LOCKED** |
| 6 | U6/U7 | Motor | Candidate 8 class: 6V, 1:10, ~1000 RPM no-load N30-family micro gearmotor | Generic (thingbits-sourced reference) | 12SG-N30VA-10 (reference part) | N30 body, D-shaft 3mm | 2 | thingbits.in or equivalent local supplier | Price not confirmed in this pass — UNVERIFIED | Sec. 6 | Best-documented speed-class candidate with real datasheet numbers | Candidate 1 (BJ-N20-12-1000) if 12V-rated behavior preferred and re-derated for 7.4V | **OPEN** — final selection pending your confirmation |
| 7 | ENC-R/ENC-L | Encoder | AS5600 magnetic encoder breakout, mounted on wheel/output shaft | ams OSRAM | AS5600 | SOIC8 or breakout module | 2 | Generic breakout supplier | US$1–3 each | Sec. 9 | Needed because candidate-8 motor has no integrated encoder | Integrated-encoder N20 (candidates 2-4) if you switch motor choice | **OPEN**, dependent on Item 6 decision |
| 8 | — | Line sensor array | 16-channel IR reflectance array, analog output, ~7-8mm pitch | Pololu-equivalent (QTR-MD-16A as reference spec) | QTR-MD-16A or verified equivalent (e.g., RoboJunkies/Robokits 16ch board — confirm exact model) | 125×16.5mm-class board | 1 | Pololu direct, or verified local equivalent | ~US$29–32 (Pololu reference price) | Sec. 8 | Directly documented, real spec sheet | JSumo XLINE 16 | **OPEN** — must add to your BOM, currently missing entirely |
| 9 | — | Analog multiplexer (Nano path only) | 74HC4051-class 8:1 mux ×2, or 16:1 equivalent | TI/ON Semi/generic | 74HC4051 | SOIC16 | 2 (if Nano path chosen) | Any generic distributor | <US$1 each | Sec. 4.7 | Required to physically read 16 analog channels on a Nano | N/A — not needed if STM32F411 path chosen | **OPEN**, conditional on Item 1 |
| Q1 | Q1 | Transistor | 2N2222 NPN | Generic (verify exact fab pinout) | 2N2222 | TO-92 | 1 | Local distributor | <US$0.10 | Sec. 4.10 | Keep, verify pinout on purchased part | BC547 | **LOCKED** (part), OPEN (pinout confirmation) |
| BZ1 | BZ1 | Buzzer | 5V active buzzer | Generic | TBD | THT 2-pin | 1 | Local distributor | <US$0.50 | — | Keep | Passive buzzer + PWM tone (firmware change) | **LOCKED** |
| R1 | R1 | Resistor | 1 kΩ | Generic | — | 0805 | 1 | Any | <US$0.01 | — | Keep | — | **LOCKED** |
| R2 | R2 | Resistor | 10 kΩ | Generic | — | 0805 | 1 | Any | <US$0.01 | — | Keep | — | **LOCKED** |
| R3 | R3 | Resistor | 4.9 kΩ (verify — non-standard value, likely meant to be 4.7 kΩ E12 value) | Generic | — | 0805 | 1 | Any | <US$0.01 | Sec. 26 | Flag possible typo vs. standard E-series | 4.7 kΩ | **OPEN — verify intended value** |
| R4 | R4 | Resistor | 10 kΩ | Generic | — | 0805 | 1 | Any | <US$0.01 | — | Keep | — | **LOCKED** |
| C1 | C1 | Capacitor | 100 nF, X7R | Generic | — | 0805 | 1 | Any | <US$0.01 | — | Keep | — | **LOCKED** |
| C2 | C2 | Capacitor | 100 nF, X7R | Generic | — | 0805 | 1 | Any | <US$0.01 | — | Keep | — | **LOCKED** |
| C3 | C3 | Capacitor | 470 µF, ≥16V rated (was unspecified) | Generic electrolytic | — | Radial/SMD-can | 1 | Any | <US$0.20 | Sec. 4.3, 26 | Voltage margin correction | 2× 220µF in parallel if physical size constrained | **OPEN — voltage rating must be added** |
| — | F1 (new) | Fuse | Resettable PTC fuse, ~2A hold current | Generic (e.g., Bel Fuse / Littelfuse class) | TBD | SMD 1206/1210 or THT | 1 | Any | <US$0.50 | Sec. 4.8, 26 | Missing safety component | Blade/glass fuse + holder | **OPEN — new part, add to BOM** |
| — | Q2 (new) | Reverse-polarity protection MOSFET | P-channel MOSFET, low RDS(on), rated ≥ 2× peak current | Generic (e.g., AO3401-class) | TBD | SOT-23 | 1 | Any | <US$0.30 | Sec. 4.8, 26 | Missing safety component | Schottky diode (simpler, lossier) | **OPEN — new part, add to BOM** |
| KEY1-4 | KEY1-4 | Switch | Momentary tactile, 4-pin | Generic | — | 4-pin tactile, footprint TBD | 4 | Any | <US$0.10 each | — | Keep, verify footprint/height | — | **LOCKED (part), OPEN (footprint)** |
| H1 | H1 | Header | 1×8, pitch TBD | Generic | — | 1×8, TBD pitch | 1 | Any | <US$0.10 | Sec. 4.2 | Must confirm pitch, or replace with mux-based sensor interface entirely | — | **OPEN — likely obsolete if mux added (Item 9)** |
| — | — | Chassis | FR4 PCB-as-chassis (see Sec 13) or 3D-printed alternative | — | — | Custom | 1 | Your PCB fab | Included in PCB fab cost | Sec. 13 | Minimizes part count | Laser-cut acrylic | **OPEN — new, needs CAD** |
| — | — | Wheels/tires | 40-42mm class wheel + high-grip tire | TBD | TBD | — | 2 | TBD | Est. US$2-5/pair | Sec. 14 | Matches speed target from Sec 6 calc | 32mm wheel | **OPEN — new, needs sourcing** |

---

# 19. COST ANALYSIS

**[CALC/ASSUMPTION combined — approximate, USD, single-unit hobbyist pricing, not bulk/production pricing]**

| Category | Prototype cost (1 unit) | Notes |
|---|---:|---|
| Motors (×2) | $6–15 (est., price UNVERIFIED for exact candidate-8-class part in this pass) | |
| Encoders (×2, AS5600) | $2–6 | Verified price class |
| Motor driver | $5 (Pololu carrier) or <$2 (bare IC + hand assembly) | Verified |
| MCU | $3–7 | Verified price class both options |
| Line sensor array | $29–32 (Pololu reference) or less for a local-market clone | Verified for Pololu part |
| PCB fabrication (chassis + main board, small qty) | $10–30 | [ASSUMPTION] typical small-batch hobbyist PCB fab pricing, not independently sourced this pass |
| Chassis extras (3D printed mounts) | $2–5 | [ASSUMPTION] |
| Wheels/tires | $2–5 | [ASSUMPTION] |
| Battery | $10–20 | Verified price class |
| Buck converter | $1–4 | Verified |
| Connectors (XT60/XT30, headers) | $2–5 | Verified price class |
| Switches/buttons | $1–2 | |
| Display (OLED) | $2–4 | |
| Passives (R/C) | <$2 | |
| Wiring | $2–3 | |
| Fasteners | $1–2 | |
| Mux ICs (if Nano path) | <$2 | |
| Fuse + reverse-protection FET | <$2 | |
| **Prototype total (approx.)** | **≈ $80–150** | Sum of above, order-of-magnitude, several line items UNVERIFIED for exact final price |

**Production/rebuild cost** (assuming you already own tools, are reordering known-good parts, and are building a second unit): **≈ $60–110**, mainly saving on one-time costs (initial PCB fab setup, initial mistakes/scrap, tooling).

I am **not** separating imported vs. locally-available parts because your location/sourcing context was not established in this conversation — this would need your actual supplier list (e.g., if you're sourcing primarily from Indian distributors like Robokits/Probots/Zbotic, most of the parts referenced above do have local-market equivalents visible in the sources already cited, e.g., probots.co.in, zbotic.in).

---

# 20. MANUFACTURING PLAN

**PCB:** schematic correction per Section 26 → PCB layout following Section 16 guidance → DRC → gerber export → fabrication (any standard 2-layer FR4 house) → assembly (hand-solder for a board this size and part count is entirely reasonable; reflow only worth it if producing many units) → visual + continuity inspection before first power-up.

**Mechanical:** finalize CAD once motor/wheel/sensor are locked (Section 6/8/14) → if PCB-as-chassis, mechanical design is largely embedded in the PCB outline itself → 3D print motor mounts and sensor mast → drill/ream any mounting holes not covered by PCB fab → dry-fit motor alignment and wheel alignment before final assembly (misaligned wheels are a common, easily-overlooked source of a systematic turning bias).

**Assembly order (recommended):** power system first (battery, fuse, protection, buck converter) → bring up and verify 5V rail with no MCU/sensor load → add MCU, verify basic firmware (blink/serial) → add motor driver + motors, verify open-loop PWM control → add encoders, verify counting direction and rate → add sensor array, verify calibration readings → add OLED/buttons/buzzer last (lowest risk, easiest to debug in isolation) → full-system calibration (Section 21) → test plan (Section 22).

---

# 21. CALIBRATION PROCEDURE

1. **Sensor calibration:** with the robot powered and sensors active, manually sweep the sensor array across the actual track's line and background several times while firmware records per-channel min/max readings; store these min/max values (EEPROM or runtime) and use them to normalize live readings to a consistent 0–1 (or 0–1000) scale regardless of absolute ambient light level.
2. **Motor direction calibration:** command each motor forward individually at low PWM and visually confirm correct rotation direction; correct in firmware (swap logical forward/reverse mapping) rather than by re-wiring, to keep the PCB/schematic mapping consistent.
3. **Motor mismatch calibration:** with the robot on a stand (wheels off the ground) or on a straight measured track, command both motors to the same PWM and compare encoder-measured speeds; record a per-motor scaling correction factor so equal *commanded* speed produces equal *actual* speed — this directly feeds the inner velocity loop from Section 11.
4. **Encoder direction/sign calibration:** confirm that positive encoder count direction matches the intended "forward" direction for each wheel; a swapped sign here causes the velocity control loop to fight itself.
5. **Center position calibration:** with the robot centered on the line, confirm the weighted position calculation reads (approximately) zero; adjust sensor weighting/offset if there's a persistent non-zero bias, which usually indicates a physical mounting asymmetry rather than a software bug.
6. **PID systematic tuning procedure** (not "increase Kp until it works"):
   - Start with Ki = 0, Kd = 0. Increase Kp from zero until the robot follows a straight line with a *small*, consistent oscillation at low speed — this identifies the ultimate gain region.
   - Reduce Kp to roughly 50–60% of that oscillation-onset value as a stable starting point (a standard Ziegler-Nichols-adjacent heuristic — **[OPINION]**, widely used starting heuristic, not a guarantee of optimal tuning for this specific nonlinear system).
   - Introduce Kd, increasing gradually while watching for high-frequency jitter (a sign of insufficient derivative filtering, Section 11.2, before it's a sign of Kd being wrong) versus improved damping of the oscillation from the Kp-only stage.
   - Only add Ki if a persistent steady-state offset is observed (e.g., the robot consistently runs slightly to one side on a straight section) — many FLF robots never need a nonzero Ki, consistent with the PD-only alternative discussed in Section 11.3.
   - Re-tune at each new speed-schedule tier (Section 12), because the effective loop gain (how much position error a given PWM correction produces) changes with speed — a gain set tuned at creep speed will typically be too aggressive at top speed and vice versa; this is why the speed-scheduling structure exists in the first place.

---

# 22. TESTING AND VALIDATION

| Test | Objective | Setup | Measurement | Pass criterion | Failure condition | What to change |
|---|---|---|---|---|---|---|
| Motor test | Confirm both motors spin correctly in both directions | Robot on stand, wheels free | Visual + tachometer/encoder reading | Both motors reach expected no-load RPM within ~10% of each other | One motor stalls, doesn't spin, or spins >15% slower | Check wiring, check driver channel, check for mechanical binding |
| Current test | Confirm current draw matches Section 15 budget | Bench PSU or multimeter in-line with battery | Peak and average current under load | Within ~20% of Section 15 calculated values | Significantly higher current | Check for mechanical binding, driver fault, shorted winding |
| Battery test | Confirm pack delivers rated voltage under load | Full charge, run robot, monitor voltage | Voltage sag under load | Sag within expected 0.3-0.6V range (Sec 6.2 assumption) | Excessive sag (>1V) or voltage collapse | Battery may be degraded/wrong C-rating |
| Encoder test | Confirm correct counts/rev and direction | Manually rotate wheel exactly 1 turn | Compare counted pulses to Section 9 calculated expected value | Within a few counts of calculated value, correct sign | Wrong count or wrong sign | Check gear ratio assumption, check quadrature decode logic |
| Sensor test | Confirm all 16 channels read correctly and distinctly | Sweep sensor over line/background | Per-channel min/max spread | All channels show clear contrast between line and background | Dead/stuck channel, or channels reading identically (mux fault) | Check mux wiring/addressing, check individual sensor solder joints |
| Straight-line test | Confirm basic PID stability | Long straight track section, low-moderate speed | Lateral deviation from line centerline | Stays within line width margin throughout | Oscillates or drifts off | Retune Kp/Kd per Sec 21 |
| Low-speed PID test | Establish baseline tuning | Full track at low, safe speed | Completes track without losing line | Completes at least 3 consecutive clean runs | Loses line, oscillates | Retune |
| Medium-speed test | Validate speed scheduling begins working | Full track at Section 12 "mild" tier | Completes with acceptable margin | Consistent completion | Line loss at specific track features (log which ones) | Adjust speed-schedule thresholds for that curvature |
| High-speed test | Push toward Section 2 target speed | Full track at max commanded speed | Completes, lap time | Completes without line loss | Line loss, tip-over, or wheel slip observed | Reduce max speed tier, or address grip/CG per Sec 13/14 |
| Cornering test | Validate cornering behavior specifically | Isolated sharp-curve track section | Repeated pass success rate | ≥90% success over repeated attempts | Frequent overshoot/line-loss on curve | Retune curve-tier speed/gains, check sensor look-ahead geometry |
| Line-loss test | Validate recovery behavior | Deliberately induce line loss (obstacle, gap) | Time to reacquire, or correct stop | Reacquires within defined search window, or stops safely | Runs away / doesn't stop | Fix recovery state machine timeout logic |
| Repeated-run test | Validate repeatability, not just one good run | 10+ consecutive full-track runs | Success rate, lap-time variance | Low variance, high success rate | High variance or degrading performance over runs | Investigate thermal drift, battery sag over session, mechanical loosening |
| Thermal test | Confirm driver/motor don't overheat during extended running | Touch-test / thermal camera if available after several consecutive runs | Component temperature | Driver and motor stay well below datasheet max operating temp (TB6612FNG: 85°C ambient rated) | Excessive heat, thermal shutdown triggers | Add heatsinking, reduce continuous current, add rest periods |
| Battery voltage variation test | Confirm behavior across the charge cycle, not just at full charge | Run tests at full charge and near low-voltage cutoff | Speed/behavior consistency | Encoder-based velocity loop (Sec 11) compensates for sag automatically | Noticeable speed/behavior change as battery discharges | Confirms value of closed-loop velocity control; if still present, check velocity loop gains |

---

# 23. FAILURE MODE ANALYSIS (FMEA)

| Failure | Cause | Effect | Severity | Likelihood | Detection | Mitigation |
|---|---|---|---|---|---|---|
| Motor stall | Mechanical jam, obstacle, excessive load | Overcurrent, possible driver/motor heating | Medium | Medium | Current spike, encoder speed = 0 while PWM > 0 | Firmware stall-detect (compare commanded vs. encoder-measured speed) → cut PWM |
| Motor overheating | Sustained high current (e.g., repeated stalls, undersized motor for load) | Reduced lifespan, possible winding damage | Medium | Low (given Sec 6.3 torque margin) | Manual thermal check post-run | Keep within calculated current budget (Sec 15.3), avoid prolonged stall |
| Driver overheating | Sustained near-max current, poor PCB thermal design | Thermal shutdown, temporary loss of motor control | Medium | Low (large current margin per Sec 7) | TB6612FNG's own thermal shutdown is self-protecting | Ensure PCB copper pour under driver per Toshiba datasheet guidance for thermal relief |
| Encoder signal loss | Wiring fault, connector failure, magnet/sensor misalignment (if AS5600) | Velocity loop loses feedback, falls back to open-loop behavior | High (directly affects high-speed control quality) | Medium (connector-based wiring is a common failure point) | Firmware: detect zero encoder activity while PWM commanded nonzero | Fall back to open-loop PID with degraded-mode warning (buzzer/OLED) rather than uncontrolled behavior |
| Sensor saturation | Excess ambient light or reflective track surface | Loss of contrast between line/background | Medium | Medium (venue-dependent) | Calibration routine reveals compressed min/max range | Recalibrate at venue under actual lighting; consider shrouding sensor from ambient light |
| Sensor noise | Electrical noise coupling from motor switching (Sec 4.6, 16) | Jittery position estimate, D-term amplified noise | Medium | Medium if grounding not carefully laid out | Visible oscillation in logged sensor data at rest | PCB ground-plane/routing fix (Sec 16), derivative filtering (Sec 11.2) |
| Line loss | Gap in track, sharp turn exceeding sensor's detection range, sensor overhang mis-tuned | Robot leaves track | High | Medium at high speed | Firmware detects all-channels-low condition | Recovery state machine (Sec 12), reduce speed near known-difficult features |
| Battery sag | High instantaneous current draw, aging battery | Reduced motor speed, possible MCU brownout if regulator margin insufficient | Medium | Low-medium | Monitor VBAT via ADC divider | Encoder-based velocity loop compensates for motor-side sag; add low-voltage cutoff/warning for MCU-side risk |
| Loose connector | Vibration during fast operation, repeated connect/disconnect cycles | Intermittent power or signal loss, sudden robot stop or erratic behavior | High (safety + reliability) | Medium | Visual/tug inspection before runs | Use locking connectors where possible, strain-relief wiring, secure connector housings to chassis |
| PCB failure | Trace fracture from flex (esp. if PCB-as-chassis), solder joint fatigue from vibration | Partial or total loss of function | High | Low-medium, depends on chassis stiffness | Continuity testing, visual inspection | Adequate board thickness (Sec 16), avoid mounting stress concentrators near thin traces |
| Software lockup | Firmware bug, blocking code in UI path stalling control loop (Sec 17) | Robot freezes or runs away uncontrolled at last commanded PWM | High | Low if firmware follows Sec 17 non-blocking guidance | Watchdog timer reset | Enable MCU watchdog timer; keep control loop ISR-driven and UI strictly non-blocking |
| Wheel slip | Excess speed into a corner beyond tire grip limit | Line loss, uncontrolled slide | Medium-High | Medium at high speed (Sec 6.3 shows traction, not torque, is the likely limiter) | Encoder speed vs. expected ground speed mismatch (advanced) | Speed-schedule tuning (Sec 12), tire selection (Sec 14) |
| Chassis flex | Under-stiff chassis material/thickness, poor mounting | Sensor height changes under acceleration/braking, degrading sensor readings precisely when they matter most | Medium | Low-medium depending on material choice (Sec 13) | Compare sensor calibration readings static vs. under load/vibration | Increase board thickness or add structural ribs/gussets |
| EMI interference | Motor driver switching noise coupling into I2C (OLED) or ADC lines | Garbled OLED display, jittery sensor readings | Low-medium | Medium if grounding not carefully laid out | Visual OLED glitches, sensor data jitter | PCB layout separation (Sec 16), possible series ferrite on affected rail |

---

# 24. DESIGN ITERATIONS / DEVELOPMENT ROADMAP

**VERSION 0 — Bench electronics**
*Changes:* Assemble power system (battery, fuse, reverse-protect, buck converter), MCU, and motor driver on a breadboard or minimal bring-up PCB. No chassis yet.
*Objective:* Confirm every electrical subsystem works in isolation.
*Tests:* Power-on test, motor test, current test (Section 22).
*Success criteria:* Correct voltages measured at every rail; motors spin correctly in both directions under manual PWM commands.

**VERSION 1 — Slow functional robot**
*Changes:* Add chassis (even a rough/temporary one), wheels, sensor array (open-loop, uncalibrated), basic firmware that reads sensors and drives motors with simple bang-bang or proportional-only steering.
*Objective:* Prove the robot can physically follow a line at low speed.
*Tests:* Straight-line test, low-speed completion.
*Success criteria:* Completes a simple track at low speed, however roughly.

**VERSION 2 — PID-controlled robot**
*Changes:* Full PID implementation (Section 11), sensor calibration routine (Section 21).
*Objective:* Smooth, stable line-following at low-moderate speed.
*Tests:* Low-speed PID test, straight-line test, cornering test at low speed.
*Success criteria:* Multiple consecutive clean runs (repeated-run test) at a fixed moderate speed.

**VERSION 3 — Encoder-assisted robot**
*Changes:* Add encoders (integrated or field AS5600 per Section 9 decision), implement cascaded velocity control (Section 11).
*Objective:* Robot maintains commanded speed despite battery sag/motor mismatch.
*Tests:* Motor mismatch calibration validated, battery voltage variation test.
*Success criteria:* Consistent lap times across a full battery discharge cycle.

**VERSION 4 — High-speed optimized robot**
*Changes:* Implement full speed-scheduling (Section 12), tune per-tier PID gains, finalize mechanical rigidity/CG per Section 13, finalize wheel/tire choice per Section 14.
*Objective:* Reach toward the Section 2 top-speed target while maintaining reliability.
*Tests:* High-speed test, cornering test at speed, thermal test.
*Success criteria:* Completes full track at target speed with ≥90% success rate across repeated attempts.

**VERSION 5 — Competition-ready robot**
*Changes:* Final PCB revision incorporating all Section 26 schematic changes, final enclosure/weight optimization, robustness hardening (connector strain relief, fastener thread-lock, spare-parts kit).
*Objective:* Reliable, repeatable, competition-day-ready hardware.
*Tests:* Full repeated-run test suite, line-loss recovery test under adversarial conditions.
*Success criteria:* Passes every test in Section 22 with defined pass criteria, across multiple practice sessions on the actual competition surface if accessible beforehand.

---

# 25. FINAL ENGINEERING CONFIGURATION

| Requirement | Chosen specification | Reason | Evidence | Trade-off | Confidence |
|---|---|---|---|---|---|
| MCU | **Open decision between Nano (keep) and STM32F411 (upgrade)** | Both are viable; STM32F411 removes two quantified bottlenecks (Sec 8.2, 10.1) | Datasheet-sourced throughput/timer specs | Nano = less redesign effort; STM32F411 = higher ceiling on control-loop rate | Medium — depends on your appetite for firmware rework |
| Motor driver | TB6612FNG (keep current choice) | Current margin adequate for all motor candidates evaluated | Toshiba datasheet + candidate motor stall currents | None significant found | High |
| Motor | Candidate 8 class (N30, 1:10, 6V, ~1000 RPM no-load) as primary recommendation | Best real-world speed-class specs found with hard numbers | thingbits.in datasheet-level listing | No integrated encoder — requires field AS5600 (added cost/assembly) | Medium — exact part availability/price UNVERIFIED, and exact RPM-per-ratio table for the magnetic-encoder N20 family (candidates 2-4) is UNVERIFIED and could beat it if numbers turn out favorable |
| Encoder | AS5600 field-mounted on output shaft, OR integrated N20 magnetic encoder if motor choice changes | Section 9 resolution/speed-ceiling analysis | ams/vendor datasheets | Added BOM/assembly complexity for AS5600 path | Medium |
| Line sensor | 16-channel analog IR array, QTR-MD-16A-equivalent spec | Directly documented, standard for this robot class | Pololu spec sheet | Requires mux (Nano path) — this is the #1 schematic fix needed regardless of MCU choice | High |
| Wheel diameter | ~40-42 mm | Favors top speed given generous torque margin (Sec 6.3) | Calculation from motor candidate + Section 14 trade table | Slightly less torque margin than 32 mm — still ample per calc | Medium — depends on final motor choice |
| Battery | 2S 7.4V, 300-500 mAh, ≥25C | Meets current and runtime budget with large margin | Section 15 calculations | Larger packs = more mass for no real benefit at this current draw | High |
| Chassis | FR4 PCB-as-chassis + 3D printed mounts | Minimizes part count/assembly flex, common practice | Section 13 reasoning | Less design freedom than pure 3D-printed chassis | Medium — [OPINION]-supported, not a single verified source |
| Control architecture | Weighted-position PID + cascaded encoder velocity loop + curvature-based speed scheduling | Directly addresses "usable speed" objective, not just top speed | Section 11/12 control theory reasoning | More tuning complexity than simple PID | High (as an architecture; specific gains require empirical tuning) |

---

# 26. SCHEMATIC CHANGE LIST

| # | Existing design | Problem | Proposed change | Reason | New component | Impact on PCB | Impact on firmware | Priority |
|---|---|---|---|---|---|---|---|---|
| 1 | H1 header exposes only 6 free analog Nano pins for a claimed 16-channel sensor | Physically cannot read 16 analog channels as drawn | Add 2× 74HC4051 (or equivalent) 8:1 analog muxes between sensor array and Nano ADC pins, OR switch MCU to STM32F411 with native 16-ch ADC | Section 4.7/8.2 — hard electrical blocker | 74HC4051 ×2 (if Nano kept) | New SOIC16 footprints, 3 mux-select GPIO lines routed from MCU | New mux-scan read routine replacing direct analogRead() calls | **CRITICAL — blocks fabrication as-is** |
| 2 | No fuse anywhere in the power path | Unprotected short-circuit/overcurrent path from battery | Add a resettable PTC fuse (or one-time fuse + holder) between XT60 and the rest of the board | Section 4.8 — safety | PTC fuse, ~2A hold | New footprint near U4 | None | **HIGH** |
| 3 | No reverse-polarity protection | Miswired battery destroys U2/U5/U1 instantly | Add a P-channel MOSFET (or Schottky diode as simpler fallback) reverse-protection circuit at VBAT input | Section 4.8 — safety | 1× P-MOSFET (e.g. AO3401-class) or Schottky diode | New footprint near U4/F1 | None | **HIGH** |
| 4 | C3 (470µF) has no specified voltage rating | Risk of exceeding capacitor voltage rating on a fully-charged 2S pack (~8.4V) | Specify C3 as ≥16V rated | Section 4.3 | Same part, new spec field | BOM/spec change only, same footprint likely | None | **HIGH** |
| 5 | U6/U7 motor spec is "N20 motor" only | Blocks every downstream mechanical/electrical/firmware decision | Lock exact motor per Section 6 recommendation (or your own final choice) | Section 6 | Specific motor part | Connector footprint for motor+encoder leads | Encoder ISR/timer setup depends on exact CPR | **CRITICAL — blocks all downstream design** |
| 6 | ENC-R/ENC-L BOM lines ambiguous re: integration | Risk of buying redundant or missing hardware | Resolve based on final motor choice (Item 5) — delete these BOM lines if motor ships with integrated encoder | Section 4.2 | N/A or new encoder part (AS5600) depending on path | Encoder signal routing either from motor connector directly, or from new AS5600 breakout | Firmware encoder-read routine depends on which path | **HIGH** |
| 7 | U3 OLED pin order unverified | Risk of reversed power/signal wiring damaging the display or MCU I2C pins | Confirm exact purchased OLED module's pin order before finalizing silkscreen/footprint | Section 4.2 | Same part, verified pinout | Footprint/silkscreen correction if needed | None | **MEDIUM** |
| 8 | H1 pitch unverified | Cannot finalize connector footprint | Confirm pitch, or eliminate H1 entirely if Item 1's mux solution routes sensor signals directly to dedicated MCU pins instead of through a loose header | Section 4.2 | Possibly removed entirely | Footprint decision | None | **MEDIUM** |
| 9 | Q1 (2N2222) exact pinout unverified | Risk of miswired buzzer transistor | Confirm E-B-C pinout of the specific purchased part/fab | Section 4.10 | Same part | Footprint orientation only | None | **LOW** |
| 10 | R3 labeled "4.9 kΩ" | Non-standard E-series value, likely a typo for 4.7 kΩ | Confirm intended value; if typo, correct to 4.7 kΩ | Section 18 BOM note | Same part, corrected value | None | **LOW** |
| 11 | No power switch | Power currently controlled only by physically connecting/disconnecting XT60, inconvenient and a bit rough on the connector over many bench sessions | Add a small SPDT power switch inline on VBAT (after fuse/protection) for bench convenience | General good practice | 1× SPDT switch | New footprint | None | **LOW/OPTIONAL** |
| 12 | No battery voltage monitoring | Firmware cannot warn of low battery or compensate | Add a resistor-divider from VBAT to a spare ADC pin | Section 4.8, supports Section 17 fault handling | 2 resistors | New divider network | New low-battery check in firmware fault handler | **MEDIUM** |
| 13 | No explicit ground-plane/routing guidance exists yet (schematic-level, becomes a layout task) | Risk of motor-switching noise coupling into ADC/I2C | Carry forward Section 16 grounding/routing rules into the PCB layout phase | Section 4.6, Section 16 | N/A (layout practice, not a component) | Layout discipline | None | **MEDIUM** |

---

# 27. INFORMATION STILL REQUIRED

Stated plainly, without hiding uncertainty, per your instructions:

1. **Your actual competition's rulebook** — track dimensions, max robot size/weight, line color/contrast convention, battery chemistry restrictions, start/stop mechanism requirements. Everything in Section 2's PRD downstream of this is provisional until you supply or I can locate this document.
2. **Exact no-load RPM figures for the N20 magnetic-encoder motor family (Section 6, candidates 2–4) at each gear ratio** — the vendor pages reference a ratio-comparison table that did not extract as readable text in this research pass; pulling that table directly (or requesting it from the vendor) would let you compare these encoder-integrated options against candidate 8 on equal footing.
3. **Exact identity and datasheet of "RoboJunkies 16-channel sensor"** if that specific board (rather than the Pololu-equivalent reference used throughout this paper) is what you intend to buy — I could not locate a distinct, current datasheet under that exact name/vendor in this pass.
4. **Pricing for candidate 8 (12SG-N30VA-10 or equivalent)** — the source used confirms specs but I did not independently verify current price/availability in your region.
5. **Real tire/wheel grip coefficient on your actual track surface** — this fundamentally cannot come from a datasheet and must be measured empirically (Section 22), and it directly determines the real-world cornering speed ceiling referenced throughout Sections 12/14/25.
6. **8–15 fully-documented comparable FLF robot builds with complete, individually-sourced spec tables**, as originally requested in Section 3 — I was not able to locate this volume of individually verifiable, datasheet-grade documentation in this research pass; what's provided instead is the pattern-level synthesis in Section 3, which is a real but smaller-evidence-base result than what was asked for.
7. **A confirmed choice between the Nano-keep and STM32F411-upgrade MCU paths** — this is presented in Section 25 as a genuine open trade-off requiring your decision, not something the available evidence resolves to a single answer.
8. **Local supplier pricing/availability** in your specific region for nearly every BOM line item in Section 18/19 — the prices given are global reference points, not confirmed local-market quotes.

---

# 28. CONCLUSION

Your FLF-01 v1.0 schematic is a reasonable, mostly-correctly-wired starting point for a first working line follower, but it is not yet buildable as a genuinely fast, competition-grade robot, for reasons that are specific and fixable rather than vague. The single most urgent fix is electrical, not stylistic: **the 16-channel sensor array cannot be read by the Nano as currently wired**, and that one gap cascades into the MCU decision, the PCB routing plan, and the firmware architecture. The second most consequential gap is that **the motor — the component every speed, torque, and current calculation in this entire paper depends on — is still an unspecified placeholder** in your own BOM; Section 6 gives you real, sourced candidates and the math to choose between them once you decide how much you value the "integrated encoder, less peak speed" trade-off (candidates 2–4) against the "higher documented top speed, needs a field-added encoder" trade-off (candidate 8). Everything else — the driver, the power topology, the button/OLED/buzzer subsystem — is fundamentally sound and mostly needs safety hardening (fuse, reverse-polarity protection, a correctly-rated bulk capacitor) rather than redesign. Where this paper could not verify a number, it says so explicitly (Section 27) rather than inventing one, consistent with your original brief.

---

# APPENDIX A — FORMULAS USED

```
Wheel circumference:            C = π × D_wheel
Robot linear speed from RPM:    v = (RPM / 60) × C
Tractive force from torque:     F = τ / r_wheel
Max acceleration (traction-unlimited): a = F_total / m_robot
Capacitor stored energy:        E = ½ C V²
Battery max continuous current: I_max = Capacity(Ah) × C_rating
Buck converter dissipated power: P_diss = P_out × (1/η − 1)
Quadrature effective CPR:        CPR_quad = CPR_raw × gear_ratio × 4
IPC-2221-style trace sizing:     used qualitatively per Section 16, current-margin based, not a single closed-form equation reproduced here (consult IPC-2221 or a trace-width calculator directly using your final current numbers before fabrication)
```

# APPENDIX B — COMPONENT COMPARISON TABLES

See Section 6.1 (motors), Section 7 (drivers), Section 8 (sensors), Section 10 (MCUs), Section 14 (wheels/tires) — consolidated there rather than duplicated here to avoid drift between two copies of the same data.

# APPENDIX C — SOURCES

- N20 encoder motor datasheet (BJ-N20-12-1000): https://cdn.sparkfun.com/assets/a/3/9/9/9/28633_datasheet_N20_Motor_With_Encoder_and_Cable_pair.pdf
- N20 magnetic encoder motor family (1:50/1:100/1:298): https://littlebirdelectronics.com.au/products/n20-dc-motor-with-magnetic-encoder-6v-with-1-50-gear-ratio.json , https://www.adafruit.com/product/4639 , https://littlebirdelectronics.com.au/products/n20-dc-motor-with-magnetic-encoder-6v-with-1-298-gear-ratio.json
- N20-BT01 75:1: https://botland.com.pl/en/micro-n20-mp-series-motors-medium-power/12541-micro-motor-n20-bt01-751-220rpm-6v.html
- N20 1000:1 HPCB: https://littlebirdelectronics.com.au/products/1000-1-micro-metal-gearmotor-hpcb-6v
- GA12-N20 1:1000: https://zbotic.in/product/high-torque-n20-6v-15rpm-micro-dc-metal-gear-reduction-motor-reduction-ratio-11000/
- 12SG-N30VA-10 (candidate 8): https://rc.thingbits.in/products/6v-1000-rpm-dc-micro-metal-gear-motor-high-speed
- TB6612FNG datasheet: https://toshiba.semicon-storage.com/info/docget.jsp?did=10660
- TB6612FNG carrier: https://www.pololu.com/product/713
- DRV8833: https://www.ti.com/product/DRV8833 , https://core-electronics.com.au/drv8833-dual-motor-driver-carrier.html
- QTR-MD-16A sensor array: https://core-electronics.com.au/qtr-md-16a-reflectance-sensor-array-16-channel-8mm-pitch-analog-output.html
- JSumo XLINE 16: https://www.robotshop.com/es/products/jsumo-xline-16-sensor-array-board
- STM32F411 Black Pill: https://docs.zephyrproject.org/latest/boards/weact/blackpill_f411ce/doc/index.html , https://mischianti.org/weact-stm32f411ceu6-black-pill-high-resolution-pinout-and-specs/
- Arduino Nano spec: https://www.pishop.us/product/arduino-nano/
- ATmega328P ADC throughput reference: https://ece-classes.usc.edu/ee459/library/documents/ADC.pdf
- AS5600 magnetic encoder: https://tinkered.ai/components/as5600 , https://zbotic.in/as5600-magnetic-encoder-absolute-position-sensing-explained/
- 2S LiPo battery reference specs: https://ozrc.com.au/products/gens-ace-2s-bashing-400mah-7-4v-35c-soft-case-lipo-battery-jst-ohr-2p-gea4002s35js , https://lindinger.at/en/RC-Electronics/Drive-Set/Batteries/PICHLER-LiPo-battery-FliteZone-500-7-4V-e.g.120X/9795182
- Mini-360 buck converter: https://components101.com/node/2228
- XT60 connector rating: https://probots.co.in/xt60pw-f-female-connector-right-angle-pcb-mount.html , https://zbotic.in/xt60-vs-xt30-vs-ec5-connector-current-rating-comparison/

---

*This document was generated from your uploaded schematic (`SCH_FLF-01_1-FLF_Main-Schematic_2026-09-02.png`) and BOM (`FLF-01_Fabrication_BOM.xlsx`), combined with live web research current as of 2026‑09‑19. Where a claim could not be verified from a primary source, it is explicitly labeled. Treat Section 27 as your action-item list before committing to PCB fabrication.*
