# ir-led-driver
Industrial IR LED constant-current source with 38 kHz modulation for textile thread detection

## Design status and operating envelope

This is a **buildable reference design, not a qualified industrial product**.
It uses an LM358-family amplifier, an N-channel MOSFET current sink, and an
NE555-family oscillator. Qualification requires the measurements below; no
simulation, assembled-board results, or EMC certification are claimed.

| Parameter | Design target / requirement |
|---|---|
| Input | Regulated, protected 5 V; **4.75–5.25 V at the head connector**, including cable drop |
| Emitter | One Vishay TSAL6200, 940 nm; do not substitute an unidentified remote LED |
| LED current | Nominal **100 mA during illumination**, approximately 50 mA average |
| Optical carrier | 38.00 kHz, adjustable; nominal 50% command duty |
| Cable | Approximately 3 m; all regulation and carrier generation at the head |
| Initial qualification range | 0–60 °C ambient, subject to measured component/LED temperatures |
| Extended-temperature option | SA555DR in place of NE555DR; requires renewed temperature qualification |

An LM358 is too slow to assume that switching its reference on and off will
produce clean 13 µs current pulses. Instead, regulate **100 mA continuously**
and bypass the LED during the dark half-cycle. A second, P-channel MOSFET is
the bypass switch; the required N-channel MOSFET remains the regulating element.
This trades power efficiency for avoiding normal PWM-induced amplifier saturation.

## 1. Complete schematic and connections

All grounds below are the same electrical ground. `SGND` denotes the Kelvin
connection at the bottom of R1, not a separately isolated supply.
Pin numbers are for the packages in the BOM; verify the footprint against each
manufacturer drawing before fabrication.

```text
J1.1 VIN (protected +5 V from controller)
    |
    +-- D2 SMF5.0A cathode; D2 anode --> GND (connector transient suppression)
    |
    +-- Q4 AO3401A drain
         Q4 source ------------------------------ V5 (head supply)
         Q4 gate -- R15 100k --> GND
         [Q4 body diode points from VIN to V5: reverse-polarity series element]

V5 ----+---- D1 TSAL6200 anode
       |          D1 cathode ------------------- LED_K
       +---- Q2 AO3401A source
                  Q2 drain --------------------- LED_K
                  Q2 gate = BYPASS_G
V5 -- R9 1k -- BYPASS_G -- R10 100R -- Q3 BC847B collector
                                               Q3 emitter --> GND
U4 /Q -- R11 2.2k -- Q3 base -- R12 100k --> GND

LED_K --------------------- Q1 IRLZ44NPBF drain
U1A OUT -- R2 1k ----------- Q1 gate
Q1 gate -- R3 100k --------- Q1 source = ISENSE
ISENSE -- R1A 10.0R -------- SGND
ISENSE -- R1B 10.0R -------- SGND (parallel: effective R1 = 5.00R)
ISENSE -- R4 10k ----------- U1A IN-
U1A OUT -- C3 4.7nF -------- U1A IN-
VSET ---------------------- U1A IN+

V5 --> U2 REF3312 IN; U2 GND --> SGND; U2 OUT = VREF (1.25 V)
VREF -- R5 15.0k -- VSET -- R6 10.0k -- SGND
VSET -- C4 100nF -- SGND

NE555 astable:
V5 -- R7 1.00k -- DISCH -- R8 8.20k -- RV1 2k -- TIMING
TIMING -- C5 1.00nF C0G -- GND
U3 pin 7 = DISCH; pins 2 and 6 = TIMING
U3 pin 3 = CLOCK76 --> U4 clock
U3 pin 5 -- C6 10nF -- GND

Divide by two:
U4 D = U4 /Q; U4 CLK = CLOCK76; U4 /PRE = V5
U4 /CLR = RESET_N
U4 /Q drives Q3 as above; Q is a diagnostic 38 kHz output

Reset:
V5 -- R13 10k -- RESET_N
U5 MCP120-450 /RESET --> RESET_N
U3 pin 4 --> RESET_N
J1.3 DISABLE_N --> RESET_N (open-drain ONLY; low means dark)
J1.2 GND --> GND
```

