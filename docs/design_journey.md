# Design journey — what went wrong and how I fixed it

This is a record of the problems I ran into while designing a sub-1 V current-mode bandgap reference based on Banba's topology. I built and simulated it in Cadence Virtuoso with Spectre and gpdk090, and redesigned the component values for this process rather than copying the paper's values.

Everything here is **pre-layout, schematic-level work**. Some numbers come from early versions of the circuit that were *not working correctly*. I've kept those results because they explain the debugging process, not because they represent the final design. The final results are collected in [results_table.md](results_table.md).

## 1. I had the OTA feedback polarity backwards

One of the biggest mistakes was in the OTA connections. I was getting DC operating points that looked reasonable with the right initial conditions, but I hadn't actually checked whether the circuit could start from power-off on its own.

When I finally ran a power-up transient **without nodesets**, `Vref` went to **983.0864 mV** instead of settling near the intended reference. The PMOS gate line was at **475.1035 mV**, and the startup node `X` was at **2.53777 mV**. These were readings from the *incorrect circuit*.

The problem wasn't the resistor values. I had connected the OTA inputs so that it reinforced the branch-voltage difference instead of correcting it: **positive feedback**. I fixed the polarity by swapping which branch the diode-side input transistor sensed. In the corrected circuit, **NM3 senses the `rs` branch (`net14`)** and **NM4 senses the Q0 branch (`net27`)**.

After that change, a power-up run at **27 °C** gave **509.9118 mV** at `Vref`, **781.4265 mV** on the gate line, and **5.247881 mV** at `X`. That was still an intermediate version, before the final resistor tuning.

The main lesson for me was that a DC result alone wasn't enough. Spectre could find a plausible solution when given initial conditions, even though the feedback was wired incorrectly. **I discarded the DC and TC results from before the polarity fix**, rather than treating them as valid design results.

## 2. I kept tuning `rs`, but the startup transistor was disturbing the temperature sweep

After fixing the feedback, the temperature response still wasn't behaving the way I expected. `rs` is supposed to help balance the rising PTAT contribution against the falling CTAT contribution, so I initially tried adjusting it.

With NM2 sized at **1 µm / 100 nm**, I got:

- **`rs = 36.2 kΩ`:** TC of **694.074 ppm/°C**, with `Vref` rising as temperature increased.
- **`rs = 54.3 kΩ`:** TC of **247.531 ppm/°C**, but the curve had a valley rather than the temperature trend I was trying to tune.

I also checked a couple of other possible explanations. I suspected OTA tail headroom at high temperature, but measured tail-node voltages of **501.468 / 413.156 / 281.109 mV** at **−40 / 27 / 125 °C** did not support that idea. I tried changing the PM7/PM8 load widths from **4 µm to 3 µm**. It changed the internal load-node voltage difference, but didn't fix the input offset; TC actually went from **247.531 to 252.865 ppm/°C**. I reverted the widths.

What finally pointed me in the right direction was measuring **NM2 current alongside the OTA input offset**. NM2 is part of the startup circuit. Once the bandgap is running, it should be nearly off, but the short-channel device was still drawing unwanted current from the PMOS gate line, especially at higher temperatures.

| Temperature | NM2 current, 100 nm channel | OTA offset (`Va − Vb`) |
|---|---:|---:|
| −40 °C | 9.45591 nA | −0.296 mV |
| 27 °C | 50.961 nA | −2.342 mV |
| 125 °C | 201.277 nA | −11.712 mV |

That mattered because the OTA had to compensate for the extra current at the gate line. The compensation showed up as a difference between the two branch voltages, and that difference fed through to `Vref`. The offset and leakage trends lined up, so I had a much better explanation for the strange temperature response than simply blaming `rs`.

I changed **NM2's length from 100 nm to 1 µm**, keeping its width at **1 µm**, and measured again:

| Temperature | NM2 current, 1 µm channel | OTA offset (`Va − Vb`) |
|---|---:|---:|
| −40 °C | 184.327 pA | +73.9 µV |
| 27 °C | 1.51745 nA | +28.8 µV |
| 125 °C | 9.61338 nA | −428.05 µV |

The unwanted valley disappeared. The circuit now had a falling temperature trend (**473.982 ppm/°C** in that intermediate configuration), which meant I could go back to tuning `rs` for the actual CTAT/PTAT balance instead of compensating for a startup-device problem.

**Important distinction:** this diagnosis applies to the *old 100 nm NM2 version*. It is **not** a proven explanation for the final circuit's FS-corner TC failure. I did not measure NM2 current in FS.

## 3. I needed to reduce supply current without upsetting the reference

The OTA initially used an ideal **5 µA** bias source. I replaced that with an on-chip PMOS bias transistor, **PM0 (6 µm / 1 µm)**, because the final circuit shouldn't depend on an ideal current source being supplied from outside the design.

That version drew **19.10447 µA** at **27 °C** — under the **20 µA** target, but only just. Rather than adjust one branch and potentially throw off the CTAT/PTAT balance, I scaled **all resistor values by 1.5×** and moved to non-silicided poly resistors.

The idea was straightforward: higher resistances reduce current for similar voltage drops. Scaling the resistors together preserves the ratios used to set the reference. In this topology,

$$
V_{ref}=R_0\left(\frac{V_a}{R_3}+\frac{\Delta V_f}{r_s}\right)
$$

so scaling `R0` along with the current-setting resistors can keep the output voltage roughly where it was while lowering the currents.

