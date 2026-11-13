# Pre-layout simulation results

This document records **schematic-level Spectre simulations** of the Banba current-mode bandgap reference in **Cadence Virtuoso / gpdk090 (90 nm)**. The final cell is `BGR/banba_bgr_real_poly`, powered at **1.0 V nominal**. Unless specified otherwise, results use the **NN (typical/TT)** model section. **These are not measurements from fabricated silicon, and no bandgap layout or post-layout results are available.**

Values are reproduced from the project's recorded simulation results. Figure links point to the corresponding project screenshots. Where the source documents a *plot reading* rather than a numerical marker, it is identified as such; no unverified plot values are added here.

## 1. Results versus targets

| Metric | Target | Pre-layout result | Conditions | Status |
|---|---|---|---|---|
| Reference voltage | Approximately 0.5 V | **499.08 mV** | 1.0 V, 27 °C, NN | **met** |
| Temperature coefficient (TC), NN | < 50 ppm/°C | **29.6769 ppm/°C** | −40 to 125 °C, NN | **met** |
| TC across process corners | < 50 ppm/°C | NN 29.68; FF 31.56; SF 36.18; SS 46.79; **FS 70.30 ppm/°C** | 1.0 V, −40 to 125 °C | **not met** (FS) |
| Operating supply range | 0.9–1.1 V | Operates over **0.9–1.1 V** | 27 °C, NN | **met** |
| Supply current | < 20 µA | **11.8662–11.9607 µA** | 0.9–1.1 V, 27 °C, NN | **met** |
| Line regulation | < 1 mV/V | **1.75 mV/V** | 0.9–1.1 V, 27 °C, NN | **not met** |
| PSRR | > 40 dB (frequency unspecified) | **57.4149 dB at 10 Hz**; remains > 40 dB only through **250.693 kHz** | 1.0 V, 27 °C, NN | **not met** beyond 250.693 kHz |
| Phase margin | ≥ 60° | **71.8°** | 1.0 V, 27 °C, NN | **met** |
| Minimum supply | Report | **748.223 mV** for Vref within 1% of its 1.0 V value | 27 °C, NN | **reported** |
| Startup | Successful startup | Documented transient tests settle to the DC solutions | 1.0 V ramp, NN/SS/FF/SF/FS as below | **met for tested cases** |

**Scope qualification:** Only TC and the specified startup cases were checked across all five process corners. Line regulation, supply sweep, PSRR and loop stability were measured at **27 °C, NN only**.

## 2. Temperature coefficient and reference voltage

**Method:** DC temperature sweep **−40 to 125 °C** in **1 °C** steps at **1.0 V**, `reltol=1e-6`, **no nodesets**. The source's TC calculation is:

```text
(ymax(Vref) - ymin(Vref)) / (average(Vref) * 165) * 1e6
```

| Corner | TC (ppm/°C) | Vref at −40 °C (mV) | Vref at 27 °C (mV) | Vref at 125 °C (mV) | TC target |
|---|---:|---:|---:|---:|---|
| NN | 29.68 | 500.222 | 499.08 | 501.512 | **met** |
| SS | 46.79 | 499.012 | 499.393 | 502.728 | **met** |
| FF | 31.56 | 501.22 | 498.71* | 500.46 | **met** |
| SF | 36.18 | 502.29 | 499.63* | 499.77 | **met** |
| FS | **70.30** | 498.018 | 498.556 | 503.664 | **not met** |

*The original record notes that the last digits of the FF and SF 27 °C values were not captured. Do not treat the displayed precision as more accurate than recorded. The unrounded NN TC recorded in ADE is **29.6769 ppm/°C**.*

**Context for corners:** SF and FS skew the MOS models but use typical resistor, capacitor, diode and PNP models; SS and FF also change those other model sections. All five recorded corner curves were smooth, without wrong-state jumps during these sweeps.

**FS check:** the measured OTA input offset (`net27 − net14`) at **125 °C** was **−629.245 µV (FS)** and **−287.635 µV (NN)**. The increased offset correlates with the worse FS TC; **its cause has not been proven**. In particular, NM2 leakage was *not measured* in FS and cannot be stated as the cause.