### Pin-level completion

| Device | Remaining connections |
|---|---|
| U1 LM358BIDR (SOIC-8) | 8=V5, 4=SGND; A: 1=OUT, 2=IN−, 3=IN+; unused B: 5=SGND, 6 tied to 7 (ground follower) |
| U2 REF3312AIDBZR (SOT-23) | IN=V5, OUT=VREF, GND=SGND; map named terminals using the manufacturer's DBZ drawing |
| U3 NE555DR (SOIC-8) | 1=GND, 8=V5; pins 2–7 as above |
| U4 SN74HCT74DR (SOIC-14) | 14=V5, 7=GND; used half: 1=/CLR, 2=D, 3=CLK, 4=/PRE, 5=Q, 6=/Q; unused half: 10=/PRE and 13=/CLR to V5, 11=CLK and 12=D to GND, 8 and 9 unconnected |
| U5 MCP120-450DI/TO (TO-92) | VSS=GND, /RESET=RESET_N, VDD=V5; map named terminals using the exact **D-bondout** package drawing, not transistor pin order |
| Q1 IRLZ44NPBF (TO-220) | 1=gate, 2=drain, 3=source; tab=drain, **not ground** |
| Q2, Q4 AO3401A (SOT-23) | 1=gate, 2=source, 3=drain |
| Q3 BC847B (SOT-23) | 1=base, 2=emitter, 3=collector |
| RV1 | Tie wiper to its TIMING-side end; use remaining end toward R8, so a lost wiper defaults to full resistance |

Mandatory capacitors:

| Reference | Value / connection |
|---|---|
| C1 | 47 µF, 16 V, low-ESR bulk: V5 to GND at connector/current loop |
| C2 | 1 µF, 16 V X7R: V5 to GND next to Q2 |
| C7–C11 | Five 100 nF, 16 V X7R: one directly across each IC's supply pins (U1–U5) |
| C12 | 1 µF, 16 V X7R: U2 VREF to SGND |

Provide test pads for V5, SGND, VREF, VSET, ISENSE, U1A OUT, LED_K,
CLOCK76, Q, and RESET_N. No floating CMOS inputs are permitted.
J1 may be a locking 3-position connector; use a fourth ground contact if the
disable signal uses a separate twisted pair.

**Input protection boundary:** upstream equipment must provide current limiting
(initial bench setting 200 mA), and fast overvoltage disconnection that keeps V5
within component ratings. D2 suppresses connector transients but **does not clamp
to a safe 5.25 V**: a “5 V TVS” can clamp well above the logic IC ratings.
Do not connect this board directly to a 24 V machine supply. Q4 handles modest
reverse connection, not arbitrary industrial surge energy. A sustained reversed
input forward-biases D2; upstream current limiting/fusing is essential.

## 2. How current regulation and modulation work

U2 supplies 1.25 V independent of supply variation. R5/R6 produce:

```text
VSET = 1.25 × 10.0k / (15.0k + 10.0k) = 0.500 V
ILED ≈ VSET / R1 = 0.500 / 5.00 = 0.100 A
```

U1 compares the sense voltage with VSET. Too little current makes its output
rise, increasing Q1 gate voltage; too much current lowers the gate voltage.
R1's source degeneration provides negative feedback. Both inputs are near
0.5 V, comfortably inside the LM358's ground-inclusive common-mode range.
The output need not reach the 5 V rail.

R2 isolates the capacitive gate load, R3 discharges the gate, and R4/C3 provide
frequency-dependent local feedback. The nominal R4/C3 corner is about 3.4 kHz;
this is **not a proven loop-bandwidth or stability specification**. Check ringing,
phase/settling behavior and current steps on the real board; tune compensation
only from measurements. C4 filters the reference and slows its startup.

