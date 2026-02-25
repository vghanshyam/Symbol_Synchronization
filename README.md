# 📡 Symbol Synchronization (Timing Recovery)

Symbol Synchronization, also called Timing Recovery, is the process in a digital communication receiver that estimates and corrects the optimal sampling instants of received symbols to minimize inter-symbol interference (ISI) and ensure accurate symbol detection.
- It ensures that the receiver samples the signal exactly at the correct symbol time.

  
It operates as a feedback control system that:

- Detects timing error in the received signal  
- Adjusts the sampling phase dynamically  
- Uses a Timing Error Detector (TED)  
- Uses a Loop Filter  
- Uses an Interpolator  

The objective is to align the sampling instants with the center of each symbol for reliable demodulation and minimum detection error.

## 📌 Project Overview

This project implements a **Symbol Synchronizer (Timing Recovery System)** based on the **Mueller & Müller (M&M) Timing Error Detector**.

The M&M algorithm is a decision-directed timing recovery method commonly used in digital communication receivers, especially for BPSK and QPSK modulation schemes.

The system dynamically estimates and compensates for symbol timing offset, aligning the sampling instants with the optimal symbol decision point to improve detection accuracy.

## 🎯 Why Timing Recovery is Required

In real-world communication systems:

- Independent transmitter and receiver clocks create timing mismatch  
- Channel propagation introduces unpredictable delay  
- Sampling phase offset occurs  
- Fractional timing errors introduce Inter-Symbol Interference (ISI)  

If timing recovery is not applied, symbols are sampled away from the optimal eye center, resulting in higher Bit Error Rate (BER).

## 📡 Digital Receiver Chain

```
ADC Input
   ↓
Matched Filter (SRRC)
   ↓
Symbol Synchronizer
   ├── Timing Error Detector (Gardner)
   ├── PI Loop Filter
   └── Fractional Delay Interpolator
   ↓
Carrier Recovery (Costas Loop)
   ↓
Symbol Decision
```
## ⏱ Symbol Synchronizer Architecture

```
                +----------------------+
Input Samples → |     Interpolator     |
                +----------+-----------+
                           |
                           v
                +----------------------+
                |      Gardner TED     |
                +----------+-----------+
                           |
                           v
                +----------------------+
                |     PI Loop Filter   |
                +----------+-----------+
                           |
                           v
                     μ (Timing Control)
                           ^
                           |
                       Feedback Path
```
# 🧠 Mueller & Müller Timing Error Detector

## 📘 Mathematical Equation

For complex signals:

```
e(n) = Re{ x(n) * d*(n-1) - x(n-1) * d*(n) }
```

---

### Where:

- `x(n)` = Current interpolated sample  
- `x(n-1)` = Previous sample  
- `d(n)` = Decided symbol  
- `e(n)` = Timing error  
- `Re{}` = Real part operator  
- `d*` = Complex conjugate of decided symbol

## ⚙ Loop Filter (PI Controller)

```
u(n) = u(n-1) + alpha * e(n) + beta * sum(e(n))
```

---

### Where:

- `alpha` → Proportional gain  
- `beta` → Integral gain  
- `e(n)` → Timing error signal  

Controls convergence speed and loop stability.

## 🐍 Python Implementation: Mueller & Müller Timing Recovery

```python
import numpy as np

# ==========================================================
# USER SETTINGS
# ==========================================================
INPUT_FILE = "Filter_out_IQ.bin"
OUTPUT_FILE = "Symbol_Sync_out_IQ.bin"
SPS = 4                # samples per symbol (adjust to your system)
GAIN = 0.3             # loop gain

# ==========================================================
# 1) LOAD 32-bit FLOAT IQ FILE
# Format assumed:
# I0(float32), Q0(float32), I1, Q1, ...
# ==========================================================
print("Loading file...")

raw = np.fromfile(INPUT_FILE, dtype=np.float32)

if len(raw) % 2 != 0:
    raise ValueError("File does not contain even number of float32 values (IQ mismatch).")

samples = raw[0::2] + 1j * raw[1::2]
samples = samples.astype(np.complex64)

print(f"Loaded {len(samples)} complex samples")

# ==========================================================
# 2) MUELLER & MÜLLER TIMING RECOVERY
# ==========================================================
print("Running timing recovery...")

mu = 0.0
out = np.zeros(len(samples) + 10, dtype=np.complex64)
out_rail = np.zeros(len(samples) + 10, dtype=np.complex64)

i_in = 0
i_out = 2

while i_out < len(samples) and i_in + 16 < len(samples):

    # Take current best sample
    out[i_out] = samples[i_in]

    # Decision device (hard slicer for QPSK)
    out_rail[i_out] = (
        (1.0 if np.real(out[i_out]) > 0 else -1.0) +
        1j * (1.0 if np.imag(out[i_out]) > 0 else -1.0)
    )

    # Mueller & Müller error calculation
    x = (out_rail[i_out] - out_rail[i_out - 2]) * np.conj(out[i_out - 1])
    y = (out[i_out] - out[i_out - 2]) * np.conj(out_rail[i_out - 1])
    mm_val = np.real(y - x)

    # Update timing estimate
    mu += SPS + GAIN * mm_val

    step = int(np.floor(mu))
    i_in += step
    mu -= step

    i_out += 1

# Trim unused samples
samples_out = out[2:i_out]

print(f"Output samples: {len(samples_out)}")

# ==========================================================
# 3) SAVE BACK TO 32-bit FLOAT IQ FILE
# ==========================================================
print("Saving output file...")

iq_out = np.empty(2 * len(samples_out), dtype=np.float32)
iq_out[0::2] = np.real(samples_out)
iq_out[1::2] = np.imag(samples_out)

iq_out.tofile(OUTPUT_FILE)

print("Done.")
print(f"Saved to {OUTPUT_FILE}")
```
