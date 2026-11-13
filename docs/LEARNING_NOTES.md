# Learning notes — understanding my Banba bandgap reference

These are my notes for explaining **why** the circuit is designed the way it is, rather than just remembering which device connects where. I start with the intuition, then connect it to the equations and the actual schematic.

The project is a **pre-layout, schematic-level** current-mode bandgap reference in **Cadence Virtuoso / Spectre**, using **gpdk090 (90 nm)**. The topology comes from Banba, but the component values were redesigned for this process. These are simulation results, not measurements from fabricated silicon.

For the full set of results and the development history, see [results_table.md](results_table.md) and [design_journey.md](design_journey.md).

## What problem am I trying to solve?

**Intuition.** A voltage reference should not move much when the chip gets hot or cold. The problem is that almost every device parameter changes with temperature. Instead of trying to find one device that never changes, the bandgap uses **two changes in opposite directions** and balances them.

**Circuit detail.** The PNP emitter–base voltage, `VEB`, is **CTAT**: it falls as temperature rises (roughly **−1.8 mV/K** in the project notes). A difference between the emitter–base voltages of PNPs at different current densities is **PTAT**: it rises with temperature. The aim is to combine CTAT and PTAT contributions so their temperature slopes nearly cancel.

This cancellation is not automatically perfect. Device models, the amplifier's error, and circuit loading still matter. That's why I measured the temperature coefficient rather than assuming the equation guaranteed it.

## Why use the Banba current-mode approach at a 1.0 V supply?

**Intuition.** A conventional bandgap adds the CTAT and PTAT **voltages** and usually ends up around **1.25 V**. That is awkward when the circuit only has a **1.0 V** supply. Banba's approach instead turns those voltage contributions into **currents**, combines the currents, and uses an output resistor to set the final voltage. This lets the reference be around **0.5 V**.

**Circuit detail.** The simplified final-reference equation is

\[
V_{ref}=R_0\left(\frac{V_a}{R_3}+\frac{\Delta V_f}{r_s}\right)
\]

Here `Va` is the voltage of the Q0 branch (`net27`), `R3` converts that voltage into one current, `ΔVf` is the PNP voltage difference, `rs` is the resistor that sets the PTAT current, and `R0` converts the combined current back into the output voltage.

The important design freedom is that **`rs` mainly adjusts the balance between the temperature contributions**, while **`R0` sets the output voltage level**. They are not perfectly independent in the real circuit: changing `R0` also changes PM3's drain voltage and can shift the temperature coefficient slightly. I saw that during final tuning.

## Why use two PNPs with different effective areas?

**Intuition.** Two identical devices operating at different current densities don't have exactly the same emitter–base voltage. Their **difference** is useful because it increases with temperature, giving the PTAT contribution needed to balance the falling `VEB`.

**Circuit detail.** The final cell uses `vpnp5` for both PNPs: `Q0` has `m=1` and `Q1` has `m=8`, giving **N = 8**. The difference is described by

\[
\Delta V_{EB}=V_T\ln(N)
\]

where `VT` is the thermal voltage. The context records about **53.8 mV at 27 °C** and a temperature slope of about **+0.18 mV/K** for `N = 8`.

The idea is not that the larger PNP somehow makes a fixed reference by itself. It creates a predictable **difference**, and `rs` turns that difference into a current.

## What do the PMOS mirror and OTA actually do?

**Intuition.** The current mirror makes the two sensing branches and the output branch carry corresponding currents. But equal currents alone are not enough: the two sensing branch voltages also need to be related correctly. The OTA watches those voltages and changes the shared PMOS gate voltage to pull them toward equality.

**Circuit detail.** `PM1`, `PM2`, and `PM3` are identical **pmos1v** devices with **W = 10 µm, L = 1 µm**, on a shared gate line (`net024`). `PM3` supplies the output resistor `R0`. `NM3` and `NM4` form the OTA input pair. Their **correct polarity matters**:

- `NM3` (the diode-side input device) senses the **`rs` branch**, `net14`.
- `NM4` senses the **Q0 branch**, `net27`.
- The OTA output is `net22`; `IPRB0` connects it to `net024` for the stability measurement.