When U4 /Q is low, Q3 and Q2 are off and current passes through D1.
When /Q is high, Q3 pulls Q2's gate down and Q2 bypasses D1. The LED sees only
Q2's small on-state voltage, while Q1/R1 still conduct approximately 100 mA.
The Q1 drain voltage changes by approximately the LED forward voltage at every
edge; finite output resistance and capacitive coupling can still cause current
glitches. **The bypass avoids routine loop restart; it does not prove clean edges.**

U5 holds the timer and divider reset during undervoltage/startup. Once U4 is
powered sufficiently to operate, reset makes Q=0 and /Q=1, selecting darkness.
The supervisor delay allows the reference/regulator to settle before clocking.
Holding DISABLE_N low keeps the LED dark but **does not turn off the 100 mA sink**.
The very-low-voltage power-up interval is not guaranteed glitch-free by CMOS
reset alone; measure it. If absolutely no startup flash is allowed, add a
qualified load-disconnect/interlock in the upstream supply.

Q1 gate voltage must fit the LM358's limited high-output swing at the worst
temperature and supply voltage. IRLZ44N is a low-threshold candidate, but its
low on-resistance specification at higher gate voltage is **not a guarantee**
of operation here. Require measured gate-drive margin (section 7). The linear
operating point must also fit the manufacturer's **DC safe operating area**;
do not select a replacement using only pulsed current or RDS(on).

An open LED during its commanded on-time breaks the regulating path and may
drive U1 high. On reconnection/bypass, overshoot is possible. The design has no
independent precise overcurrent shutdown and is not single-fault safe.
Controller interlocking, fault detection and a separate hardware cutoff are
required where a failed emitter could cause damage or unsafe machine operation.

## 3. NE555 calculations: 38 kHz and 50% duty

A conventional NE555 astable cannot produce exactly 50% duty with nonzero RA:

```text
tH = ln(2) × (RA + RB) × C
tL = ln(2) × RB × C
fclock = 1 / [ln(2) × (RA + 2RB) × C]
Dclock = (RA + RB) / (RA + 2RB) > 0.5
```

Use the NE555 at **76 kHz**, followed by the HCT74 toggle flip-flop:
D is connected to /Q so each rising clock edge reverses its state.
The HCT part has TTL-level inputs suitable for the bipolar 555; do not
substitute HC logic without checking the 555's guaranteed output-high level.

With RA=R7=1.00 kΩ and C=C5=1.00 nF:

```text
RB = [1 / (ln(2) × 76000 × 1nF) − 1000] / 2 = 8991 Ω
RB = R8 + RV1 ≈ 8.20k + 0.791k
tH ≈ 6.925 µs; tL ≈ 6.232 µs; timer duty ≈ 52.63%
fLED = fclock / 2 = 38.00 kHz
TLED = 26.316 µs; ideal light/dark times = 13.158 µs each
```

The nominal trim range is approximately **33.71–41.46 kHz at the LED**.
Set RV1 using a frequency counter, not its mechanical position. The formulas
neglect threshold/discharge errors, parasitics and IC temperature drift.
A C0G timing capacitor and metal-film resistors reduce passive drift; a trimmed
NE555 is not a crystal-stable source.

The divider produces equal numbers of oscillator periods in each state.
Its propagation delays and the Q3/Q2/LED transitions cause a small optical
duty error. Measure light using a fast photodiode; do not label the optical
waveform “exactly 50%” merely because Q is 50%.

## 4. Bill of materials and sourcing

Order manufacturer parts through authorized distributors (for example DigiKey,
Mouser, Farnell/element14, or RS). These are sourcing routes, **not verified
stock, price, or lifecycle guarantees**. Check availability and datasheet revision
before purchase. No firmware or extra software dependencies are needed.

