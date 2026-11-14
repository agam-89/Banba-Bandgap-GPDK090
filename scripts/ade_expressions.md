# ADE settings and output expressions

These are the Cadence Virtuoso **ADE L / Spectre** settings and calculator expressions used to reproduce the documented **pre-layout, schematic-level** results for `BGR/banba_bgr_real_poly` in **gpdk090**. This is a record of the simulation setup, not a new simulation or a post-layout verification.

## Before running an analysis

- **Model library:** In ADE L, go to **Setup → Model Libraries** and use `/home/work/gpdk090_v4.6/models/spectre/gpdk090.scs`. Select `NN` for nominal results; change the model section to `SS`, `FF`, `SF`, or `FS` for the corresponding corner runs. The supplied netlists have `section=NN` as the starting point.
- **Accuracy:** Set **Simulation → Options → Analog → `reltol=1e-6`** for DC temperature sweeps. The recorded project notes found that the default `1e-3` made temperature-coefficient results less repeatable.
- **Initial conditions:** The final temperature sweeps and startup transients were run **without nodesets**. This matters because an initial condition can hide an incorrect startup state or feedback connection.
- **Supply:** For DC, AC, and `stb`, use `V0` as `vsource dc=1 mag=1 type=dc`. The `mag=1` is the supply's AC excitation for measuring PSRR. For startup, use the separate `vpulse` supply described below.
- **Probe:** Keep `IPRB0 (net024 net22) iprobe` in both netlists. It is between the OTA output (`net22`) and the shared PMOS gate line (`net024`). It acts as a zero-volt short in analyses other than `stb` and enables the recorded loop-gain measurement.

## DC temperature sweep and TC

Set the DC sweep variable to **Temperature**, from **−40 to 125 °C** with a **1 °C** step. Use `V0=1.0 V`, the selected model corner, `reltol=1e-6`, and no nodesets.

Add these named ADE outputs (each line below is the actual calculator expression, not a shell command):

**`TC`** — temperature coefficient from the span of the DC Vref curve, using the sweep's average Vref and temperature range:

```text
(ymax(v("/Vref" ?result "dc")) - ymin(v("/Vref" ?result "dc"))) / (average(v("/Vref" ?result "dc")) * 165) * 1e6
```

**`vref_m40`** — reference voltage at −40 °C:

```text
value(v("/Vref" ?result "dc") -40)
```

**`vref_27`** — reference voltage at 27 °C:

```text
value(v("/Vref" ?result "dc") 27)
```

**`vref_125`** — reference voltage at 125 °C:

```text
value(v("/Vref" ?result "dc") 125)
```

**`vref_wave`** — plot the entire Vref curve; checking its shape is important because a low TC number is meaningless if part of the sweep is in the wrong operating state:

```text
v("/Vref" ?result "dc")
```

**`offset_125`** — OTA input/branch-voltage difference `net27 − net14` at 125 °C:

```text
value(v("/net27" ?result "dc") 125) - value(v("/net14" ?result "dc") 125)
```

This offset was used to compare the hot FS and NN cases; the comparison **does not prove the cause** of the FS TC miss.

## Startup transient

For startup runs, replace the DC source with `analogLib vpulse` for `V0` using the documented settings:

| Parameter | Value |
|---|---|
| Voltage 1 | `0` |
| Voltage 2 | `1` |
| Delay | `10u` |
| Rise | `100u` |
| Fall | `1n` |
| Width | `1` |
| Period | `2` |
| DC voltage | `1` |

Run **tran from 0 to 500u**, setting the ADE temperature box to the required temperature, and use no nodesets. The recorded startup checks cover −40 °C and 125 °C in `NN`, `SS`, `FF`, `SF`, and `FS`; an additional 27 °C run was recorded for `NN`.

**`vref_tran`** — reference voltage through startup:

```text
v("/Vref" ?result "tran")
```

**`idd_tran`** — supply current through V0, with the sign selected so drawn supply current is positive:

```text
-i("/V0/PLUS" ?result "tran")
```

The following outputs take the value at the documented **450u** measurement point, after the supply ramp:

**`vref_450`**

```text
value(v("/Vref" ?result "tran") 450u)
```

**`idd_450`**

```text
value(-i("/V0/PLUS" ?result "tran") 450u)
```

**`x_450`** — startup-node voltage `X = net033`; a low settled X is consistent with startup drive switching off:

```text
value(v("/net033" ?result "tran") 450u)
```

## Supply sweep and line regulation

For the saved supply-sweep setup, make V0's **DC voltage** the ADE variable `vdd`. In **Variables**, use **Copy From Cellview**, then set `vdd=1` if needed so Spectre does not stop on an undefined variable.

The documented DC supply sweep was **0.6 to 1.2 V**, stepping by **10 mV**, at **27 °C**, `NN`. The line-regulation result was calculated over **0.9–1.1 V** from the corresponding Vref readings, rather than from an additional named expression in the context file.

## AC analysis and PSRR

Use the DC voltage source with **AC magnitude `mag=1`**, and run AC from **1 Hz to 100 MHz**, at **1.0 V, 27 °C, NN**.

**PSRR at 1 kHz** — the negative sign reports positive rejection in dB:

```text
value(-dB20(v("/Vref" ?result "ac")) 1e3)
```

In the documented result, PSRR was **57.4124 dB at 1 kHz** and was **above 40 dB only through 250.693 kHz**. This does not establish the same rejection at higher frequencies or other corners.

## Loop stability (`stb`)

Keep the **`IPRB0`** current probe in the circuit and select it for `stb`. Sweep **1 Hz to 100 MHz**, using **20 points/decade**, at **1.0 V, 27 °C, NN**.

**`ph_dc`** — low-frequency loop-gain phase (read at 1 Hz):

```text
value(phase(getData("loopGain" ?result "stb")) 1)
```

**`ph_ugf`** — loop-gain phase at the recorded unity-gain frequency:

```text
value(phase(getData("loopGain" ?result "stb")) 2.00931e6)
```

For this measurement, the recorded low-frequency phase was **179.997°**, and the phase at the unity-gain frequency (**2.00931 MHz**) was **71.8078°**. Because this Spectre phase convention starts near **+180°**, the reported phase margin was interpreted as:

```text
180 - (ph_dc - ph_ugf) = ph_ugf ≈ 71.8 degrees
```

The reported **71.8°** is for **27 °C, NN, pre-layout** only. Do not treat the simulator's raw `pm` row as the reported margin for this trace; the phase was checked directly.

## Generic crossing expression

The project notes also recorded the following **calculator syntax template**:

```text
cross(wave threshold 1 "falling")
```

Here `wave` and `threshold` are placeholders for the desired signal and threshold. This is a reference template, **not a recorded output with a specific signal and threshold already filled in**.

## What the recorded runs do not cover

The supply sweep, line regulation, AC PSRR and `stb` results were recorded at **27 °C, NN** only. They were not run across the other process corners and extreme temperatures. The bandgap has no layout or extracted parasitics, and the recorded results are **pre-layout** only.

**Monte Carlo: planned; pilot run, not completed.**

The **FS TC of 70.30 ppm/°C** and **line regulation of 1.75 mV/V** did not meet their targets. Their proposed explanations remain hypotheses unless supported by additional measurements. For the full results and limitations, see [`../docs/results_table.md`](../docs/results_table.md).
