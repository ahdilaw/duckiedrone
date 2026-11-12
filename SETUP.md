# Duckiedrone Setup (flyingduckmen)

A simple guide to set up the Raspberry Pi on our DD24 Duckiedrone from scratch.

- **Drone name:** `flyingduckmen`
- **Distro:** `ente`
- **Login:** `duckie` / `quackquack`
- **Hardware:** Raspberry Pi 4, Mamba F405 flight controller (PX4 v1.15.4), 4S battery

> **Where to run each command:**
> 💻 **Laptop** = WSL Ubuntu terminal on your laptop
> 🦆 **Drone** = SSH session on the drone

---

## 0. One-time laptop setup (💻 Laptop)

You need Docker and the `dts` tool inside WSL Ubuntu.

```bash
sudo systemctl start docker
docker ps                      # should not error
pip3 install --user --upgrade duckietown-shell
dts --set-version ente
dts tok set                    # paste your Duckietown token
```

---

## 1. Flash the SD card (💻 Laptop)

**a) Plug in the SD card reader and attach it to WSL.** Run this in **Windows PowerShell as Admin**:

```powershell
usbipd list                     # find the card reader's BUSID
usbipd bind --busid <BUSID>
usbipd attach --wsl --busid <BUSID>
```

**b) Find the card in WSL.** Match it by size, e.g. a 64 GB card shows as about 58G:

```bash
lsblk
```

**c) Flash it:**

```bash
dts init_sd_card --hostname flyingduckmen --type duckiedrone --configuration DD24 \
  --wifi "flyingdm:<hotspot-password>,duckietown:quackquack" --country US
```

- When it asks for the device, type the **full path**, e.g. `/dev/sdf`, not just `sdf`.
- If the image is already downloaded, add `--workdir /mnt/e/duckietown-flash` to reuse it.

**d) When it says done:** eject the card, put it in the Pi, and power the Pi on.

> ⚡ Use **USB-C bench power** for the Pi until the drone's 5V rail has been checked with a multimeter (it should read 5.0–5.25 V).

---

## 2. Find the drone and SSH in (💻 Laptop)

The first boot takes about **5–10 minutes**. Then run:

```bash
ping flyingduckmen.local
```

If `.local` doesn't resolve, find the IP on your hotspot's device list, then:

```bash
echo "<DRONE_IP> flyingduckmen.local" | sudo tee -a /etc/hosts
```

SSH in:

```bash
ssh duckie@flyingduckmen.local
```

---

## 3. Stop the Wi-Fi hijacker before updating (🦆 Drone)

The `wifi-access-point` container steals `wlan0` and drops your connection. Keep it stopped:

```bash
nohup bash -c 'while true; do docker stop -t 0 wifi-access-point >/dev/null 2>&1; sleep 1; done' >/dev/null 2>&1 &
```

---

## 4. Install the drone software (💻 Laptop)

```bash
dts duckiebot update -t duckiedrone --distro=ente -f flyingduckmen
```

- When it asks for the hardware, pick `raspberry_pi_64`.
- Wait for **"Update completed!"**. It takes a while.

---

## 5. Post-update fix block (🦆 Drone)

This disables the Wi-Fi hijacker, fixes the rosbridge port clash with the dashboard, and checks the power:

```bash
pkill -f "docker stop -t 0 wifi-access-point" 2>/dev/null
docker update --restart=no wifi-access-point ros2-rosbridge-websocket
docker stop wifi-access-point ros2-rosbridge-websocket
docker run -d --name ros2-rosbridge-9002 --restart unless-stopped --network host \
  -e DT_SUPERUSER=1 -e ROS_DOMAIN_ID=42 -e DT_LAUNCHER=rosbridge -e WEBSOCKET_BRIDGE_PORT=9002 \
  -v /var/run/avahi-daemon/socket:/var/run/avahi-daemon/socket \
  duckietown/dt-ros2-interface:ente-arm64v8
docker restart dashboard
sleep 40
sudo vcgencmd get_throttled     # want: throttled=0x0
docker ps --format "table {{.Names}}\t{{.Status}}"
```

---

## 6. Check the flight controller settings (🦆 Drone)

These live on the flight controller, so they normally survive a Pi or SD card swap.

```bash
for p in CBRK_SUPPLY_CHK BAT1_N_CELLS BAT1_V_EMPTY SYS_HAS_MAG SYS_HAS_BARO; do echo -n "$p: "; docker exec ros2-mavros bash -c "source /opt/ros/*/setup.bash; ros2 param get /mavros/param $p"; done
```

| Param | Expected |
|---|---|
| CBRK_SUPPLY_CHK | 894281 |
| BAT1_N_CELLS | 4 |
| BAT1_V_EMPTY | 3.5 |
| SYS_HAS_MAG | 0 |
| SYS_HAS_BARO | 0 |

If any are wrong, set them in QGroundControl.

---

## 7. (Optional) PX4 event decoder on port 8099 (💻 Laptop)

```bash
cd /mnt/c/Users/ahmed/Downloads && unzip -o px4-event-decoder.zip
scp -r px4-event-decoder duckie@flyingduckmen.local:~/
ssh duckie@flyingduckmen.local "bash ~/px4-event-decoder/install.sh"
```

Open `http://<DRONE_IP>:8099`.

---

## 8. Arm test, props OFF (Browser)

1. Open `http://<DRONE_IP>` in its **own Chrome window**. A background tab drops the joystick stream.
2. **Sign in as Developer.** Signed-out users are read-only and can't arm.
3. With the drone flat, the IMU should show roll and pitch of about **0°**.
4. Set **Thrust Cap 40%**, turn on **Override**, choose mode **STABILIZED**, then **ARM**.
5. All 4 motors should spin. Give it throttle within about 10 s or PX4 auto-disarms.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| "Arming denied: Resolve system health failures" | Switch the mode to STABILIZED. AUTO.LOITER can't arm. |
| Low battery warning won't clear | Charge the battery and **reboot** the drone. The warning latches. |
| Manual control lost/regained repeatedly | Move the dashboard to its own visible window. |
| Dashboard shows 502 Bad Gateway | Run `docker restart dashboard`. If commands fail with I/O errors, the SD card is corrupted: reflash it. |
| `vcgencmd` error about `/dev/vcio` | Use `sudo vcgencmd`. |
| Flashing wrote to the wrong place | Re-run it and type the full path, e.g. `/dev/sdf`. |
| Pi LED stays solid green, even with no card | The Pi is dead or its EEPROM is corrupted. Swap the Pi. |