| Ref | Qty | Manufacturer / example ordering part | Rating / purpose |
|---|---:|---|---|
| U1 | 1 | TI LM358BIDR | Improved LM358, SOIC-8; preferred over legacy LM358 for offset |
| U2 | 1 | TI REF3312AIDBZR | 1.25 V reference, SOT-23 |
| U3 | 1 | TI NE555DR | Bipolar timer, SOIC-8, commercial temperature grade |
| U3 alternative | — | TI SA555DR | NE555-family industrial-temperature variant; not an additional IC |
| U4 | 1 | TI SN74HCT74DR | Dual D flip-flop, TTL input thresholds, SOIC-14 |
| U5 | 1 | Microchip MCP120-450DI/TO | 4.5 V-class open-drain reset supervisor, TO-92 |
| Q1 | 1 | Infineon IRLZ44NPBF | N-MOSFET, TO-220; linear sink, verify gate margin/DC SOA |
| Q2, Q4 | 2 | Alpha & Omega AO3401A | P-MOSFET, SOT-23; LED bypass and reverse protection |
| Q3 | 1 | Nexperia BC847B,215 | NPN, SOT-23 |
| D1 | 1 | Vishay TSAL6200 | 940 nm IR LED, through-hole |
| D2 | 1 | Littelfuse SMF5.0A | Unidirectional TVS, SOD-123FL; see protection boundary above |
| R1A, R1B | 2 | Vishay TNPW080510R0BEEA | Two 10.0 Ω in parallel, effective 5.00 Ω; each 0.1%, 25 ppm/°C, 0805, 0.125 W; verify temperature derating |
| R5, R6 | 1 each | Vishay TNPW080515K0BEEA / TNPW080510K0BEEA | 15.0 kΩ / 10.0 kΩ, 0.1%, 25 ppm/°C |
| R2, R7, R9 | 3 | Yageo RC0805FR-071KL | 1.00 kΩ, 1%, 0805 |
| R3, R12, R15 | 3 | Yageo RC0805FR-07100KL | 100 kΩ, 1%, 0805 |
| R4, R13 | 2 | Yageo RC0805FR-0710KL | 10.0 kΩ, 1%, 0805 |
| R8 | 1 | Yageo RC0805FR-078K2L | 8.20 kΩ, 1%, 0805 |
| R10 | 1 | Yageo RC0805FR-07100RL | 100 Ω, 1%, 0805 |
| R11 | 1 | Yageo RC0805FR-072K2L | 2.20 kΩ, 1%, 0805 |
| RV1 | 1 | Bourns 3296W-1-202LF | 2 kΩ multiturn trimmer |
| C1 | 1 | Panasonic EEU-FR1C470 | 47 µF, 16 V electrolytic; polarized |
| C2, C12 | 2 | Murata GRM21BR71C105KA01L | 1 µF, 16 V, X7R, 0805 |
| C3 | 1 | Murata GRM21BR71H472KA01L | 4.7 nF, 50 V, X7R, 0805 |
| C4, C7–C11 | 6 | Murata GRM21BR71C104KA01L | 100 nF, 16 V, X7R, 0805 |
| C5 | 1 | KEMET C0805C102F5GACTU | 1 nF, 1%, 50 V, C0G, 0805 |
| C6 | 1 | Murata GRM21BR71H103KA01L | 10 nF, 50 V, X7R, 0805 |
| J1 | 1 | Phoenix Contact 1751248 (MSTBA 2,5/3-G-5,08) | 3-way board header; matching plug 1757022; verify mechanical fit |

Passive manufacturer numbers are ordering examples; verify current catalogue
ratings and footprints. Replacements must preserve tolerance, temperature
coefficient, capacitor dielectric, and semiconductor pinout/limits.

## 5. Sensor-head PCB recommendations

- Keep Q1, R1, D1 and Q2 together. Keep both LED and bypass current loops local:
  V5 → D1/Q2 → Q1 → R1 → bulk-capacitor ground.
- Route two independent Kelvin traces from R1: top to R4, bottom to U1/U2's
  quiet ground/reference return. Do not sense through power-current copper.
- Place R4/C3 at U1 and R2 at Q1's gate. Keep VSET and amplifier inputs away
  from CLOCK76, DISCH, Q3, and Q2 gate traces.
- Use a continuous ground plane with placement that steers timer/discharge
  current away from the sense/reference region; avoid a split-plane crossing.
