# TinyML-Enabled Intelligent Optical Communication System
 
A course project for **Optical Fiber Communication** (VIT Vellore), presenting a TinyML-based intelligent monitoring system that predicts signal loss / BER performance for an 8-channel CWDM optical link, simulated in **OptiSystem** and deployed on an **ESP-32 WROOM** microcontroller.
 
This work has been written up as an IEEE-formatted paper: *"TinyML-Enabled Intelligent Optical Communication System: Design, Implementation, and Comparative Performance Analysis"* (see `docs/`).
 
---
 
## 📡 Project Overview
 
Traditional optical link monitoring relies on fixed threshold-based mechanisms that miss nonlinear relationships between system parameters, causing high false-alarm rates and delayed fault detection. This project explores whether a very small neural network — small enough to run on a $8.50 microcontroller — can outperform threshold-based monitoring while using a fraction of the power and cost of an Edge ML solution (e.g., Raspberry Pi).
 
**Pipeline:** OptiSystem CWDM simulation → data collection & augmentation → compact neural network training → int8 quantization → deployment on ESP-32 WROOM → real-time inference over Serial.
 
---
 
## 🧪 OptiSystem Simulation Setup
 
An 8-channel CWDM link was modeled in OptiSystem to generate realistic, physically-grounded training data.
 
| Parameter | Unit | Min | Max |
|---|---|---|---|
| Bit Rate | Gbps | 10 | 100 |
| Transmission Power | dBm | -10 | 10 |
| Fiber Length | km | 50 | 500 |
| Attenuation Coefficient | dB/km | 0.2 | 0.35 |
| Chromatic Dispersion | ps/nm/km | 16 | 20 |
| Nonlinear Coefficient | W⁻¹km⁻¹ | 1.2 | 2.5 |
| Amplifier Gain | dB | 15 | 30 |
| Noise Figure | dB | 4 | 8 |
 
Fiber impairments — attenuation, chromatic dispersion, and nonlinear effects — along with amplifier gain and noise were included so the dataset captures both stable and degraded link conditions.
 
**Initial data collection:** 1000 samples, spanning BER (10⁻¹² to 10⁻³), Q-factor (2–12 dB), and loss percentage (0–35%).
 
---
 
## 🔁 Data Augmentation
 
To improve generalization, the original 1000 samples were expanded to 5000+ samples using a three-part augmentation strategy while preserving physical validity:
 
- **Interpolation** — linear mixing between adjacent samples, `d = α·D[a] + (1-α)·D[b]`, α ~ U(0,1)
- **Noise injection** — Gaussian perturbation, `d = D[c] + ε`, ε ~ N(0, 0.01)
- **Extrapolation** — controlled extension beyond observed samples within physical constraints, β ~ U(1, 1.2)
A statistical comparison confirmed the augmented dataset preserved the original distribution (e.g., BER mean shifted by only 0.47%, Q-factor mean by 0.61%).
 
---
 
## 🧠 Model Architecture
 
The final architecture reported in the paper, optimized under a strict `Memory(θ) < 5KB` constraint:
 
| Layer | Type | Neurons | Activation | Parameters |
|---|---|---|---|---|
| Input | Dense | 2 | Linear | – |
| Hidden 1 | Dense | 8 | ReLU | 24 |
| Hidden 2 | Dense | 4 | ReLU | 36 |
| Output | Dense | 1 | Linear | 5 |
 
**Total: 65 parameters (53 trainable) — 4.2 KB quantized (int8)**
 
**Training configuration:**
- Optimizer: Adam (α = 0.001, β₁ = 0.9, β₂ = 0.999)
- Loss: MSE + L2 regularization (λ = 0.001)
- Batch size: 32, up to 100 epochs with early stopping (patience = 15) — converged at epoch 78
- Validation split: 20%, stratified by loss range
- Post-training int8 quantization for deployment
> **Note:** The firmware currently in this repo implements an earlier **2 → 6 (ReLU) → 1 (Linear)** prototype. The 2 → 8 → 4 → 1 architecture above is the version reported in the paper. If the paper's architecture is the one you want published as the final result, the ESP32 sketch and `model.h` should be regenerated from that model before merging — otherwise, keep both and clearly label the 2→6→1 version as an earlier prototype in this README.
 