Figures: [NN](../results/tc/tc_nn.png) · [SS](../results/tc/tc_ss.png) · [FF](../results/tc/tc_ff.png) · [SF](../results/tc/tc_sf.png) · [FS](../results/tc/tc_fs.png) · [FS OTA offset](../results/tc/offset_125_fs.png).

**Ideal-op-amp comparison, not final silicon design:** the earlier ideal-VCVS / ideal-resistor reference recorded **499.724 mV at 27 °C** and **15.70 ppm/°C**. It is not directly equivalent to the final transistor-level implementation. [Comparison screenshot](../results/ideal_opamp/ideal_opamp_tc.png).

## 3. Startup verification

**Method:** transient with V0 rising **0 to 1 V over 100 µs** (`V1=0`, `V2=1`, `delay=10u`, `rise=100u`, `fall=1n`, `width=1`, `period=2`), running **0–500 µs**, with **no nodesets**. The reported settled values were taken at **450 µs**.

| Corner | Temperature | Vref at 450 µs (mV) | Supply current at 450 µs (µA) | Startup node X at 450 µs (mV) | Matches DC solution? |
|---|---|---:|---:|---:|---|
| NN | −40 °C | 500.222 | 11.9078 | 3.57 | yes |
| NN | 27 °C | 499.08 | 11.912 | 5.42 | yes |
| NN | 125 °C | 501.512 | 12.0025 | 8.54 | yes |
| SS | −40 °C | 499.012 | 10.3763 | 5.97 | yes |
| SS | 125 °C | 502.728 | 10.4976 | 13.97 | yes |
| FF | −40 °C | 501.216 | 13.9797 | 2.36 | yes |
| FF | 125 °C | 500.468 | 14.0451 | 5.72 | yes |
| SF | −40 °C | 502.292 | 11.9138 | 5.90 | yes |
| SF | 125 °C | 499.771 | 11.9485 | 14.10 | yes |
| FS | −40 °C | 498.018 | 11.8959 | 2.38 | yes |
| FS | 125 °C | 503.664 | 12.0638 | 5.67 | yes |

The measured startup node X is low after settling, consistent with the startup injection turning off. Cold startup overshoot was observed (**plot reading**, not a marker measurement), but its cause was not diagnosed; no overshoot cause is claimed.

The **27 °C NN** startup values are retained because they are recorded numerically in the source context; its screenshot was intentionally excluded from the required image collection. All other listed startup cases have screenshots in [`results/startup/`](../results/startup/).

## 4. Supply sweep, current and line regulation

**Method:** DC supply sweep at **27 °C, NN**, from **0.6–1.2 V** in **10 mV** steps. The intended operating interval is **0.9–1.1 V**.

| Supply (V) | Vref (mV) | Supply current (µA) |
|---:|---:|---:|
| 0.7 | 520.569 | — |
| 0.8 | 500.686 | — |
| 0.9 | 499.352 | 11.8662 |
| 1.0 | 499.08 | 11.912 |
| 1.1 | 499.002 | 11.9607 |

Unreported current values are left blank, not inferred.

| Interval | Measured line regulation | Target | Status |
|---|---:|---|---|
| 0.9–1.1 V | **1.75 mV/V** (0.350 mV change across 0.2 V) | < 1 mV/V | **not met** |
| 0.9–1.0 V | **2.72 mV/V** | < 1 mV/V | **not met** |
| 1.0–1.1 V | **0.78 mV/V** | < 1 mV/V | **met for this sub-range only** |

The minimum supply meeting the documented **within-1% Vref** criterion was **748.223 mV**. This does **not** redefine the intended **0.9–1.1 V** verification range.

**One combined figure** contains the Vref supply sweep, supply-current results and line-regulation values: [supply_sweep_vref.png](../results/supply_sweep/supply_sweep_vref.png). Separate `supply_current.png` and `line_regulation_values.png` files are intentionally omitted.

## 5. Power-supply rejection (PSRR)

**Method:** AC analysis at **1.0 V**, **27 °C**, **NN**, covering **1 Hz–100 MHz**. The DC supply source also provides **AC magnitude 1**.

| Frequency | Measured PSRR |
|---|---:|
| 10 Hz | 57.4149 dB |
| 1 kHz | 57.4124 dB |
| 100 kHz | 48.8153 dB |
| 10 MHz | 29.0121 dB |

