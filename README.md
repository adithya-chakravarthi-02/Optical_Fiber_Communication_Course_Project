# Optical_Fiber_Communication_Course_Project
TinyML BER predictor for an 8-channel CWDM optical link. OptiSystem simulates the link and generates Q-factor/BER data; a compact neural net (2→6→1) is trained and deployed on ESP32-WROOM via a hand-coded C++ forward pass — no TFLite runtime — enabling real-time BER prediction on low-cost edge hardware.
