# Cavicapture Installation Guide - Step by Step

This guide walks you through installing Cavicapture v2 from scratch, starting with a fresh Raspberry Pi OS installation.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [System Preparation](#system-preparation)
3. [Python 3 Installation](#python-3-installation)
4. [Cavicapture Download](#cavicapture-download)
5. [Dependency Installation](#dependency-installation)
6. [Hardware Setup](#hardware-setup)
7. [Configuration](#configuration)
8. [Testing](#testing)

---

## Prerequisites

You will need:
- **Raspberry Pi** (3B+, 4, or 5 recommended)
- **Raspberry Pi Camera Module** (v2 recommended)
- **16GB or larger microSD card**
- **Power supply** (5V, 3A minimum)
- **LED lights** with **GPIO controller**
- **Ethernet or WiFi** connection
- **Computer** to access Pi (SSH or directly)

---

## System Preparation

### Step 1: Fresh Raspberry Pi OS Installation

1. Download [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
2. Insert microSD card
3. Select:
   - **OS:** Raspberry Pi OS (64-bit, Latest)
   - **Storage:** Your microSD card
4. Click "Next" → Configure:
   - Set hostname (e.g., `cavicapture`)
   - Enable SSH
   - Set username: `pi`
   - Set password
5. Write image to SD card

### Step 2: First Boot

1. Insert SD card into Raspberry Pi
2. Connect:
   - Power cable
   - Monitor/HDMI
   - Keyboard/Mouse (or use SSH)
3. Pi will boot - wait 2-3 minutes
4. Login with your credentials

### Step 3: Update System

```bash
# Update package lists
sudo apt-get update

# Upgrade all packages
sudo apt-get upgrade -y

# Install essential tools
sudo apt-get install -y git nano curl wget build-essential
```

This may take 5-10 minutes.

### Step 4: Enable Camera Interface

On older Raspberry Pi OS releases the camera can be enabled via the
configuration tool.  Run:

```bash
sudo raspi-config
```

Then navigate to **Interface Options** → **Camera** → **Enable**, exit and
reboot:

```bash
sudo reboot
```

> **Note for Pi 4 / 64‑bit OS (Bookworm and later):**
> the "Camera" item is often missing from the `raspi-config` menu.  this is
> because the libcamera stack is used by default and no software switch is
> required.  the camera device is enabled automatically by the device tree
> overlay; you can verify it with `vcgencmd get_camera` or simply run the
> libcamera test below.  if you are running a mixed/legacy environment you
> may still see a "Legacy Camera" setting under Interface Options – you do **not**
> need to enable this for picamera2/libcamera to work.

Test camera after boot (libcamera stack):
```bash
# On Bookworm the tools may be named `rpicam-hello` / `rpicam-jpeg`.
# Try the rpicam tool first, otherwise fall back to libcamera tools:
rpicam-hello || libcamera-hello --width 3280 --height 2464 -t 1000
# Capture test image (either tool):
rpicam-jpeg -o test.jpg || libcamera-still -o test.jpg
ls -lh test.jpg
```

---

## Python 3 Installation

### Step 1: Install Python 3

```bash
# Install Python 3 and pip
sudo apt-get install -y python3 python3-pip python3-venv python3-dev
```

### Step 2: Verify Installation

```bash
python3 --version
pip3 --version
```

Expected output:
```
Python 3.11.x (or later)
pip 20.x (or later)
```

### Step 3: Set Default Python to Python 3 (Optional)

```bash
# Check current python
which python

# Create alias (if still using Python 2)
echo "alias python=python3" >> ~/.bashrc
source ~/.bashrc
```

---

## Cavicapture Download

### Option A: Clone from GitHub (Recommended)

```bash
# Navigate to home directory
cd ~

# Clone the repository
git clone https://github.com/bsf2608/Cavicam-Py3.git

# Go into directory
cd cavicapture

# Checkout version 2.0
git checkout 2.0

# List files
ls -la
```

### Option B: Download ZIP File

1. Visit: https://github.com/bsf2608/Cavicam-Py3
2. Click "Code" → "Download ZIP"
3. Extract on your computer
4. Transfer to Pi via SCP:

```bash
scp -r cavicapture-master pi@<your-pi-ip>:/home/pi/
ssh pi@<your-pi-ip>
cd cavicapture-master
```

---

## Dependency Installation

### Option A: Recommended (Virtual Environment)

Using a virtual environment keeps your system Python clean.

```bash
# Navigate to cavicapture directory
cd ~/cavicapture

# Create virtual environment
python3 -m venv cavicapture_env

# Activate it
source cavicapture_env/bin/activate

# You should see: (cavicapture_env) pi@... $

# Upgrade pip
pip install --upgrade pip setuptools wheel

# Install all dependencies
pip install \
  opencv-python-headless \
  numpy \
  pillow \
  matplotlib \
  RPi.GPIO \
  picamera2

# note: libcamera-apps (system package) provides capture tools used in testing; install via apt if not present
```

### Option B: System-wide Installation

```bash
# Install all dependencies at once
sudo apt-get install -y \
  python3-opencv \
  python3-numpy \
  python3-pil \
  python3-matplotlib \
  python3-rpi.gpio \
  python3-picamera2 \
  sqlite3 \
  libcamera-apps
# note: legacy python3-picamera is not required on 64-bit OS
```

### Step 2: Verify Installation

```bash
# If using virtual environment, make sure it's activated:
# source cavicapture_env/bin/activate

# Test imports
python3 << EOF
import sys
print("Python version:", sys.version)

try:
    import cv2
    print("✓ OpenCV:", cv2.__version__)
except ImportError as e:
    print("✗ OpenCV error:", e)

try:
    import numpy as np
    print("✓ NumPy:", np.__version__)
except ImportError as e:
    print("✗ NumPy error:", e)

try:
    import RPi.GPIO as GPIO
    print("✓ RPi.GPIO installed")
except ImportError as e:
    print("✗ RPi.GPIO error:", e)

try:
    from picamera2 import Picamera2
    picam = Picamera2()
    print("✓ picamera2 available")
except ImportError as e:
    print("✗ picamera2 error:", e)

try:
    import sqlite3
    print("✓ SQLite3 available")
except ImportError as e:
    print("✗ SQLite3 error:", e)

try:
    import matplotlib.pyplot as plt
    print("✓ Matplotlib installed")
except ImportError as e:
    print("✗ Matplotlib error:", e)

print("\nAll dependencies OK!")
EOF

# Verify hardware camera with libcamera / rpicam
# (this confirms the kernel driver and physical connection)
# Try rpicam tools first (Bookworm); otherwise use libcamera commands:
rpicam-hello || libcamera-hello --width 3280 --height 2464 -t 2000
rpicam-jpeg -o camtest.jpg || libcamera-still -o camtest.jpg

echo "Check camtest.jpg for a captured image"

# simple Python test using picamera2
python3 test_camera.py

```

Expected output:
```
Python version: 3.11.x ...
✓ OpenCV: 4.8.0
✓ NumPy: 1.24.x
✓ RPi.GPIO installed
✓ picamera2: available
✓ SQLite3 available
✓ Matplotlib installed

All dependencies OK!
```

---

## Hardware Setup

### Wiring Diagram for LED Control

You need to connect an LED (or LED relay) to a Raspberry Pi GPIO pin to control lighting:

```
GPIO Pin (e.g., GPIO4) → 330Ω Resistor → LED Anode (+)
LED Cathode (-) → Ground (GND)
```

### GPIO Pin Reference (BCM numbering)

Common available pins:
```
GPIO4  (Pin 7)  - Default for cavicapture
GPIO17 (Pin 11)
GPIO27 (Pin 13)
GPIO22 (Pin 15)
```

### Test GPIO Control

```bash
# Test your LED light pin
sudo python3 << EOF
import RPi.GPIO as GPIO
import time

# Set GPIO numbering (BCM mode)
GPIO.setmode(GPIO.BCM)

# GPIO4 is the default (adjust if needed)
GPIO_PIN = 4

# Setup pin
GPIO.setup(GPIO_PIN, GPIO.OUT)

print(f"Testing GPIO{GPIO_PIN}...")

# Turn on
GPIO.output(GPIO_PIN, True)
print("Light ON")
time.sleep(2)

# Turn off
GPIO.output(GPIO_PIN, False)
print("Light OFF")
time.sleep(2)

# Turn on again
GPIO.output(GPIO_PIN, True)
print("Light ON (final test)")
time.sleep(2)

# Clean up
GPIO.output(GPIO_PIN, False)
GPIO.cleanup()
print("Test complete - Light OFF")
EOF
```

If GPIO test works, you should see your light turn on/off twice.

---

## Configuration

### Step 1: Create Configuration File

```bash
# Navigate to cavicapture directory
cd ~/cavicapture

# Copy example config
cp example_config.ini my_experiment.ini

# Edit with nano
nano my_experiment.ini
```

### Step 2: Basic Configuration

Edit the following values in `my_experiment.ini`:

```ini
[capture]
output_dir=/home/pi/captures          # Keep as is (you can change)
sequence_name=test_sequence           # Name this experiment

[camera]
ISO=100                               # Start low, increase if dark
shutter_speed=1500                    # Adjust for exposure

[pi]
GPIO_light_channel=4                  # Match your LED GPIO pin
```

### Step 3: Create Output Directory

```bash
# Create captures directory
mkdir -p ~/captures

# Set permissions
chmod 755 ~/captures
```

---

## Testing

### Test 1: Preview Image

Generate a single test image to verify camera and lighting:

```bash
# Activate virtual environment (if used)
# source ~/cavicapture/cavicapture_env/bin/activate

# Generate preview
sudo python3 ~/cavicapture/cavicapture.py --config ~/cavicapture/my_experiment.ini --preview

# Check result
file ~/captures/test_sequence/preview.png
display ~/captures/test_sequence/preview.png   # View image (if GUI)
```

✅ If `preview.png` exists and shows your sample, you're ready!

### Test 2: Short Capture Sequence

Capture 10 images (with 10-second interval = 90 seconds total):

**First, edit config:**
```bash
nano ~/cavicapture/my_experiment.ini
```

Change:
```ini
[capture]
duration=0.025              # ~1.5 minutes (0.025 hours)
interval=10                 # 10 seconds between captures
```

**Run capture:**
```bash
sudo python3 ~/cavicapture/cavicapture.py --config ~/cavicapture/my_experiment.ini
```

Monitor output - you should see:
```
Reading from config file: ...
Configuring camera...
Configuration complete.
Capturing 20260302-120530.png
Capturing 20260302-120540.png
... (continues)
```

### Test 3: Process Images

In a **new terminal** (keep capture running):

```bash
# Navigate to cavicapture
cd ~/cavicapture

# Activate virtual environment (if used)
# source ~/cavicapture/cavicapture_env/bin/activate

# Start processing
python3 caviprocess.py --config ~/cavicapture/my_experiment.ini
```

You should see:
```
Reading from config file: ...
Processing...
Updating record
... (processes image pairs)
```

### Test 4: Check Results

```bash
# View directory structure
tree ~/captures/test_sequence/

# Or without tree:
find ~/captures/test_sequence/ -type f | head -20

# Check database
sqlite3 ~/captures/test_sequence/capture.db "SELECT filename, area FROM captures LIMIT 5;"
```

Expected output:
```
20260302-120530.png|0
20260302-120540.png|12345
20260302-120550.png|23456
...
```

---

## Creating an Activation Script (Optional)

For convenience, create a script to activate everything:

```bash
cat > ~/cavicapture/activate.sh << 'EOF'
#!/bin/bash

# Activate Cavicapture environment
echo "Activating Cavicapture environment..."

cd ~/cavicapture
source cavicapture_env/bin/activate

echo "✓ Virtual environment activated"
echo "Run: python3 cavicapture.py --config <config.ini>"
EOF

chmod +x ~/cavicapture/activate.sh
```

Then use:
```bash
source ~/cavicapture/activate.sh
```

---

## Troubleshooting Installation Issues

### Issue: "No module named cv2"

```bash
# Make sure virtual env is activated (if used)
source ~/cavicapture/cavicapture_env/bin/activate

# Try, in order:
pip install opencv-python-headless
pip install --upgrade opencv-python-headless
```

### Issue: "Cannot import RPi.GPIO"

For testing on non-Pi machines:
```bash
# Skip RPi.GPIO, use mock:
pip install fake-rpigpio
```

Then modify cavicapture.py to use mock (commented out for now).

### Issue: "Camera not detected"

```bash
# Verify camera is enabled
sudo raspi-config
# Interface Options → Camera → Enable → Reboot

# Test camera directly (libcamera)
libcamera-still -o test.jpg

# Check camera with Python
python3 -c "from picamera2 import Picamera2; print(Picamera2())"
```

### Issue: "GPIO permission denied"

```bash
# Always use sudo for GPIO operations:
sudo python3 cavicapture.py --config my_config.ini
```

Or add user to GPIO group:
```bash
sudo usermod -a -G gpio pi
# Then reboot: sudo reboot
```

---

## Next Steps

1. ✅ **Installation Complete!**
2. 📖 Read [USAGE.md](USAGE.md) for detailed usage instructions
3. 🎯 Configure [example_config.ini](example_config.ini) for your experiment
4. 🔧 Run calibration if needed: `python3 calibrate.py --config config.ini`
5. 🚀 Start your first capture sequence

---

**Installation completed!** You're ready to capture images!