The response stays **above 40 dB only until 250.693 kHz**. Therefore, a statement such as “PSRR > 40 dB across 1 Hz–100 MHz” would be incorrect. A small high-frequency bump is noted only as a **plot reading**; its cause has not been diagnosed.

Figure: [PSRR frequency response](../results/psrr/psrr_plot.png).

## 6. Loop stability

**Method:** Spectre `stb` analysis with the **IPRB0** probe, **1 Hz–100 MHz**, **20 points/decade**, **27 °C**, **1.0 V**, **NN**.

| Quantity | Recorded result |
|---|---:|
| DC loop gain | 46.8765 dB |
| Unity-gain frequency | 2.00931 MHz |
| Loop-gain phase at 1 Hz (`ph_dc`) | 179.997° |
| Phase at unity-gain frequency (`ph_ugf`) | 71.8078° |
| **Phase margin** | **71.8° — met** |

The documented Spectre loop-gain phase uses a convention that begins near **+180°**. The phase margin is determined from the phase change and the directly measured phase at unity gain; the simulator's raw `pm` output row was not used as the reported margin.

**IPRB0 stays in both netlists.** It is a zero-volt current probe between OTA output `net22` and PMOS gate line `net024`, behaving as a short in analyses other than `stb`.

Figures: [loop-gain and phase](../results/stb/stb_gain_phase.png) · [phase values](../results/stb/stb_ph_values.png).

## 7. Limitations and what I would change

**Unmet targets and measurement limits are part of the result, not omitted.** Proposed explanations and changes below are *hypotheses or untested ideas*, not demonstrated improvements.

| Limitation | Measured evidence | Hypothesis / what I would test next |
|---|---|---|
| **Line regulation — not met** | **1.75 mV/V** versus **< 1 mV/V**, 0.9–1.1 V, 27 °C NN | **Hypothesis (untested):** PM3's approximately 0.5 V drain differs from PM1/PM2 nodes near 0.6 V, so differing drain-source voltages can disturb mirror matching as supply changes. Test longer PM1–PM3 channels or a better mirror/cascode, then retune and remeasure. |
| **FS TC — not met** | **70.30 ppm/°C** versus **< 50 ppm/°C**. FS OTA offset at 125 °C **−629.245 µV**, NN **−287.635 µV** | **Hypothesis only:** offset may contribute to the hot FS deviation. The correlation does not establish causation. Examine offset and branch operating points across corners. Do **not** attribute FS behavior to NM2 leakage without an FS leakage measurement. |
| **PSRR bandwidth — limited** | Above **40 dB only through 250.693 kHz**; low-frequency PSRR **57.4 dB** | **Untested idea:** inspect OTA, mirror and compensation response at higher frequency before changing the circuit. |

**Simulation coverage not completed:** supply sweep, line regulation, AC PSRR and `stb` were not run at −40 °C / 125 °C or non-typical process corners; startup at 27 °C was not run for SS/FF/SF/FS; combined 0.9/1.1 V supply and process corner coverage, noise, trimming and completed mismatch study are not reported.

**Model and implementation limitations:** no bandgap layout, parasitic extraction or post-layout verification; startup pull-up **RS is an ideal 2 MΩ resistor**, not a poly layout; the poly-resistor model has zero TC fields, so resistor temperature drift is unmodeled; SF/FS retain typical resistor and PNP models. Cold startup overshoot and the high-frequency PSRR bump remain undiagnosed.

**Monte Carlo: planned; pilot run, not completed.**

## 8. Reproduction pointers

- Final circuit and sizing: [`README.md`](../README.md) and [`schematics/`](../schematics/).
- Supply-source versions: [`netlist/banba_bgr_dc_ac_stb.scs`](../netlist/banba_bgr_dc_ac_stb.scs) for DC/AC/STB and [`netlist/banba_bgr_startup.scs`](../netlist/banba_bgr_startup.scs) for startup. Keep **IPRB0** in both.
- Both repository netlists use model section `NN`. To reproduce a process corner, change `section=NN` in the model include line to `SS`, `FF`, `SF` or `FS`.
- Temperature sweeps use `reltol=1e-6`; ADE expressions and analysis settings are documented in [`scripts/ade_expressions.md`](../scripts/ade_expressions.md).

These paths describe the planned repository organization; links to images and additional Markdown files become active when those files are published.