- Put each 100 nF directly at its IC pins. Give the NE555 its own short
  supply/ground branch to the bulk capacitor, not a daisy chain through U1.
- Put D2 at J1 with a short, wide transient return. Separate the connector
  “dirty” protection area from the reference. Use approximately 0.5 mm or wider
  LED/supply traces on 1 oz copper; select widths for actual board temperature.
- Q1's TO-220 tab is live at LED_K. Provide mechanical restraint and spacing;
  use an insulating pad if attaching it to grounded metal. Do not replace it
  with an unverified tiny switching MOSFET solely to shrink the head.
- Keep the LED's optical axis repeatable. Use an aperture/lens and receiver
  baffle sized to the smallest thread; the TSAL6200's native beam is not a
  fine-thread spot. Prevent lint accumulation and provide access for cleaning.
- Use locking connectors, strain relief, standoffs, and appropriate enclosure
  sealing. Conformal coat electrical areas if needed, **not the optical window,
  connector contacts, or trimmer adjustment**.

## 6. Thermal, compliance and current accuracy

### Compliance and gate drive

At nominal LED VF=1.35 V:

```text
VDS(Q1), light = 5.00 − 1.35 − 0.50 = 3.15 V
VDS(Q1), dark ≈ 5.00 − 0.50 = 4.50 V
```

At local V5=4.75 V and a conservative LED VF=1.60 V, light-state VDS is still
2.65 V. Cable/protection drop must be subtracted before this calculation.
There is drain-voltage compliance, but **gate-drive compliance is a separate
constraint**: at minimum supply and temperature extremes, require U1 OUT to
remain at least 0.3 V below its measured upper swing limit while regulating.
If it cannot, reject that Q1/operating range or redesign the gate drive; do not
assume the typical MOSFET transfer curve guarantees production yield.

### Dissipation

Ignoring small bypass/protection losses, with continuous sink current:

| Element | Nominal dissipation |
|---|---:|
| Q1, illuminated interval | 0.100 × 3.15 = 0.315 W |
| Q1, dark interval | 0.100 × 4.50 = 0.450 W |
| Q1, 50% carrier average | **0.3825 W** |
| R1, all the time | 0.100² × 5.00 = **0.050 W** |
| D1, 50% carrier, VF=1.35 V | Approximately 0.0675 W electrical input |
| Q2 bypass | Small I²R loss during darkness; use hot RDS(on), not zero |

At 5.25 V and LED VF=1.10 V, Q1's 50% average is approximately 0.420 W.
The worst persistent dark/reset condition is approximately 0.475 W at 100 mA;
at +5% current, use approximately **0.50 W**. If the assembled Q1 thermal path
were 62 °C/W, that would mean about 31 °C junction rise. This is an estimate,
not an enclosure measurement. Verify DC SOA at hot junction and actual mounting;
target Q1 junction below 110 °C for margin.

The source always draws 100 mA, even when disabled. Budget **at least 130 mA**
for the head including bipolar timer/logic/reference/driver overhead and supply
transients; measure consumption. With 24 AWG copper at approximately
0.084 Ω/m/conductor, the 3 m cable has approximately 0.505 Ω loop resistance:
at 130 mA, drop is approximately 66 mV, excluding connectors and temperature.

TSAL6200's 100 mA continuous rating applies only under its datasheet conditions.
Its repetitive-pulse and power/temperature derating curves, not a surge rating,
govern operation here. Check the **actual** peak current, pulse width, duty,
ambient and lead/enclosure thermal conditions. A stuck-illuminated 100 mA at
high ambient may exceed the LED's derated continuous dissipation. It is a fault
requiring shutdown, not a permitted indefinite mode. IR is invisible: prevent
close viewing and assess optical exposure under IEC 62471 for the finished optics.

### Accuracy budget

This is a design budget, not measured accuracy. With the chosen precision parts:

