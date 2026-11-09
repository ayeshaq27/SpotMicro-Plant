# Stereo Camera Bring-Up: RDK X5 + GS130W

Notes from getting the D-Robotics RDK Stereo Camera GS130W (dual SC132GS global-shutter sensors) streaming on the RDK X5, including what went wrong and how it was fixed.

**Setup:** RDK X5, system image 3.5.0, `tros-humble-mipi-cam` 2.5.2, GS130W connected with two 22-pin FFC cables, viewed from a Mac over Wi-Fi.

---

## 1. Hardware connection

The GS130W uses **two** ribbon cables, one per camera, so both of the X5's MIPI camera ports (CAM1 and CAM2) are used.

- **Power the board off before connecting or disconnecting the cables.**
- If you have the black cables, they are directional: the end marked **CAM** goes to the camera, and the end marked **RDK_X5** goes to the board.
- **Board side:** insert the cable vertically, with the contacts facing the **Ethernet port**, then close the latch.
- **Camera side:** the cables go into the **two middle connectors** on the back of the camera. The two outer connectors hold the camera's own internal sensor cables. On the GS130W, the gold contacts face **away from the camera's circuit board**. This is the opposite of the GS130WI variant, so don't follow GS130WI instructions.
- Push each cable in fully and evenly, so no gold contacts show above the connector, before closing the latch.

> ⚠️ **Lesson learned:** my first attempt had the camera-side cables flipped (contacts facing the camera's PCB). Software gave no hint it was a cable problem; it just failed with `capture init failure`. Flipping both cables fixed it.

---

## 2. Load the ROS 2 environment

```bash
source /opt/tros/humble/setup.bash
```

TogetheROS (TROS) is D-Robotics' distribution of ROS 2 Humble, installed in `/opt/tros/humble`. `source` loads its environment into the current terminal so `ros2` and the D-Robotics packages can be found. It only applies to that terminal session, so run it first in every new terminal.

---

## 3. Test the camera alone

```bash
ros2 launch mipi_cam mipi_cam_dual_channel.launch.py mipi_image_width:=1280 mipi_image_height:=1088 mipi_lpwm_enable:=True
```

This starts only the camera node, which keeps the log clean for debugging.

**Success looks like:**

```
[mipi_cam]: [init]->cap sc132gs-1280p init success.
```

**Failure looks like:**

```
[mipi_cam]: [init]->cap capture init failture.
[mipi_node]: [init]->mipinode init failure.
```

If it fails, check the cables (section 1) before anything else.

Press `Ctrl+C` to stop.

---

## 4. View the live feed in a browser

```bash
ros2 launch mipi_cam mipi_cam_dual_channel_websocket.launch.py mipi_image_width:=1280 mipi_image_height:=1088 mipi_lpwm_enable:=True
```

Then, on a computer on the same network, open:

```
http://<board-ip>:8000
```

and click **Web Display**. Both camera images appear stacked vertically.

### What this launch file starts

`ros2 launch` runs a launch file that starts several ROS 2 nodes together. This one starts three, forming a pipeline:

| Node | Subscribes to | Publishes | Job |
|---|---|---|---|
| `mipi_cam` | (camera hardware) | `/image_combine_raw` | Captures from both cameras over MIPI, stacks the images, publishes raw NV12 frames |
| `hobot_codec_republish` | `/image_combine_raw` | `/image_combine_jpeg` | Compresses frames to JPEG using the board's hardware encoder |
| `websocket` | `/image_combine_jpeg` | (web page on port 8000) | Serves the stream to the browser |

Because it's a chain, one failure produces errors from all three nodes. When `mipi_cam` couldn't capture, the codec reported it wasn't receiving images, and the websocket reported it wasn't receiving JPEGs. **Only the first error in the chain is the real problem.**

---

## 5. Parameters

Arguments are passed to a launch file with `name:=value`.

| Parameter | Value | Why |
|---|---|---|
| `mipi_image_width` | `1280` | Matches the SC132GS sensor resolution |
| `mipi_image_height` | `1088` | The launch file's defaults are set up for a different camera (SC230AI at 1280×640) |
| `mipi_lpwm_enable` | `True` | Enables a sync signal so both sensors expose at the same moment, which matters for stereo depth. D-Robotics' examples for this camera turn it on. |

Note: these are the correct settings for this camera, but the cable flip was the actual fix. The launch still failed with these parameters until the cables were corrected.

---

## 6. Diagnostic commands

Used to rule out software before taking the hardware apart:

```bash
cat /etc/version
apt list --installed 2>/dev/null | grep mipi-cam
```

- `cat /etc/version` prints the board's system image version (mine: `3.5.0`).
- `apt list --installed | grep mipi-cam` shows the installed camera package version (mine: `2.5.2`).
- `2>/dev/null` hides a warning `apt` prints when its output is piped into another command.

D-Robotics' stereo camera docs list system image 3.4.1+ and mipi-cam 2.5.0+ as requirements, so both were new enough. That pointed to hardware.

---

## Known quirks

- The feed appears rotated sideways because of how the camera was sitting during testing. This will be addressed once the camera is mounted on the robot.

## References

- [GS130W installation guide](https://developer.d-robotics.cc/accessories_doc/en/stereo_camera_gs130w/installation)
- [GS130W quick start](https://developer.d-robotics.cc/accessories_doc/en/stereo_camera_gs130w/quick_start)
- [hobot_mipi_cam on GitHub](https://github.com/D-Robotics/hobot_mipi_cam)