---
 
## 🔌 TinyML Deployment on ESP32-WROOM
 
Rather than using the full TensorFlow Lite Micro interpreter, the trained (quantized) weights are hard-coded into a lightweight C++ forward pass, keeping the firmware minimal and fast:
 
- Reads two floats (Q-factor, BER) over **Serial (115200 baud)**
- Runs the manual forward pass
- Prints the prediction back over Serial with 6-decimal precision
**Reported on-device performance (paper, int8 quantized model):**
 
| Metric | Value |
|---|---|
| Inference latency | 2.3 ms |
| Power consumption | 15 mW |
| Model size | 4.2 KB |
| Startup time | 15 ms |
| MTBF | 50,000 hours |
 
---
 
## 📊 Results
 
### Accuracy vs. alternative approaches
 
| Metric | TinyML (proposed) | Edge ML (Raspberry Pi 4) | Threshold-based | Non-intelligent |
|---|---|---|---|---|
| Accuracy (%) | 94.2 | 94.8 | 78.3 | N/A |
| MAE (%) | 0.87 | 0.82 | 3.45 | N/A |
| R² Score | 0.942 | 0.948 | 0.671 | N/A |
| False Alarm Rate (%) | 5.8 | 5.2 | 21.7 | N/A |
 
### Resource & cost efficiency
 
| Metric | TinyML | Edge ML |
|---|---|---|
| Inference latency | 2.3 ms | 45 ms |
| Power | 15 mW | 2500 mW |
| Memory | 4.2 KB | 131,072 KB |
| 5-year deployment cost | $93.25 | $903.00 |
 
**Headline result:** the TinyML model achieves accuracy within ~0.6% of the much heavier Edge ML solution, while using 99.4% less power, 99.997% less memory, and costing 89.7% less over a 5-year deployment — a large accuracy gain over simple threshold-based monitoring, at a very small trade-off versus the full Edge ML pipeline.
 
Statistical significance of the improvement over threshold-based monitoring was confirmed with a paired t-test (t = 12.47, p < 0.001).
 
---
 
## 🛠️ Repository Structure
 
```
├── optisystem/            # OptiSystem project file (.osd) and layout screenshot
├── data/                   # Q-factor / BER sweep results
│   ├── cwdm_ber_dataset.csv
│   └── raw/                # Original exported PDF report
├── model_training/         # Training script/notebook, TFLite export (model.h)
├── firmware/                # ESP32 Arduino sketch (manual forward-pass inference)
├── docs/                    # IEEE paper (PDF)
└── README.md
```
 
---
 
## 🚀 Hardware Used
 
- **ESP32-WROOM-32** development board
- USB Serial connection (115200 baud) for input/output
---
 
## 📚 Citation
 
If you reference this work, please cite:
 
> Jabeena A, Abisek S, Adithya T C, Viswin Kumar, Sanjay Krishna, Senapathy C, "TinyML-Enabled Intelligent Optical Communication System: Design, Implementation, and Comparative Performance Analysis," Department of Electronics and Communication Engineering / Department of Sensor and Biomedical Technology, Vellore Institute of Technology, Vellore, India.
 
---
 
## 🔭 Future Work
 
As identified in the paper:
- Multi-parameter integration (OSNR, dispersion, nonlinear effects)
- On-device adaptive/incremental learning
- Federated learning across distributed edge nodes
- Hardware acceleration for sub-millisecond inference
- Explainable AI for operator trust and regulatory compliance
## ⚠️ Limitations
 
- Requires retraining for updated network configurations
- Constrained to BER and Q-factor as inputs
- No redundancy against single-point failure
---
 
## 📚 Course Context
 
Developed as a course project for **Optical Fiber Communication**, Vellore Institute of Technology, exploring the intersection of photonic system simulation and TinyML for real-time optical network diagnostics.
