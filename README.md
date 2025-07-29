# 🎛️ MIDI-over-Network Setup

Set up MIDI communication between two computers over a local network to trigger video playback from the stage PC to the FOH laptop.

---

## 🧾 Overview

### 🎚️ Physical Setup

* **Stage Mixer:** [Midas MR18](https://www.midasconsoles.com/product.html?modelCode=0605-AAF) *(not directly relevant for MIDI control)*
* **Audio Interface:** [Cymatic LP-16 Live Player](https://cymaticaudio.com/product/lp-16-live-player-16-track-backing-track-system/)
* **Router:** Provides local Wi-Fi network (used for both in-ear app control and MIDI communication)
* **FOH Laptop:** Receives MIDI over the network and triggers video output (projector or screen)
* **Raspberry Pi (or Mini PC):** Connected to the Cymatic; sends MIDI over the network

📶 FOH Laptop is connected via **Wi-Fi** to the router.
🔌 Raspberry and Mixer are **wired** to the router.

---

## 🍓 Raspberry Pi Setup

Using a **Raspberry Pi 3 Model B**, but any Linux-compatible device will work.

---

### 💽 OS Installation

1. Download [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
2. Choose a Raspberry Pi OS (Lite version is lighter and faster)
3. Configure:

   * **Hostname**
   * **Username & Password**
   * **Wi-Fi settings**
   * **Enable SSH**

> 💡 These options can be set via Raspberry Pi Imager before writing the SD card.

---

### 🛠️ Software Installation

#### 🐧 For Debian/Ubuntu systems (Mini PC, etc.)

```bash
sudo apt update
sudo apt install rtpmidid
```

#### 🍓 For Raspberry Pi OS (manual build required)

1. **Install dependencies:**

```bash
sudo apt update
sudo apt install build-essential cmake libasound2-dev libavahi-client-dev git libfmt-dev
```

2. **Clone & build:**

```bash
git clone https://github.com/davidmoreno/rtpmidid.git
cd rtpmidid
mkdir build && cd build
cmake ..
make
```

3. **Install binary:**

```bash
sudo cp ./src/rtpmidid /usr/local/bin/
```

4. **Run rtpmidid:**

```bash
rtpmidid --port 5004 --name StageSession &
```

---

### 🎹 Connect MIDI Controller (Cymatic, AKAI, etc.)

1. Plug your MIDI device into the Raspberry
2. List available devices:

```bash
aconnect -l
```

3. Connect device to `rtpmidid` (replace `<client_id>` with yours):

```bash
aconnect <client_id>:0 128:0
```

> 🎯 `128:0` is usually the `rtpmidid` port.
> Use the output of `aconnect -l` to find your actual client number.

---

## 💻 FOH Laptop Setup (Windows)

This computer receives the MIDI signal wirelessly and controls video playback.

---

### ⚙️ Configuration

1. Download and install [**rtpMIDI** by Tobias Erichsen](https://www.tobias-erichsen.de/software/rtpmidi.html)
2. Open the application

🖱️ **Steps:**

* ➕ Add a new session
* ✅ Check “Enabled”
* 🔗 Select `StageSession` under “Directory” and click **Connect**


<img width="611" height="395" alt="rtpmidi-setup" src="https://github.com/user-attachments/assets/37b7c8da-f961-4d37-ad15-df3b4e4bc17b" />


> 📈 You should see latency metrics once connected.

---

## 🔁 Run on Boot (Raspberry Pi)

Ensure the MIDI service starts automatically at boot.

---

### 1️⃣ Create `rtpmidid` systemd service

```bash
sudo nano /etc/systemd/system/rtpmidid.service
```

```ini
[Unit]
Description=rtpmidid MIDI networking daemon
After=sound.target

[Service]
ExecStart=/usr/local/bin/rtpmidid --port 5004 --name StageSession
Restart=on-failure
User=<your-username>
WorkingDirectory=/home/<your-username>

[Install]
WantedBy=multi-user.target
```

---

### 2️⃣ Auto-connect MIDI ports with script

```bash
sudo nano /usr/local/bin/connect-midi-ports.sh
```

```bash
#!/bin/bash

sleep 5

rtpmidid_client=$(aconnect -l | awk '/client [0-9]+:.*StageSession/ { gsub(":", "", $2); print $2 }')

if [[ -z "$rtpmidid_client" ]]; then
  echo "Could not find rtpmidid ALSA client ID. Exiting."
  exit 1
fi

aconnect -l | while read -r line; do
  if echo "$line" | grep -q "^client [0-9]\+:.*type=kernel.*card=" && ! echo "$line" | grep -q "Midi Through"; then
    client_id=$(echo "$line" | awk '{print $2}' | sed 's/://')
    port_line=$(sed -n "/^client ${client_id}:/,/^client /p" < <(aconnect -l) | grep "'")
    if echo "$port_line" | grep -q " 0 "; then
      echo "Connecting ${client_id}:0 to ${rtpmidid_client}:0"
      aconnect "${client_id}:0" "${rtpmidid_client}:0"
    fi
  fi
done
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/connect-midi-ports.sh
```

---

### 3️⃣ Create service for MIDI connection

```bash
sudo nano /etc/systemd/system/rtpmidi-connect.service
```

```ini
[Unit]
Description=Connect MIDI devices to rtpmidid
After=rtpmidid.service
Requires=rtpmidid.service

[Service]
ExecStart=/usr/local/bin/connect-midi-ports.sh
Type=oneshot

[Install]
WantedBy=multi-user.target
```

---

### 4️⃣ Enable both services

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload

sudo systemctl enable rtpmidid.service
sudo systemctl enable rtpmidi-connect.service
```

Then test manually:

```bash
sudo systemctl start rtpmidid.service
sudo systemctl start rtpmidi-connect.service
```

---

## 🔌 Hotplugging Support (Optional)

Ensure that if you plug in the MIDI controller **after boot**, it still gets connected.

---

### 🧩 1. Create a udev rule

```bash
sudo nano /etc/udev/rules.d/99-midi-hotplug.rules
```

```bash
SUBSYSTEM=="sound", ACTION=="add", KERNEL=="midiC*", RUN+="/usr/local/bin/connect-midi-ports.sh"
```

> ⏱️ The `sleep 5` in the script ensures ALSA is ready when hotplugging.

---

### ♻️ 2. Reload udev rules

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

---

## 📽️ Controlling the Video

At this point, your FOH laptop should be **receiving MIDI over the network**, as if the controller were physically connected to it.

You can verify the connection in your **DAW** (e.g., Ableton Live, Reaper) or any **MIDI-compatible software**:

* The MIDI input should appear with the **session name** (e.g. `nicdenny`, `StageSession`, etc.)
* You can monitor incoming MIDI messages to ensure everything is working correctly.

---

### 🎛️ What Can You Control?

Once MIDI is flowing, you can use it to trigger video playback, switch scenes, or activate effects — depending on the software you use.

Some common tools include:

* 🎞️ **Resolume Arena** – Trigger video clips and effects via MIDI
* 🎭 **QLab** – Cue-based control for multimedia and theater
* 🎬 **VDMX** – Real-time visual performance software
* 🧰 **Custom applications/scripts** – For tailored show control
* 📺 **OBS Studio** – Great for switching scenes or controlling overlays

> 💡 MIDI messages (like notes or CCs) can be mapped inside each software to control specific actions (e.g. play video, change scene, apply effect).

---

### 🧩 Controlling OBS via MIDI (Coming Up)

I’ll explain how to use **OBS Studio** with a **MIDI plugin** to switch scenes, start/stop video sources, and control your stream or projection using MIDI messages from the stage.
