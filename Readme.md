# ✋ Hand Gesture Recognition & Servo Control

A real-time **hand gesture recognition system** built with **MediaPipe** that reproduces gestures on physical servos through an **Arduino** and a **PCA9685** servo driver.
Tested on a **Raspberry Pi 4 (4 GB RAM)** using a **phone camera over Wi-Fi (DroidCam)** as the video input.

<!-- TODO: add a demo GIF or photo, e.g. ![Demo](docs/demo.gif) -->

---

## 🧠 Overview

- Real-time hand landmark detection and gesture classification with **MediaPipe**, using Google's pretrained Gesture Recognizer model (`gesture_recognizer.task`).
- Serial link from the Raspberry Pi to an **Arduino**, which drives 5 servos (thumb → pinky) through a **PCA9685** over I²C.
- **Gesture stability logic**: a gesture must be detected with confidence > 0.7 for **5 consecutive results** before a command is sent, and commands are rate-limited to one per 0.1 s. This avoids false servo movements.
- **Smooth motion** on the Arduino side: servos move toward the target in 2° steps every 20 ms instead of jumping.

---

## 🗂️ Project Structure

```text
iheb_hand_gesture_recognition/
├── Arduino/
│   └── servo_control/
│       └── servo_control.ino      # Arduino sketch: serial commands -> servos via PCA9685
├── gesture_recognizer.task        # Google's pretrained MediaPipe Gesture Recognizer model
├── recognition.py                 # Base gesture recognition (non-optimized)
├── optimized_recognition.py       # Optimized recognition (higher FPS, cleaner output)
├── optimized_control.py           # Full system: gesture recognition + servo control
├── requirements.txt               # Pinned Python dependencies
└── README.md
```

The virtual environment (`mediapipe_env/`) is created locally in the next section and is not part of the repository.

---

## ⚙️ Requirements

### 🔹 Hardware