The intended negative feedback brings `Va` and `Vb` close together. With equal branch currents and `R2 = R3`, the simplified current balance is

\[
I_{Q0}+\frac{V_a}{R_3}=\frac{V_b}{R_2}+\frac{\Delta V_f}{r_s}
\]

Using `Va = Vb` and `R2 = R3`, the resistor terms cancel, leaving

\[
I_{Q0}=\frac{\Delta V_f}{r_s}.
\]

So the shared PMOS current is the sum of a `Va/R3` contribution and a `ΔVf/rs` contribution. `PM3` copies that current into `R0`.

**Why the feedback sign deserves attention.** I initially connected the OTA inputs the other way around. A DC solution could look reasonable with initial conditions, yet a power-up transient without nodesets showed `Vref` latched at **983.0864 mV**. Swapping the sensed branches corrected the positive feedback. A plausible DC result is not proof that the loop is connected properly. The detailed debugging sequence is in [design_journey.md](design_journey.md).

## What is inside the OTA, and why is it not an ideal op-amp?

**Intuition.** An ideal amplifier is useful for checking whether the bandgap idea can work, but the actual chip has to bias its amplifier using real transistors. That amplifier also introduces offset and needs voltage headroom.

**Circuit detail.** The final OTA uses:

| Role | Devices | Final dimensions |
|---|---|---|
| Input pair | NM3, NM4 | 10 µm / 1 µm |
| PMOS mirror load | PM8 (diode), PM7 | 4 µm / 1 µm |
| Tail-current mirror | NM0 (diode), NM1 | 2 µm / 1 µm |
| On-chip bias | PM0 | 6 µm / 1 µm |

The earlier ideal-op-amp model used a gain-**1000** VCVS with a **0.78 V** series source and ideal resistors. That version gave **15.70 ppm/°C** in the recorded nominal test, but it didn't capture real OTA offset and bias effects. In the final design, `PM0` replaces an ideal **5 µA** bias source.

I also tried a PMOS "input pair" in an early version, but its drains were joined so it effectively behaved like source followers. `rs` barely influenced the output and TC was around **5,800 ppm/°C**. I moved to the NMOS input pair. The idea that a PMOS pair needed too much headroom was reasoning, **not a separate simulated proof**.

## Why choose these resistor values?

**Intuition.** The resistors are how the circuit translates voltage into current. Their **ratios** are as important as their absolute values. If I change every resistor by the same scale factor, I can reduce current without automatically changing the reference-voltage ratio.

**Circuit detail.** The final values are:

| Function | Device | Value | Why it matters |
|---|---|---:|---|
| Branch resistor | R3 | 450 kΩ | Sets `Va/R3` current |
| Other branch resistor | R2 | 450 kΩ | Matched to R3 for the branch-current cancellation |
| PTAT tuning resistor | R4 = `rs` | 40.6 kΩ | Sets how strongly the PNP voltage difference contributes |
| Output resistor | R5 = `R0` | 184.4 kΩ | Converts the combined current into `Vref` |

In the development process, scaling all the resistors **1.5×** reduced recorded supply current from **19.10447 µA** to **12.91932 µA**, while `Vref` stayed close at **510.4926 mV** in that intermediate version. In simple terms: increasing the resistance lowered current, while preserving the relevant ratios largely preserved the output. This was an intermediate version, not the final supply-current measurement.

The final `rs` tuning was **40.6 kΩ**. With an earlier `R0` of **174 kΩ**, the TC was **27.419 ppm/°C** but the voltage level was below the target. Raising `R0` to **184.4 kΩ** brought the reference near **0.5 V**, with the final NN TC at **29.6769 ppm/°C**. The small TC shift is a reminder that real mirror loading couples the adjustments.

These are **non-silicided poly** resistors (`resnsnpoly`) built from series segments rather than ideal resistor symbols, apart from the startup resistor discussed below. The model's resistor temperature-coefficient fields read **0**. This means the simulations do **not** include realistic resistor temperature drift; I would not present the simulated TC as a full physical-process guarantee.

## Why does the circuit need a startup block?

**Intuition.** A self-biased circuit can have a valid operating point where **nothing is flowing**. If it starts there, its own feedback cannot necessarily pull it out. Startup gives the circuit a temporary push and then gets out of the way.

