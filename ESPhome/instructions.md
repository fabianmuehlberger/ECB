# ESPHome local development (Python venv)

Use a virtual environment in this directory so ESPHome and PlatformIO stay isolated from system Python.

## Prerequisites

- **Python 3.11+** (3.14 on Linux is supported by current ESPHome releases).

## One-time setup

From this directory (`ESPhome/`):

```bash
python -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install esphome
```

Ignore the venv in git: add `/.venv/` to `ESPhome/.gitignore` if it is not already listed there.

## Daily use

```bash
cd /path/to/ECB/ESPhome
source .venv/bin/activate
```

Pick the YAML you want (examples: `fanctrl.yaml`, `ecb1.yaml`, …).

| Goal | Command |
|------|---------|
| Validate config | `esphome config fanctrl.yaml` |
| Compile only | `esphome compile fanctrl.yaml` |
| Compile and flash (USB or OTA) | `esphome run fanctrl.yaml` |
| Flash to a specific port | `esphome run fanctrl.yaml --device /dev/ttyACM0` |
| Serial logs | `esphome logs fanctrl.yaml` |
| Web dashboard | `esphome dashboard .` |

Build output and caches live under `.esphome/` in this folder (PlatformIO, SDK, etc.).

## USB flashing on Arch Linux

### User permissions (serial port access)

Serial adapters usually appear as `/dev/ttyUSB0` (USB-UART bridges) or `/dev/ttyACM0` (CDC ACM, common on ESP32 dev boards).

On Arch Linux, add your user to the **`uucp`** group (Debian/Ubuntu docs often say `dialout`; on Arch the usual group for serial devices is `uucp`):

```bash
sudo usermod -aG uucp "$USER"
```

Log out and log back in (or reboot). Confirm:

```bash
groups
```

You should see `uucp` in the list.

**If ESPHome says to add the `dialout` group:** ignore that on Arch — `dialout` often does not exist. Use `uucp` as above.

**Important:** `groups` / `id` print what your account *can* belong to from `/etc/group`. Processes keep the group list they had at **login time**. After `usermod -aG uucp`, you must **end your graphical session** (log out of Plasma/GNOME/etc. and log back in) or **reboot**. Opening a new terminal tab is often *not* enough if it inherits the old session.

Check whether *this shell* actually has the `uucp` GID (example GID `985` — yours may differ):

```bash
getent group uucp
grep '^Groups:' /proc/self/status
```

The number after `Groups:` must include the GID from `getent group uucp`. If it does not, log out fully or run one of:

```bash
newgrp uucp    # use this terminal for esphome after this
# or one-shot:
sg uucp -c 'cd /path/to/ECB/ESPhome && . .venv/bin/activate && esphome run fanctrl.yaml --device /dev/ttyUSB0'
```

**Still permission denied after a full re-login:** inspect the device node (board plugged in):

```bash
stat -c '%A %U %G %n' /dev/ttyUSB0
```

You want group `uucp` and group read/write (e.g. `crw-rw---- root uucp`). If the group is not `uucp`, unplug/replug after `sudo udevadm trigger`, or add a custom udev rule. If **ModemManager** grabs the port, try `sudo systemctl stop ModemManager` and flash again.

If a device still looks inaccessible, check ownership:

```bash
ls -l /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
```

### Drivers / kernel modules

Most USB-serial chips use in-tree modules (`ch341`, `cp210x`, `ftdi_sio`, `cdc_acm`). After plugging in the board:

```bash
lsusb
dmesg | tail -30
```

If no `ttyUSB` or `ttyACM` node appears, check `dmesg` for missing firmware or unsupported hardware.

### Conflicting software

Only one process can use a serial port at a time. Close other serial monitors (`minicom`, another `esphome logs`, IDE serial terminals) before flashing.

### Flash from the venv

With the board connected:

```bash
source .venv/bin/activate
esphome run fanctrl.yaml --device /dev/ttyACM0
```

Use the device path that exists on your machine (`ttyACM0` vs `ttyUSB0`).

### Optional: udev rule (stable symlink)

For a fixed device name when several USB serial devices are plugged in, add a rule under `/etc/udev/rules.d/` (for example `99-esphome-serial.rules`). Use `SUBSYSTEM=="tty"`, `ATTRS{idVendor}`, `ATTRS{idProduct}`, and `ATTRS{serial}` from:

```bash
udevadm info -a -n /dev/ttyACM0
```

Then reload:

```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

## Pinning ESPHome version (optional)

For reproducible builds:

```bash
pip install 'esphome==2024.11.0'
```

Match the `min_version` in your YAML where relevant.

## Deactivate

```bash
deactivate
```
