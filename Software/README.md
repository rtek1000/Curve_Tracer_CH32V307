# Curve_Tracer_CH32V307

Fault locator for electronic circuits, comparing V.I. curves (voltage and current)

Operation based on this project: [Curve Tracer ATmega328](https://github.com/rtek1000/Curve-Tracer-ATmega328)

------------

Note: Work in progress, just demonstrative, not functional, sorry.

------------

## 📊 Operating Modes and Expected 3D Scanning (Software/PC)

The system is designed to collect three-dimensional data arrays `[X, Y, Z]` via a single physical analog channel, sending raw packets for continuous surface rendering in Python (using `PyQtGraph` or `Open3D`). The firmware must be capable of modulating the `DAC1` signal to deliver three types of advanced pin-to-pin analyses:

### Mode A: Frequency Sweep (Z-Axis = Frequency)
- **How ​​it works:** The firmware maintains a fixed amplitude of 1.4Vpp and alters the Timer overflow value (`TIM6->ATRLR`) at the end of each complete wave cycle. The DAC frequency increases progressively (e.g., from 100 Hz to 50 kHz).
- **Technical Objective:** To map the reactance and internal parasitic capacitance of the CPLD/Processor ports. As the frequency increases, the 2D plot opens into ellipses, creating a "tunnel" or 3D surface on the screen to detect subtle dynamic faults.

### Mode B: Amplitude Modulation (Z-Axis = Peak Amplitude)
- **How ​​it works:** The firmware uses the Floating-Point Unit (FPU) to calculate the sine wave in real-time, multiplying the points by a digital gain factor that increases linearly with each cycle (ramp envelope), starting from 0V up to the hard limit of 1.4Vpp.
- **Technical Objective:** To create a continuous 3D surface (cone or ramp shape). This allows for the analysis of semiconductor junction linearity and dynamic resistance across multiple simultaneous energy levels, highlighting leakage issues that occur only at specific voltages. ### Mode C: Dual-Frequency Summed Injection (Dynamic Spectrum Sweep)
- **How ​​it works:** The CH32V307 performs 100% software-based mixing of two pure frequencies (e.g., a slow 60 Hz sine wave summed with a fast 10 kHz one), digitally ensuring the combined peak never exceeds 1.4 Vpp, and outputs the composite signal via a single DAC pin.
- **Technical Objective:** The computer receives the composite signal and applies fast digital filters (High-Pass and Low-Pass). This allows for plotting high-frequency micro-ellipses "surfing" along the low-frequency static curve, providing ultra-fast thermal and capacitive mapping of the silicon in a single test shot.

------------

Other considerations for analysis:

1. Z-Axis via Frequency Sweep (Dynamic Impedance Analysis)
In complex digital components such as processors or the MAX V CPLD, I/O pins do not merely possess resistance or protection diodes; they also exhibit significant internal parasitic capacitances.
How to scale the signals: Keep the amplitude fixed (e.g., 1.4 Vpp; the value may vary depending on the lowest supply voltage of the device under test) and generate a sequence of sine waves, but increment the frequency with each cycle (e.g., start at 100 Hz, then 500 Hz, 1 kHz, 5 kHz... up to 100 kHz).
The resulting 3D graph:
X-Axis: Voltage ($V$)
Y-Axis: Current ($I$)
Z-Axis: Frequency ($f$)
What you see: The graph will form a "tunnel" or a surface. At low frequencies, you will see only the static diode curve. As the frequency (Z-axis) increases, the curve opens up into an elliptical shape due to the capacitive reactance of the chip's gates. Damaged pins or those with thermal degradation will exhibit unique, asymmetric deformations at higher frequencies.

2. Z-Axis via DC Voltage Bias (Parametric Sweep)
Often, a logic gate or internal transistor only manifests leakage or a defect when subjected to a constant bias voltage while an alternating current (AC) signal is superimposed upon it.
How to scale the signals: Use a composite waveform. Generate your 1.4 Vpp test sine wave, but add a DC voltage step (DC offset) to the signal with each complete cycle. For example: Cycle 1 (Offset = 0 V), Cycle 2 (Offset = 0.2 V), Cycle 3 (Offset = 0.4 V), etc. The resulting 3D graph:
X-axis: Instantaneous AC voltage
Y-axis: Instantaneous current
Z-axis: DC offset voltage (bias)
What you see: A surface showing how the dynamic behavior of the CPLD pin changes under DC voltage stress. It is the perfect tool for mapping the exact semiconductor breakdown or latch-up point.

3. Z-axis via Amplitude Modulation (Power Signature)
Evaluating the component at a single voltage step might miss failures that occur only at very low voltages or specific conduction thresholds.
How to scale the signals: Instead of using a fixed-amplitude sine wave, generate an amplitude-modulated (AM) sine wave—that is, a sine wave where the peak grows linearly with each cycle (enveloped by a ramp), starting from 0V up to the safe limit of 1.4Vpp.
The resulting 3D graph:
X-axis: Instantaneous voltage
Y-axis: Instantaneous current
Z-axis: Maximum cycle amplitude ($V_{peak}$)
What you see: Instead of a single line or flat ellipse, you plot a continuous surface (a cone or three-dimensional ramp). This allows for the analysis of component linearity across multiple energy levels simultaneously, highlighting subtle variations in dynamic resistance (curve slope) that would go unnoticed in standard testing.

4. Z-axis via Multi-Pin (Cross-Signature Matrix)
When testing processors and CPLDs, the most advanced test is not pin-by-pin against GND, but rather analyzing interference or coupling between neighboring pins.
How to scale the signals: Inject the 1.4Vpp signal into Pin A (and measure the current there for the Y-axis), but use extra channels to inject phase-shifted or static signals into the surrounding Pins B, C, and D. Sequentially toggle the active pin.
The resulting 3D graph:
X-axis: Voltage on the pin under test
Y-axis: Current on the pin under test
Z-axis: Adjacent pin index or neighbor's logic state
What you see: A three-dimensional map of crosstalk or current leakage between the internal channels of the CPLD or processor. If the chip's internal isolation is degraded, the curve for Pin A will change shape drastically depending on the electrical state of Pin B.

5. If the user specifies the integrated circuit's operating voltage before starting the test (e.g., 1.8V for CPLDs or 3.3V for processor), you can implement a software-based "Smart Safety" layer on the CH32V307.
The firmware will use this information to dynamically recalculate the mathematical saturation limits for the DAC1 and DAC2 registers. This physically prevents the microcontroller from generating any voltage that exceeds the dangerous conduction thresholds for that logic family, even when operating in Bridge mode.
Below is the exact mathematical logic the CH32V307 must execute in its firmware to ensure this real-time protection:
The Semiconductor Golden Rule ($V_{CC} + 0.3V$)
Almost all modern IC datasheets specify in the "Absolute Maximum Ratings" table that no I/O pin should receive a voltage higher than $V_{CC} + 0.3V$ or lower than $-0.3V$, as doing so risks causing chip latch-up or burning out the junctions.
If the user provides the nominal voltage ($V_{test}$), the firmware must automatically clamp the probe's voltage swing:
Safe Upper Limit: $V_{max} = V_{test} + 0.3V$
Safe Lower Limit: $V_{min} = -0.3V$
How the CH32V307 Translates This to the 12-bit DACs
The CH32V307's 12-bit DAC converts values ​​ranging from 0 to 4095 into voltages ranging from 0V to 3.3V. The conversion constant is:
$$\text{DAC step} = \frac{3.3\text{ V}}{4095} \approx 0.0008058\text{ V (or } 0.805\text{ mV per step)}$$
If the user enters into the PC interface that they will test the MAX V CPLD (1.8 V internal operation):
The software calculates the physical voltage limits:
Maximum Voltage = $1.8\text{ V} + 0.3\text{ V} = 2.1\text{ V}$
Minimum Voltage = $0\text{ V}$ (For total safety, avoiding negative virtual AC).
The software converts these voltages into register values ​​(0–4095):
VALOR_MAX_DAC = $2.1\text{ V} / 0.0008058\text{ V} = 2606$
VALOR_MIN_DAC = $0\text{ V} / 0.0008058\text{ V} = 0$

6. In the CH32V307, the internal operational amplifiers (referred to as OPA or AMP) have their input and output pins mapped to fixed physical GPIOs on Port A and Port B. Leveraging the 4 available hardware blocks—split between the generation (injection) and reading (return) circuits:
🏗️ Generation Side (DAC Output Attenuation)
Internal OPA1: Connected to the DAC1 output. It is configured via registers (firmware) using internal resistors to attenuate the maximum amplitude from 3.3V down to the safe test level of 1.4 Vpp. Its output feeds into the first external OPA356. [2]
Internal OPA2: Connected to the DAC2 output. It performs the same role as OPA1 when running dual-frequency mode or a dynamic virtual ground reference, independently limiting the voltage ceiling. [2]
📈 Reading Side (Signal Restoration before the ADC)
Since the signal returning from the probe is attenuated to a maximum of 1.4V, injecting it directly into the CH32V307 ADC (which reads up to 3.3V) would result in losing more than half of the 12-bit converter's resolution. [2]
Internal OPA3: Connected to the X-Axis (Voltage) reading line. It is configured via software as an amplifier with a gain of ~2.35x (3.3V / 1.4V = 2.35). It "stretches" the 1.4V signal back to the full 3.3V scale before passing it internally to the ADC1 pin.
Internal OPA4: Connected to the Y-Axis (Current) reading line. It applies the same ~2.35x gain, allowing ADC2 to utilize the full 4095-step resolution range, ensuring incredibly sharp 3D plots in Python.

------------

Note: Developed with the help of Google AI Gemini

------------

## Licence

#### Hardware:
Released under CERN OHL 1.2: https://ohwr.org/cernohl

#### Software:
This library is free software; you can redistribute it and/or modify it under the terms of the GNU Lesser General Public License as published by the Free Software Foundation; either version 2.1 of the License, or (at your option) any later version.

This library is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU Lesser 
