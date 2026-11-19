# ✋ Hand Gesture Recognition & Servo Control

A real-time **hand gesture recognition system** built with **MediaPipe**, driving physical servos through an **Arduino** and a **PCA9685** servo driver.
Tested on a **Raspberry Pi 4 (4 GB RAM)** using a **phone camera over Wi-Fi (DroidCam)** as the video input.

<!-- TODO: add a demo GIF or photo, e.g. ![Demo](docs/demo.gif) -->

---

## 🧠 Overview

- Real-time hand landmark detection with **MediaPipe**.
- Gesture classification with the trained model `gesture_recognition.task`.
- Servo control through an **Arduino** (PCA9685 over I²C) to reproduce gestures on physical servos.
- **Gesture stability logic**: a gesture must be detected for N consecutive frames (`<N>`, see `optimized_control.py`) before the servos move, which avoids false triggers.

The phone's video feed is streamed over Wi-Fi, so no extra camera wiring is needed.

---

## 🗂️ Project Structure

```text
iheb_hand_gesture_recognition/
├── arduino/
│   ├── servo_control/
│   │   └── servo_control.ino    # Arduino sketch: drives servos via PCA9685 from serial commands
│   └── libraries/               # Libraries for the PCA9685 module
├── mediapipe_env/               # Python virtual environment (activate with bin/activate)
├── gesture_recognition.task     # Trained gesture recognition model
├── recognition.py               # Base gesture recognition (non-optimized)
├── optimized.py                 # Optimized recognition (higher FPS, cleaner output)
├── optimized_control.py         # Full system: gesture recognition + servo control
├── requirements.txt             # Python dependencies
└── README.md
```

---

## ⚙️ Requirements

### 🔹 Hardware

- **Raspberry Pi 4** (tested on 4 GB RAM)
- Phone with **DroidCam**, or a USB webcam (see [Using a USB webcam](#-using-a-usb-webcam))
- Arduino board (Uno, Mega, etc.)
- **PCA9685** 16-channel servo driver (connected to the Arduino over **I²C**)
- Servomotors (SPT5632 or similar), 5 used
- USB cable between the Raspberry Pi and the Arduino
- **External 5–6 V power supply for the servos**

> ⚠️ **Do not power the servos from the Arduino's 5 V pin.** Servos draw high current, which causes jitter and brownouts. Feed the PCA9685 `V+` terminal from a dedicated supply and connect all grounds together (Arduino, PCA9685, supply).

### 🔹 Software

- Raspberry Pi OS / Ubuntu (**64-bit** OS recommended, MediaPipe ARM wheels require it)
- Python 3.11+
- Arduino IDE
- DroidCam app on the phone (nothing to install on the Raspberry Pi)

---

## 🐍 Python Dependencies

- `mediapipe`
- `opencv-python`
- `numpy`
- `pyserial`

Installation:

```bash
python3 -m venv mediapipe_env
source mediapipe_env/bin/activate
pip install -r requirements.txt
```

Or install the packages manually:

```bash
pip install mediapipe opencv-python numpy pyserial
```

---

## ▶️ How to Run

### 1. Start DroidCam on your phone

- Open DroidCam and leave it running.
- Make sure the phone and the Raspberry Pi are on the **same Wi-Fi network**.
- Note the video URL shown by DroidCam, for example:

  ```text
  http://192.168.1.12:4747/video
  ```

- `optimized_control.py` contains this URL. Update it to match your phone's IP address (it can change whenever the router reassigns it) and keep the endpoint path DroidCam displays.

### 2. Activate the Python environment

```bash
cd path_to_your_project_folder/
source mediapipe_env/bin/activate
```

### 3. Find your Arduino serial port

```bash
ls /dev/ttyACM* /dev/ttyUSB*
```

Typical results: `/dev/ttyACM0` (Uno/Mega) or `/dev/ttyUSB0` (clones).

### 4. Run the script

```bash
python3 optimized_control.py
```

- Serial port: set it via `<TODO: CLI argument or constant name in optimized_control.py>`.
- The script connects to the phone camera over Wi-Fi, detects gestures, and sends servo commands to the Arduino.
- Press **ESC** to exit.

---

## 🔌 Arduino Setup

1. Install the PCA9685 library from `arduino/libraries/` (or via the Arduino Library Manager).
2. Upload `arduino/servo_control/servo_control.ino` with the Arduino IDE.
3. The Arduino listens for serial commands from Python and drives the 5 servos through the PCA9685.

### Serial protocol

| Parameter | Value |
|-----------|-------|
| Baud rate | `<TODO>` |
| Command format | `<TODO: e.g. one line per gesture>` |

### Gesture → servo mapping

| Gesture | Servo positions / action |
|---------|--------------------------|
| `<TODO>` | `<TODO>` |

---

## 🔄 Workflow

1. The phone streams video via DroidCam over Wi-Fi.
2. MediaPipe extracts hand landmarks.
3. `gesture_recognition.task` classifies the gesture.
4. Stability logic confirms the gesture over consecutive frames.
5. Python sends the command to the Arduino over serial.
6. The Arduino moves the servos accordingly.

---

## 📷 Using a USB webcam

In `optimized_control.py`, replace the DroidCam URL passed to OpenCV with a camera index:

```python
cap = cv2.VideoCapture(0)  # 0 = first USB camera
```

---

## 🚀 Future Improvements

- Support multiple hands simultaneously
- Add more gestures and retrain the model
- Integrate with ROS 2 for robotic applications
- Develop a GUI for visualization
- Test on more powerful boards (e.g. NVIDIA Jetson) to improve performance and enable real-time multi-hand recognition

---

## 📄 License

`<TODO: add a license, e.g. MIT>`
