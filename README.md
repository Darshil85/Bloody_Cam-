# 🚨 Bloody Cam: Real-Time Motion Tracking and Smart Deterrence System 🎥

This project implements a smart surveillance prototype that uses **computer vision** to detect and track motion in real-time. It features automated pan-tilt control, face recognition to distinguish owners from intruders, and sends instant alerts via Telegram.

---

## ✨ Features

* **Real-Time Tracking:** Uses Python/OpenCV/YOLO for motion detection and centroid tracking.
* **Smart Deterrence:** Distinguishes between known and unknown faces to trigger appropriate responses (e.g., "Welcome home" vs. "Get Back").
* **Pan-Tilt Control:** An **ESP32** drives two **SG90 servo motors** mounted on a custom 3D-printed structure to track the target.
* **Alert System:** Integrated **Telegram Bot** for instant notifications upon detection.
* **Live Feed:** Local streaming available via **Flask** server (implementation assumed in separate files).

---

## 🛠️ Setup and Installation

### Hardware Required

* ESP32-WROOM/S3 Microcontroller
* USB Webcam
* 2x SG90 Servo Motors (or similar pan/tilt servos)
* 3D-Printed Pan-Tilt Mount
* Jumper Wires, Breadboard, 5V Power Supply

### Software Requirements

1.  **Arduino IDE:** For flashing the ESP32.
2.  **Python 3.x:** With the following libraries:
    ```bash
    pip install opencv-python numpy pyttsx3 pyserial ultralytics
    # For face recognition (facenet_pytorch requires torch)
    pip install torch torchvision facenet-pytorch Pillow
    # For additional features
    pip install python-telegram-bot flask
    ```

---

## ⚙️ Usage Guide

### Step 1: Flash the ESP32

1.  Open the **Arduino Code (Appendix C)** in the Arduino IDE.
2.  Ensure you have the **ESP32 board package** installed in the IDE.
3.  Connect the servo motors:
    * Horizontal Servo (Pan): Connect signal pin to **GPIO 13** on ESP32.
    * Vertical Servo (Tilt): Connect signal pin to **GPIO 12** on ESP32.
4.  Upload the code to your ESP32 board.

### Step 2: Run the Python Script

1.  Open the `bloody_cam_main.py` file.
2.  **Crucially, update the `SERIAL_PORT` variable** (e.g., `SERIAL_PORT = 'COM3'` or `/dev/ttyUSB0`) to match your ESP32's port.
3.  Run the main script from your terminal:
    ```bash
    python bloody_cam_main.py
    ```

The system will start in **SWEEP** mode ('I' command sent), looking for a person. Upon detection, it transitions to **TRACKING** mode ('F' command sent).

---

## 🎓 Academic Project Details

* **Institute:** Indian Institute of Technology Indore
* **Course:** ME-309: Instrumentation and Control Systems
* **Submitted to:** Dr I.A. Palani

---