- **Raspberry Pi 4** (tested on 4 GB RAM)
- Phone with **DroidCam**, or a USB webcam (see [Using a USB webcam](#-using-a-usb-webcam))
- Arduino Uno or Mega (the sketch relies on `serialEvent()`)
- **PCA9685** 16-channel servo driver, connected to the Arduino over **I²C**
- 5 servomotors (SPT5632 or similar), on PCA9685 channels 0–4
- USB cable between the Raspberry Pi and the Arduino
- **External power supply for the servos**

> ⚠️ **Do not power the servos from the Arduino's 5 V pin.** Servos draw high current, which causes jitter and brownouts. Feed the PCA9685 `V+` terminal from a dedicated supply, and connect all grounds together (Arduino, PCA9685, supply). The servos accept **4.8 V – 8.4 V**, but many PCA9685 boards are rated for **6 V or less** on `V+`, so use a regulated **5–6 V** supply unless your board's documentation says otherwise. Size the supply for the servos' combined stall current (check the servo datasheet), not just their idle draw.

### 🔹 Software

- Raspberry Pi OS / Ubuntu, **64-bit** (MediaPipe ARM wheels require it)
- **Python 3.10 – 3.12** (`mediapipe==0.10.18` has no wheels for 3.13+)
- Arduino IDE with the **Adafruit PWM Servo Driver Library** (install from the Library Manager)
- DroidCam app on the phone (nothing to install on the Raspberry Pi)

---

## 🐍 Python Setup

```bash
python3 -m venv mediapipe_env
source mediapipe_env/bin/activate
pip install -r requirements.txt
```

`requirements.txt` is a pinned freeze of the full environment. The direct dependencies are:

- `mediapipe`
- `opencv-python`
- `numpy`
- `pyserial`

(The rest, such as `jax`, `matplotlib` and `scipy`, are pulled in by MediaPipe.) To install only the direct dependencies:

```bash
pip install mediapipe opencv-python numpy pyserial
```

---

## ▶️ How to Run

### 1. Start DroidCam on your phone

- Open DroidCam and leave it running.
- Make sure the phone and the Raspberry Pi are on the **same Wi-Fi network**.
- Note the video URL shown by DroidCam. `optimized_control.py` currently uses:

  ```text
  http://192.168.1.12:4747/video_feed
  ```

- The URL is hard-coded in `run()` (`cv2.VideoCapture(...)`). Update the IP to your phone's address; it can change whenever the router reassigns it.

### 2. Activate the Python environment

```bash
cd path_to_your_project_folder/
source mediapipe_env/bin/activate
```

### 3. Find your Arduino serial port

```bash
ls /dev/ttyACM* /dev/ttyUSB*
```

Typical results: `/dev/ttyACM0` (Uno/Mega) or `/dev/ttyUSB0` (clones). If you get *Permission denied*, add your user to the `dialout` group (`sudo usermod -aG dialout $USER`) and log in again.

### 4. Run the script

```bash
python3 optimized_control.py --arduinoPort /dev/ttyACM0
```

- Replace `/dev/ttyACM0` with your actual port (it is also the default).
- The script connects to the phone camera over Wi-Fi, detects gestures, and sends servo commands to the Arduino.
- If the Arduino is not found, the script still runs without servo control.
- Press **ESC** to exit.

### Command-line options

| Option | Default | Description |
|--------|---------|-------------|
| `--model` | `gesture_recognizer.task` | Gesture recognition model path |
| `--numHands` | `1` | Maximum number of hands |
| `--minHandDetectionConfidence` | `0.5` | Minimum hand detection confidence |
| `--minHandPresenceConfidence` | `0.5` | Minimum hand presence confidence |
| `--minTrackingConfidence` | `0.5` | Minimum tracking confidence |
| `--cameraId` | `0` | Camera index (unused with the DroidCam feed) |
| `--frameWidth` / `--frameHeight` | `320` / `240` | Capture size (capped at 320×240 for performance) |
| `--arduinoPort` | `/dev/ttyACM0` | Arduino serial port |

> The script enforces floors of 0.7 (detection), 0.6 (presence) and 0.6 (tracking), so lower values have no effect.

---

## 🔌 Arduino Setup

1. Install the **Adafruit PWM Servo Driver Library** in the Arduino IDE.
2. Open `Arduino/servo_control/servo_control.ino` and upload it to the board.
3. Wire the servos to PCA9685 channels **0–4** in this order: thumb, index, middle, ring, pinky.
4. If the servo range is off, adjust `SERVOMIN` (150) and `SERVOMAX` (600) in the sketch for your servos.

### Serial protocol

| Parameter | Value |
|-----------|-------|
| Baud rate | `9600` (8N1) |
| Line ending | `\n` |
| Servo command | `S,<thumb>,<index>,<middle>,<ring>,<pinky>` with angles 0–180 (clamped by the Arduino) |
| Other commands | `STOP` (hold current positions), `HOME` (all servos to 90°), `STATUS` (print current positions) |

Example: `S,90,90,90,90,90\n`. The Python script only sends `S,...`; the other commands can be typed in the Arduino Serial Monitor for testing.

### Gesture → servo mapping

Angles per servo, in the order thumb, index, middle, ring, pinky (defined in `gesture_to_servo_positions()` in `optimized_control.py`):

| Gesture | Thumb | Index | Middle | Ring | Pinky |
|---------|:-----:|:-----:|:------:|:----:|:-----:|
| `Open_Palm` | 90 | 90 | 90 | 90 | 90 |
| `Closed_Fist` | 180 | 180 | 180 | 180 | 180 |
| `Pointing_Up` | 180 | 180 | 90 | 180 | 180 |
| `Thumb_Up` | 90 | 180 | 180 | 180 | 180 |
| `Victory` | 180 | 180 | 90 | 90 | 180 |
| `ILoveYou` | 90 | 90 | 90 | 180 | 180 |
| `None` | 20 | 20 | 20 | 20 | 20 |

The pretrained model recognizes `None`, `Closed_Fist`, `Open_Palm`, `Pointing_Up`, `Thumb_Down`, `Thumb_Up`, `Victory` and `ILoveYou`. Any gesture without an entry above (currently `Thumb_Down`) falls back to `90` on all servos.

### Alternative: continuous finger angles

Instead of fixed positions per gesture, `optimized_control.py` can compute servo angles directly from the hand landmarks. In the servo control block, comment out the `gesture_to_servo_positions(...)` line and uncomment `calculate_finger_angles(hand_landmarks)`.

---

## 🔄 Workflow

1. The phone streams video via DroidCam over Wi-Fi.
2. MediaPipe extracts hand landmarks and classifies the gesture.
3. The stability logic confirms the gesture (confidence > 0.7, 5 consecutive results).
4. Python sends the servo command to the Arduino over serial.
5. The Arduino moves the servos smoothly to the target angles.

---

## 📷 Using a USB webcam

In `optimized_control.py`, replace the DroidCam URL with the camera index that `run()` already receives:

```python
cap = cv2.VideoCapture(camera_id)  # instead of the DroidCam URL
```

Then run with `--cameraId 0` (or the index of your camera).

---

## 🚀 Future Improvements

- Support multiple hands simultaneously
- Add more gestures with a custom model (e.g. MediaPipe Model Maker)
- Integrate with ROS 2 for robotic applications
- Develop a GUI for visualization
- Test on more powerful boards (e.g. NVIDIA Jetson) to improve performance and enable real-time multi-hand recognition

---

## 📄 License

Parts of `optimized_control.py` are adapted from the MediaPipe Gesture Recognizer example (© The MediaPipe Authors, Apache License 2.0).

<!-- TODO: add a license for this project, e.g. a LICENSE file (MIT or Apache-2.0) -->