The measured current dropped from **19.10447 µA to 12.91932 µA**. After scaling, `Vref` was **510.4926 mV**. These are intermediate results; after final tuning, the NN circuit measured **11.912 µA** and **499.08 mV** at **27 °C**.

This change was useful because it reduced power without having to redesign the basic Banba current-summing idea.

## Other mistakes and things I had to check

Not every problem was a big circuit-design issue. Some were simple schematic errors or simulation settings that sent me in the wrong direction.

- **Basic connection mistakes:** Earlier schematics had a floating gate in the OTA load, a floating reset input, and a startup device drawn as PMOS instead of NMOS. I corrected those connections before trying to interpret the circuit's behaviour.
- **A false low-current state:** At one point `Vref` sat around **173–178 mV**, `Va` and `Vb` were around **0.43 V**, and the PNP currents were only in the nA range. Increasing OTA bias from **1 µA to 5 µA** helped the DC solver converge, but that did **not** fix the underlying feedback problem.
- **The wrong kind of OTA input stage:** I had wired two PMOS transistors as source followers with their drains joined, instead of a proper differential input pair. TC was around **5,800 ppm/°C**, and changing `rs` barely helped. I switched to an NMOS input pair. I suspected the PMOS approach also needed more headroom than a **1 V** supply allowed, but I didn't independently simulate that headroom claim.
- **Temperature sweeps starting in the wrong state:** Reusing nodesets from **27 °C** gave `Vref` readings of **237.8 / 173.4 / 96.3 mV** at **−40 / 27 / 125 °C**. Using nodesets from the **−40 °C** solution (`net11` around **0.305 V**) or splitting the sweeps helped at that stage. **The final design does not need nodesets.**
- **TC numbers that looked better than they really were:** Some parametric runs went near **980 mV** or got stuck around **95–235 mV**. The calculated TC by itself could be misleading, so I started checking the `Vref` curve for every run before accepting the number.
- **Confusing STB output:** Spectre's automatic `pm` output didn't represent the phase margin the way I initially expected for this loop's phase convention. I checked the phase at low frequency and at unity gain instead. The final **NN, 27 °C** phase margin was **71.8°**. I kept the `IPRB0` stability probe in both netlists because the reported simulations used it.

## How I arrived at the final resistor values

Once the feedback polarity and NM2 leakage issues were dealt with, tuning became more predictable.

With **`rs = 40.6 kΩ`**, I got an intermediate TC of **27.419 ppm/°C**, but the output voltage was still low: **473.047 / 471.716 / 473.852 mV** at **−40 / 27 / 125 °C**.

I then raised **`R0` from 174 kΩ to 184.4 kΩ** to bring `Vref` closer to **0.5 V**. `R0` sets the voltage produced by the mirrored output current, although it isn't a completely independent adjustment: it also changes the drain voltage of PM3. That's why the final TC changed slightly instead of staying at the previous tuned value.

The final **NN, 1.0 V** results were **29.6769 ppm/°C** and **500.222 / 499.08 / 501.512 mV** at **−40 / 27 / 125 °C**. These are pre-layout Spectre results, not silicon measurements.

## What didn't meet the target

I don't want the successful NN result to hide the parts that still need work.

- **Line regulation — not met:** **1.75 mV/V** over **0.9–1.1 V**, at **27 °C, NN**, against a target below **1 mV/V**. **Hypothesis, not measured as a cause:** PM3 drives `Vref` at about **0.5 V**, while PM1 and PM2 have drain voltages around **0.6 V**, so their unequal drain-source voltages may make the mirror error vary with supply. I'd test longer PM1–PM3 channels or a different mirror arrangement, then retune the design.
- **FS-corner TC — not met:** **70.30 ppm/°C**, against the target below **50 ppm/°C**. The other corners measured **NN 29.68, FF 31.56, SF 36.18, SS 46.79 ppm/°C**. At **125 °C**, the measured OTA offset was **−629.245 µV** in FS versus **−287.635 µV** in NN. That correlation suggests an avenue to investigate, **not a proven cause**. In this PDK, SF and FS retain typical resistor and PNP models.
- **PSRR frequency range is limited:** The NN, **27 °C** result was **57.4 dB** at low frequency, but PSRR stayed above **40 dB** only through **250.693 kHz**. I can't claim it meets that threshold at higher frequencies.

The startup overshoot and the high-frequency PSRR bump were not diagnosed, so I haven't assigned causes to them.

## What I would do differently in another revision

I'd keep the no-nodeset startup transient as an early check instead of trusting a DC operating point. I'd also measure currents around any sensitive node before spending time retuning resistors; that was the difference between guessing at the temperature problem and finding the NM2 leakage path.

For the remaining performance gaps, the next things to investigate would be the output current mirror's supply sensitivity and the FS-corner OTA offset. Those are **ideas for future tests**, not fixes I have already verified. I'd also replace the ideal **2 MΩ** startup resistor with a modeled physical resistor. The poly resistor model used here has **zero temperature-coefficient fields**, so resistor temperature drift is not represented in these simulations.

The bandgap was **not laid out**, and there are no parasitic extraction or post-layout results. Supply sweep, line regulation, PSRR and STB were not evaluated at other temperatures or process corners; noise and trimming were not completed. The startup checks cover **−40 °C and 125 °C** in all five corners, plus **27 °C in NN**.

**Monte Carlo: planned; pilot run, not completed.**

For the exact simulation settings and final measured values, see [results_table.md](results_table.md) and the repository netlists. Historical results above are included only to explain what I changed during development.