**Circuit detail.** The startup node is `X` (`net033`). The devices are `RS` (an **ideal 2 MΩ** resistor from VDD to X), `NM2` (an NMOS pulling the shared PMOS gate line low), and `NS` (an NMOS that pulls X low when `Vref` rises).

- When `Vref` is low, `NS` is off. `RS` pulls `X` upward, switching `NM2` on. `NM2` pulls the PMOS gate line down, which makes the PMOS branches conduct.
- As the reference builds up, `NS` turns on and pulls `X` low. `NM2` then turns off so the normal feedback loop controls the gate line.

The startup senses **`Vref`, not `Vb`**. In the recorded investigation, healthy `Vb` at **125 °C** was around **0.44 V**, close to the stuck-state `Vb` of around **0.43 V**. Using `Vb` would make it harder to distinguish those two states.

The startup mechanism was checked by ramping the supply and running transients **without nodesets**. All documented runs settled to the corresponding DC solutions. The cold startup waveform has an overshoot in some corners; the cause was **not diagnosed**. Peak heights described in the project context are **plot readings**, not precise markers, so I leave them out here.

## Why did NM2 channel length matter so much?

**Intuition.** "Off" is not the same as "zero current." When a startup transistor is supposed to be off, a small unwanted current can still disturb the sensitive gate node. In this circuit, the amplifier reacts to that disturbance and accidentally introduces a temperature-dependent error.

**Circuit detail.** In the problematic version, `NM2` was **1 µm / 100 nm**. At **125 °C**, the recorded NM2 current was **201.277 nA**, with OTA branch-voltage offset (`Va − Vb`) of **−11.712 mV**. Increasing its length to **1 µm** reduced the measured NM2 current at **125 °C** to **9.61338 nA** and the corresponding offset to **−428.05 µV** in that intermediate test. I then retuned `rs`.

This is a **measured debugging result for the earlier version**, not the final corner-wide leakage measurement. In the final NN design, the recorded NM2 current at **125 °C** was **8.80064 nA**. I have **not** measured NM2 leakage in the FS corner, so I cannot claim it caused the FS TC miss.

## Why are C0 and C1 included?

**Intuition.** The OTA and current mirror form a feedback loop. If parts of the loop respond at different speeds, the correction can overshoot or oscillate. Small capacitors are included to shape that response; a capacitor is not there to make the DC bandgap equation work.

**Circuit detail.** The final schematic has **C0 = 1 pF** between `net27` and ground, and **C1 = 1 pF** between VDD and the PMOS gate line (`net024`). They are part of the compensation/stability arrangement. I verified overall loop stability with Spectre `stb`, but I did **not** record a separate capacitor-by-capacitor stability comparison, so I should not claim a measured improvement from either capacitor alone.

## What do line regulation, PSRR, and stability tell me?

**Intuition.** Temperature is only one way a reference can move. If the supply changes, I want the output to resist slow changes and supply ripple. I also want the feedback loop to settle instead of oscillating. These are related questions, but they are **not the same test**.

**Circuit detail.** All values in this section are **pre-layout, at 27 °C, NN**, unless stated otherwise.

- **Line regulation** measures the change in DC `Vref` as supply voltage changes. The measured result was **1.75 mV/V over 0.9–1.1 V**, against a target **under 1 mV/V**: **not met**. A **hypothesis**, not a verified cause, is that PM3 and PM1/PM2 have different drain-source voltages, so the mirror error changes with supply. Longer mirror channels or cascodes would be ideas to test, not completed fixes.
- **PSRR** measures how much an AC disturbance on the supply appears at the output. It was **57.4149 dB at 10 Hz** and **57.4124 dB at 1 kHz**; it stayed **above 40 dB only through 250.693 kHz**. It would be incorrect to call this 40 dB rejection over the entire frequency sweep.
- **Loop stability** was tested with Spectre `stb` at the probe `IPRB0`. The recorded unity-gain frequency was **2.00931 MHz**, the low-frequency loop-gain phase was **179.997°**, and the phase at unity gain was **71.8078°**. The interpreted phase margin is **71.8°** at **27 °C, NN**. The raw `pm` output was misleading with this loop-gain phase convention, so I used the phase readings directly.