| Contribution | Conservative calculation / treatment |
|---|---|
| Reference initial error | Use ±0.15% only if guaranteed by the ordered REF3312 grade |
| Divider, two 0.1% resistors | Worst-case ratio error approximately ±0.12% |
| Sense resistor | ±0.10% initially; self-heating and drift additional |
| Amplifier offset | Every 1 mV input offset contributes 0.20% current error at 0.5 V |
| Input bias through R4 and divider | Bound by `(abs(IB−)×10k + abs(IB+)×6k)/0.5`; every 100 nA per input gives up to 0.32% |
| MOSFET/loop residual error | Measure both light and dark plateaus and edge transients |

For example, 3 mV offset and 100 nA per input would give an initial arithmetic
bound of 0.15+0.12+0.10+0.60+0.32 = **1.29%**, before drift and dynamic errors.
Those example offset/bias values are not asserted as this amplifier's
all-temperature limits: use the **maximum** datasheet limits for the ordered
grade and full temperature range.

Across a 35 °C change from room temperature, a 25 ppm/°C sense resistor alone
adds about 0.088%; mismatched divider drift and reference/amplifier drift add
more. Do not substitute a legacy LM358 and retain the same error budget.
Use ±5% pulse-plateau current as the initial qualification target. Frequency
trim does not calibrate current; tighter current accuracy requires measuring
the actual error and selecting R1 or the divider accordingly.

## 7. Testing procedure and validation checklist

No automated test infrastructure exists in this repository. Hardware acceptance
requires an oscilloscope (preferably ≥50 MHz), short ground springs or differential
probes, a frequency counter, DMM, temperature measurement, a current-limited
supply, a fast photodiode, and the actual cable/receiver/optics.
These are proposed acceptance limits, not reports of passing tests.

1. **Unpowered inspection:** check pinouts, Q4 orientation, diode/capacitor
   polarity, shorts, Kelvin routes, and continuity against section 1.
2. **First power:** hold DISABLE_N low, apply 5 V with 200 mA current limit.
   Check VREF≈1.25 V, VSET≈0.500 V, ISENSE≈0.500 V, /Q high, and LED dark.
   If supply current is excessive or U1 rails, switch off and investigate.
3. **Carrier:** release disable, trim CLOCK76 to 76.00 kHz (or Q to 38.00 kHz).
   Record electrical and optical duty, including Q2 delays. Inspect the timer
   output for clean clock crossings without spurious flip-flop edges.
4. **Current waveform:** probe R1 differentially and calculate I=V/R1.
   This shows **total sink current**, not LED current in the dark interval.
   For LED current, use a suitable wideband current probe in D1's lead (or a
   temporary calibrated series shunt, accounting for its added voltage drop).
   Use the photodiode to independently verify optical darkness and pulse edges.
5. **Dynamic/stability test:** observe startup, disable/re-enable, supply steps,
   and Q1 drain transitions. Look for persistent oscillation, current spikes
   and long tails. Prove performance before reducing compensation.
6. **Corners:** repeat at 4.75/5.00/5.25 V at J1 and 0/25/60 °C after thermal
   soak, using several U1/Q1/LED samples. Record gate margin, reference,
   frequency, current, Q1 temperature and LED temperature.
7. **Faults, current-limited:** exercise open LED, shorted LED, open/short
   bypass switch, interrupted/reconnected cable, reset held low and stopped
   clock in each state. Reconnection overshoot must be captured, not averaged.
   Do not leave a stuck-bright LED running hot; verify the system shutdown.
8. **Machine/optical testing:** run the real receiver with motor/VFD/contactors
   active; vary distance, vibration, lint, ambient light, thread speed, colour,
   diameter and alignment. Record missed detections and false trips against
   application-specific acceptance limits.

Release checklist (mark only after recording evidence):

- [ ] On-pulse current 95–105 mA across the declared operating range.
- [ ] Electrical carrier within ±1% of 38 kHz after calibration and thermal soak.
- [ ] Optical duty 48–52%; current reaches its plateau within 1 µs of illumination.
- [ ] Off-state LED current <1 mA; no unintended secondary optical pulses.
- [ ] Peak LED current ≤120 mA and within the LED's derated repetitive limits;
      edges settle without sustained ringing. This 120 mA ceiling is a test
      criterion, **not a hardware-enforced current limit**.
