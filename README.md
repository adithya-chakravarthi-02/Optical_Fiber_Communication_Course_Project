# TinyML-Based BER Predictor for CWDM Optical Systems
 
A course project for **Optical Fiber Communication**, combining an 8-channel CWDM optical link simulation in **OptiSystem** with a lightweight neural network deployed on an **ESP32-WROOM** microcontroller (no TensorFlow Lite runtime — hand-rolled forward pass) to predict Bit Error Rate (BER) performance in real time.
 
---
 
## 📡 Project Overview
 
The goal of this project was to bridge optical system simulation with edge machine learning:
 
1. Simulate an 8-channel CWDM (Coarse Wavelength Division Multiplexing) optical communication link in OptiSystem and sweep system parameters to generate a dataset of Q-factor and BER values.
2. Train a small feed-forward neural network on this dataset.
3. Convert the trained model into a **TinyML** deployment that runs directly on an ESP32-WROOM — without relying on the TensorFlow Lite Micro library — by hard-coding the trained weights and implementing the forward pass manually in C/C++.
4. Verify that the microcontroller can take live Q-factor/BER-related inputs over Serial and produce a real-time prediction.
This demonstrates an end-to-end pipeline: **optical system modeling → data generation → ML training → edge deployment.**
 
---
 
## 🧪 OptiSystem Simulation Setup
 
The optical link was modeled as an 8-channel CWDM system:
 
| Parameter | Value |
|---|---|
| Number of channels | 8 |
| Channel wavelengths | 1550, 1570, 1590, 1610, 1630, 1650, 1670, 1690 nm |
| Bit rate | 2.5 Gbit/s per channel |
| Sequence length | 1024 bits |
| Samples per bit | 32 |
| Sample rate | 8 × 10¹⁰ Hz |
| Symbol rate | 1 × 10¹⁰ symbols/s |
| Fiber length | 100 km (Optical Fiber CWDM) |
| Sweep iterations | 20 |
 
**Signal chain per channel:** CW Laser → Pseudo-Random Bit Sequence Generator (NRZ/BiNRZ) → Mach-Zehnder Modulator (Analytical) → WDM Mux (8×1) → CWDM Fiber (100 km) → WDM Demux (1×8) → PIN Photodiode → Low-Pass Bessel Filter → 3R Regenerator → BER Analyzer, with Optical Spectrum Analyzers monitoring the multiplexed and demultiplexed signals.
 
The system was swept over 20 iterations, and for each iteration the **Q-factor** and **BER** were logged for every channel — producing the labeled dataset used for training.
 
---
 
## 📊 Dataset
 
BER Analyzer results were exported across all 20 sweep iterations and 8 channels, capturing:
- **Max Q-Factor** per iteration/channel
- **Min BER** per iteration/channel
These paired (Q-factor, BER) values form the training data for the TinyML predictor. Q-factor values ranged roughly from ~4 to ~165+ across the sweep, with corresponding BER values spanning from measurable error rates down to effectively error-free (BER → 0) operation — giving the model a wide dynamic range to learn from.
 
---
 
## 🧠 Model Architecture
 
A minimal fully-connected network was chosen to keep the model small enough for microcontroller deployment:
 
```
Input (2 features: Q-factor, BER)
        ↓
Dense Layer 1 — 6 neurons, ReLU activation
        ↓
Dense Layer 2 — 1 neuron, Linear activation (output)
```
 
- **Layer 1:** 2×6 weight matrix + 6 biases, ReLU activation
- **Layer 2:** 6×1 weight matrix + 1 bias, linear output
The model was trained offline (Keras/TensorFlow) and the resulting weights were extracted and hard-coded as `static const float` arrays for on-device inference — avoiding the overhead of the full TensorFlow Lite Micro interpreter.
 
A `.tflite` (`model.h`) export is also included in this repo for reference/comparison against the manual C implementation.
 
---
 
## 🔌 TinyML Deployment on ESP32-WROOM
 
Rather than running the model through TensorFlow Lite Micro, the trained weights were embedded directly into the firmware and the forward pass was implemented from scratch in C++:
 
- `predict(x1, x2)` performs the Dense(6, ReLU) → Dense(1) forward pass manually.
- The ESP32 reads two floating-point values (Q-factor and BER) over **Serial (115200 baud)**.
- The model computes and prints a prediction back over Serial with 6-decimal precision.
This "no-framework" approach keeps the firmware extremely lightweight and fast, which is ideal for resource-constrained microcontrollers like the ESP32-WROOM.
 
**Example serial interaction:**
```
Enter Q-factor and BER separated by space:
Example: 0.42 0.0001
 
Input Q-factor: 0.42
Input BER: 0.0001
Model Output Prediction: 0.183245
```
 
---
 
## 🛠️ Repository Structure
 
```
├── optisystem/          # OptiSystem project files (.osd) and sweep configs
├── data/                 # Exported Q-factor / BER sweep results (all channels, 20 iterations)
├── model_training/       # Training scripts / notebook, exported model.h (TFLite)
├── firmware/             # ESP32 Arduino sketch (manual forward-pass inference)
└── README.md
```
 
---
 
## 🚀 Hardware Used
 
- **ESP32-WROOM-32** development board
- USB Serial connection for input/output (host PC or serial terminal)
---
 
## 📈 Results
 
- Successfully simulated an 8-channel, 100 km CWDM link and generated a Q-factor/BER dataset across 20 sweep iterations.
- Trained a compact 2→6→1 neural network capable of learning the relationship between the swept parameters and link BER performance.
- Deployed the model on ESP32-WROOM with real-time inference over Serial, with no TensorFlow Lite runtime dependency — proving feasibility of BER prediction on low-cost edge hardware for optical network health monitoring.
---
 
## 🔭 Future Work
 
- Expand training data with additional sweep parameters (launch power, fiber length, dispersion).
- Compare manual inference accuracy/latency against TensorFlow Lite Micro on the same board.
- Extend to multi-channel simultaneous prediction and integrate with a live optical testbed.
---
 
## 📚 Course Context
 
Developed as a course project for **Optical Fiber Communication**, exploring the intersection of photonic system simulation and TinyML for real-time optical network diagnostics.