`IPRB0 (net024 net22) iprobe` must **stay in both repository netlists**. It is a zero-volt current probe between the OTA output (`net22`) and the PMOS gate line (`net024`); it acts like a wire outside the `stb` analysis. All reported simulations were made with the probe present.

## What changes between process corners?

**Intuition.** A design that works at the typical device model might behave differently when manufacturing variation makes some transistors or other components faster, slower, stronger, or weaker. A corner is one predefined combination of those model changes, not a random sample of manufactured chips.

**Circuit detail.** The tested model sections were `NN`, `SS`, `FF`, `SF`, and `FS`.

- `NN` uses typical MOS, resistor, capacitor, diode and BJT models.
- `SS` and `FF` change the MOS models **and** their associated resistor, capacitor, diode and BJT model sections.
- `SF` and `FS` skew the MOS models but keep typical resistor, capacitor, diode and BJT sections. That distinction matters when trying to explain their behavior.

The measured pre-layout TCs over **−40 to 125 °C** were **29.68 (NN)**, **31.56 (FF)**, **36.18 (SF)**, **46.79 (SS)**, and **70.30 ppm/°C (FS)**. The target was **under 50 ppm/°C**: **FS did not meet it**.

At **125 °C**, OTA offset was **−629.245 µV in FS** versus **−287.635 µV in NN**. That correlation is worth investigating, but **the cause of FS degradation is unproven**. I would test amplifier-offset sensitivity before attributing the miss to any one device.

## How I reproduce and check the simulations

**Intuition.** A good number is only useful if I can reproduce it and check that the circuit actually stayed in the intended operating state during the sweep.

**Circuit detail.** The documented setup is **Spectre** through **ADE L**, the final cell `BGR/banba_bgr_real_poly`, with the gpdk090 model section selected from `NN/SS/FF/SF/FS`. The temperature sweep runs from **−40 to 125 °C**, in **1 °C** steps, with `reltol=1e-6` and **no nodesets** for the final results. I check the entire `Vref` curve, not just the reported TC expression, to catch wrong-state jumps.

For DC, AC, and `stb`, the source is `V0` with `dc=1 mag=1 type=dc`. For startup, `V0` is the documented pulse source. To reproduce a corner from the repository netlist, change `section=NN` in the model include line to `SS`, `FF`, `SF`, or `FS`.

The final NN temperature coefficient is calculated from the maximum and minimum of the simulated `Vref` sweep, normalized by its average and the documented temperature interval:

```text
(ymax(v("/Vref" ?result "dc")) - ymin(v("/Vref" ?result "dc"))) / (average(v("/Vref" ?result "dc")) * 165) * 1e6
```

The other recorded output expressions and source settings will be documented in [`../scripts/ade_expressions.md`](../scripts/ade_expressions.md).

## What these results do *not* establish

**Intuition.** A schematic simulation tells me how this circuit behaves under the models and conditions I actually tested. It doesn't prove how a fabricated or laid-out design will behave.

**Circuit detail.** The bandgap has **no layout**, **no parasitic extraction**, and **no post-layout results**. `RS` remains an **ideal 2 MΩ** startup resistor. The poly-resistor model has zero temperature-coefficient fields. Temperature corners and startup were checked in the documented conditions, but supply sweep, PSRR, and `stb` were **not** repeated across every temperature and process corner. Noise and trimming were not tested.

**Monte Carlo: planned; pilot run, not completed.**

The outstanding reported limitations are **line regulation of 1.75 mV/V (not met)**, **FS TC of 70.30 ppm/°C (not met)**, and **PSRR above 40 dB only through 250.693 kHz**. Possible fixes remain **hypotheses** until they are actually simulated.

## The short version I would use to explain the design

I used Banba's current-mode approach because a conventional voltage-summing bandgap needs more voltage headroom than this **1.0 V** supply provides. The two PNPs generate temperature terms that move in opposite directions. The resistors turn those terms into currents, the OTA and PMOS mirror establish the branch relationship, and `R0` converts the combined current into roughly **0.5 V**. Startup gets the circuit out of the zero-current state, while the compensation capacitors help the loop remain stable. The main debugging lesson was to check feedback polarity and startup leakage before trusting a good-looking DC or TC number.
