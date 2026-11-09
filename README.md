# Robotics Dream Keeper Challenge — Winter 2026

**Participant:** Ayesha Qureshi
**Board:** D-Robotics RDK X5 (8GB) with the RDK Stereo Camera GS130W
**Long-term project:** An agricultural quadruped (SpotMicro-based) that inspects multi-crop farms in Sindh, Pakistan, and sends findings to farmers as voice messages in their own language.

---

## Stage 1 — Ignite

| Challenge | Status | Evidence |
| --- | --- | --- |
| 1. Board bring-up (flash, network, SSH, check-in) | ✅ Done | `stage1/evidence/` |
| 2. Sensor explorer (stereo camera over MIPI, ROS 2 node) | ✅ Done | `stage1/evidence/`, [`docs/camera-setup.md`](docs/camera-setup.md) |
| 3. First AI task (YOLO object detection) | ✅ Done | `stage1/evidence/` |

Community check-in: TODO (Discord permalink)

---

## Hardware

- RDK X5 8GB kit
- RDK Stereo Camera GS130W (dual SC132GS global-shutter sensors, MIPI CSI-2)
- 64GB microSD card
- Mac laptop as the development computer

## Software and dependencies

| Component | Version |
| --- | --- |
| RDKOS system image | 3.5.0 |
| TogetheROS (ROS 2 Humble) | Preinstalled with RDKOS |
| `tros-humble-mipi-cam` | 2.5.2 |
| RDK Studio (macOS) | Used to flash the image and manage device connections |
| YOLO model | YOLOv8n (`yolov8_640x640_nv12.bin`, preinstalled, COCO 80 classes) via `dnn_node_example` |

---

## Setup

### 1. Flash the image

1. Insert the microSD card into the Mac's card reader.
2. Open RDK Studio, choose the RDK X5 and the RDKOS image, and flash the card.
3. Insert the card into the X5's microSD slot and power on.

### 2. Connect to the network

Connect to the board first, either over USB-C (see Troubleshooting if it times out) or an existing Wi-Fi connection, then join Wi-Fi from the board's terminal:

```bash
nmcli device wifi rescan
nmcli device wifi list
nmcli device wifi connect "<network-name>" password "<password>"
```

Find the board's IP address:

```bash
hostname -I
```

### 3. SSH from the development computer

With the Mac on the same network:

```bash
ssh root@<board-ip>
```

### 4. Stereo camera

```bash
source /opt/tros/humble/setup.bash
ros2 launch mipi_cam mipi_cam_dual_channel_websocket.launch.py mipi_image_width:=1280 mipi_image_height:=1088 mipi_lpwm_enable:=True
```

Then open `http://<board-ip>:8000` in a browser on the same network and click **Web Display**.

Full walkthrough, including cable orientation and what each command does: [`docs/camera-setup.md`](docs/camera-setup.md).

### 5. YOLO object detection

```bash
source /opt/tros/humble/setup.bash
export CAM_TYPE=mipi
ros2 launch dnn_node_example dnn_node_example.launch.py dnn_example_config_file:=config/yolov8workconfig.json dnn_example_image_width:=640 dnn_example_image_height:=480
```

Then open `http://<board-ip>:8000` and click **Web Display**. Detected objects appear with labeled boxes.

- `export CAM_TYPE=mipi` tells the example to use the MIPI camera instead of a USB camera.
- `dnn_example_config_file` selects the model. `yolov8workconfig.json` points to YOLOv8n compiled for the X5's BPU (AI accelerator).
- The YOLO example runs **one** camera of the stereo pair, unlike the dual-camera launch in section 4.

---

## Troubleshooting

### YOLO: camera fails with `creat_vse_node failed, ret -10`

**Symptom:** The YOLO example loaded the model fine, but `mipi_cam` died with `creat_vse_node(1176) failed, ret -10`, then `capture init failture`. The codec and websocket nodes then reported no image or AI data.
**Cause (likely):** The VSE (video scaling engine) couldn't produce the requested output sizes from the 1280×1088 sensor. The camera node also produces a 1920×1080 output by default, which is larger than the sensor.
**Fix:** Request a smaller resolution:

```bash
dnn_example_image_width:=640 dnn_example_image_height:=480
```

### Camera: `capture init failure`

**Symptom:** `mipi_cam` failed with `[init]->cap capture init failture`, followed by errors from the codec and websocket nodes saying they weren't receiving images.
**What I checked:** Setting the correct resolution for the SC132GS sensors (1280×1088) didn't help. System image (3.5.0) and `mipi-cam` package (2.5.2) versions were new enough, which pointed to hardware.
**Fix:** The ribbon cables were flipped at the camera end. On the GS130W, the gold contacts face **away** from the camera's circuit board (the opposite of the GS130WI). After flipping both cables, the log showed `cap sc132gs-1280p init success`.

### USB-C connection times out in RDK Studio

**Symptom:** RDK Studio's USB-C (Type-C direct) connection to `192.168.128.10` timed out, and its prompt to set up the Mac's network adapter didn't complete.
**Fix:** Set the Mac's USB network adapter IP manually, then SSH directly:

```bash
# On the Mac (adapter name may differ; RDK Studio's connect dialog shows it)
sudo ifconfig en7 192.168.128.100 netmask 255.255.255.0
ping 192.168.128.10
ssh root@192.168.128.10
```

This resets when the cable is unplugged or the Mac restarts, so rerun the `ifconfig` command after reconnecting.

### Board keeps dropping off Wi-Fi

**Symptom:** The board went offline repeatedly and SSH sessions timed out.
**What I checked:** Turning off Wi-Fi power saving didn't fix it. The signal strength where the board was sitting was weak (around 52). `nmcli device wifi list` shows signal strength in the `SIGNAL` column.
**Fix:** Move the board closer to the router, or use a stronger network. A phone hotspot right next to the board also works.

---

## Repository layout

```
.
├── README.md
├── docs/
│   └── camera-setup.md   # Stereo camera bring-up notes
└── stage1/
    ├── evidence/         # Screenshots and clips (Wi-Fi details, IPs, and serials redacted)
    └── scripts/          # Scripts used for the camera and AI task
```

## Notes

No Wi-Fi passwords, tokens, private keys, or device serials are published in this repository.