- [ ] Gate-drive margin ≥0.3 V; Q1 hot DC SOA and thermal limits satisfied.
- [ ] Startup/brownout/reconnection flash behavior meets the application's limits.
- [ ] Fault detection/cutoff handles stuck-bright and loss of regulation.
- [ ] Supply remains within limits over the complete 3 m cable and connector set.
- [ ] EMC and environmental tests pass for the finished enclosure/install.
- [ ] Fine-thread detection meets agreed latency, missed-thread and false-trip limits.

If any item fails, the design is not released: document the failure and redesign
or restrict the operating envelope. In particular, an LM358/large MOSFET and a
resistively driven shunt cannot be presumed to meet the 1 µs target without data.

## 8. Cable management, EMC and receiver considerations

- Put this entire board at the sensor head; carry DC power rather than a switched
  LED-current loop down 3 m of wire. Do not run VSET, ISENSE or the MOSFET gate
  over the cable.
- Use a twisted power/return pair (24 AWG minimum starting point; check loss and
  mechanical requirements). If remote disable is required, use a separate
  twisted disable/return pair. J1.3 accepts only an open-drain contact referenced
  to head ground; do not connect a 24 V PLC output or a 5 V push-pull output.
- For a noisy installation or differing ground potentials, use an isolated or
  differential interface with a local open-drain reset output. Input filtering
  and protection must be designed for that interface; the bare RESET_N pin
  is **not** a qualified 3 m industrial control-line interface. If unused,
  leave the disable contact unconnected and keep RESET_N internal.
- Route away from motor/VFD cables; cross them at right angles where needed.
  Terminate shields to chassis with short/360° connections according to the
  machine bonding plan, normally at enclosure entry. Never use the shield as
  the LED return. Evaluate both-end bonding versus ground-loop constraints.
- Bipolar 555 supply spikes require local decoupling even though LED-path
  current is nearly constant. Keep edge loops small. If a ferrite/input filter
  is added, check DC drop, resonance, startup and transient response with C1.
- Determine applicable EMC standards with the equipment integrator: typically
  IEC 61000-6-2 immunity / IEC 61000-6-4 emissions, with IEC 61000-4-2 ESD,
  -4-4 EFT and -4-5 surge as applicable to the selected ports/environment.
  Test levels, coupling network, grounding, and performance criteria must be
  specified by the product standard; no compliance follows from D2 alone.
- A normal AGC TV-remote receiver may reject continuous carrier. Evaluate a
  fixed-gain sensing receiver such as TSSP4038, or a photodiode/TIA plus synchronous
  detection for partial beam attenuation. **This transmitter is not a complete
  thread detector.** Geometry, thread diameter, optical gap, detection speed and
  receiver thresholds remain application requirements; do not claim guaranteed
  fine-thread detection from emitter current alone.

## Manufacturer references

Use the latest manufacturer revision and the exact ordering suffix:

- [TI LM358 / LM358B family](https://www.ti.com/lit/ds/symlink/lm358.pdf)
- [TI NE555 / SA555 timer family](https://www.ti.com/lit/ds/symlink/ne555.pdf)
- [TI SN74HCT74](https://www.ti.com/lit/ds/symlink/sn74hct74.pdf)
- [TI REF3312](https://www.ti.com/lit/ds/symlink/ref3312.pdf)
- [Microchip MCP120/MCP130](https://www.microchip.com/en-us/product/MCP120)
- [Infineon IRLZ44N](https://www.infineon.com/part/IRLZ44N)
- [Alpha & Omega AO3401A](https://www.aosmd.com/sites/default/files/res/datasheets/AO3401A.pdf)
- [Vishay TSAL6200](https://www.vishay.com/docs/81010/tsal6200.pdf)
- [Vishay TSSP4038](https://www.vishay.com/docs/82458/tssp40.pdf)
