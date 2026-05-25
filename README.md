# GPSD – Home Assistant Add-on

Run [gpsd](https://gpsd.io/) inside a Home Assistant OS container, exposing GPS data over TCP port **2947** for any client on your LAN.

## Supported Hardware

Any USB GPS receiver that presents a serial port. Common chipsets:

- **u-blox** (e.g. VK-162, BN-220)
- **SiRF Star** (e.g. GlobalSat BU-353)
- **MediaTek** (e.g. Adafruit Ultimate GPS)
- **Silicon Labs CP2102N** USB-to-UART bridges

The device typically appears as `/dev/ttyUSB0` or `/dev/ttyACM0`.

## Installation

1. In Home Assistant, go to **Settings → Add-ons → Add-on Store**.
2. Click the **⋮** menu (top-right) → **Repositories**.
3. Add the repository URL:
   ```
   https://github.com/leo-stan/ha-addon-gpsd
   ```
4. Find **GPSD** in the store and click **Install**.
5. Configure the add-on (see below) and click **Start**.

## Configuration

| Option         | Default         | Description                                 |
|----------------|-----------------|---------------------------------------------|
| `device`       | `/dev/serial/by-id/usb-...` | Path to the serial GPS device (use `/dev/serial/by-id/` for stability) |
| `baud`         | `9600`          | Baud rate for the serial connection          |
| `gpsd_options` | `"-n"`          | Extra gpsd flags (e.g. `-n` to poll on open)|

### Finding Your Device Path

**Option 1 — HA UI (easiest):**
1. Go to **Settings → System → Hardware**
2. Click **All Hardware** (bottom of the page)
3. Search for your GPS device (e.g. `CP2102`, `ttyUSB`, or `ttyACM`)
4. Copy the `/dev/serial/by-id/...` path

**Option 2 — Terminal / SSH:**
```bash
ls -l /dev/serial/by-id/
```

Always prefer the `/dev/serial/by-id/...` path — it is stable across reboots and doesn't change if you plug in other USB devices.

### Example Configuration

```yaml
device: /dev/serial/by-id/usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_2c109c3eadf4ef118da5c41b6d9880ab-if00-port0
baud: 9600
gpsd_options: "-n"
```

## Testing

From another machine on the same network (install clients first: **Debian/Ubuntu** `sudo apt install gpsd-clients`, **macOS** `brew install gpsd`, **Fedora** `sudo dnf install gpsd-clients`):

```bash
# Interactive GPS monitor
cgps -s <HA-IP>:2947

# Raw NMEA/JSON stream (5 sentences)
gpspipe -w -n 5 <HA-IP>:2947
```

Replace `<HA-IP>` with the **host only** — the same address you use in the browser to open Home Assistant (no `http://`, no path). For example, if the UI is at `http://192.168.1.2:8123/...`, use `192.168.1.2`. That is also the value to use as **Host** in the GPSd integration below.

## Troubleshooting

### "GPS device not found"
- Verify the device is plugged in and detected: `ls /dev/ttyUSB* /dev/ttyACM*`
- Try the `/dev/serial/by-id/` path instead
- Check the add-on configuration for typos in the device path

### "Operation not permitted" or "Permission denied" on the serial device
- The Supervisor only passes through host devices listed in the add-on manifest (`devices:` in `gpsd/config.yaml`). This repository includes **`/dev/ttyUSB0`** and **`/dev/ttyACM0`** (most u-blox modules enumerate as `ttyACM0`).
- If your receiver is on **`ttyACM0`**, set the add-on **device** option to **`/dev/ttyACM0`**, or use your `/dev/serial/by-id/...` path if it resolves correctly inside the container.
- If you still see permission errors, confirm you are on the latest add-on build (rebuild or reinstall after a repo update) so the new `devices:` list is applied.

### No GPS fix
- Make sure the antenna has a clear view of the sky
- Some receivers need a cold-start time of 30–60 seconds
- Check baud rate matches your receiver (most default to 9600)
- Look at the add-on logs for gpsd output

## Using with the Home Assistant GPSd Integration

Once the add-on is running, you can connect it to the built-in [GPSd integration](https://www.home-assistant.io/integrations/gpsd/) to get GPS entities in Home Assistant (latitude, longitude, fix mode, etc.).

1. Go to **Settings → Devices & Services → Add Integration**
2. Search for **GPSd** and select it
3. Enter the following:
   - **Host**: your Home Assistant machine's **LAN IP address** (e.g. `192.168.1.50`), **not** `127.0.0.1`. Home Assistant Core runs in its own container; `localhost` there is not the gpsd add-on. On Home Assistant OS / Supervised, use the same IP you use in the browser to open HA. ([Community note](https://community.home-assistant.io/t/gpsd-custom-add-on-not-seen-by-home-assistant-in-hassos/561110))
   - **Port**: `2947`
4. Click **Submit**

If entities stay empty or **mode** stays **Unknown**, wait for a satellite fix near a window or outside (often 30–60+ seconds). **Unknown** is normal until gpsd reports a fix.

The integration will create a `sensor.gpsd` entity with attributes including:
- **Latitude / Longitude**
- **Altitude**
- **Speed**
- **Fix mode** (no fix, 2D, 3D)

You can use these in automations, device trackers, or dashboards.

## License

MIT – see [LICENSE](LICENSE).
