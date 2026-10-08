# Elan 04f3:0c6e Fingerprint Reader Fix for Linux

This repository provides a working `libfprint` driver (`elanpress`) for the **Elan 04f3:0c6e** fingerprint sensor, commonly found in various ASUS laptops (such as the Zenbook 14, ROG Flow X13, and others).

---

## The Problem

Upstream `libfprint` misclassifies the `04f3:0c6e` sensor under the standard `elan` swipe driver:

1. **Swipe vs. Touch Misclassification**: The sensor is physically a touch/press pad, but upstream hardcodes the driver class to `FP_SCAN_TYPE_SWIPE`. As a result, the desktop environment and `fprintd` expect continuous swiping gestures, leading to `"swipe too short"` errors.
2. **Hardware Wedging / Protocol Error**: When polling the hardware presence byte (`pre_scan_cmd`), the sensor unblocks once and then wedges at "finger present" until a power cycle or times out, returning unexpected status bytes (`0x00`, `0xaf`). The standard driver treats this as a fatal `FP_DEVICE_ERROR_PROTO`, aborting the session and reporting "device disconnected".
3. **Matcher Incompatibility**: Upstream `libfprint` uses the NIST NBIS Bozorth3 minutiae matcher. Bozorth3 expects large swiped impressions with many minutiae points (threshold 24). The compact touch pad yields too few minutiae for Bozorth3, causing `"verify-no-match"` even when enrollment succeeds.

---

## The Solution

This fork integrates the dedicated **`elanpress`** driver:

- **Native Press/Touch Support**: Configured natively as a press sensor (`FP_SCAN_TYPE_PRESS`).
- **Image-Based Presence Detection**: Bypasses the unreliable hardware status byte by inferring finger presence directly from sensor image data.
- **SIFT Keypoint Matching**: Replaces Bozorth3 with a robust SIFT (Scale-Invariant Feature Transform) keypoint matcher designed specifically for compact touch pads.
- **Multi-Stage Enrollment**: Captures 12 presses during enrollment to construct a complete composite representation of the finger pad.

---

## How to Build & Install

### 1. Install Build Dependencies

- **Fedora**:
  ```bash
  sudo dnf install meson ninja gcc git libgusb-devel pixman-devel openssl-devel systemd-devel glib2-devel
  ```
- **Ubuntu / Debian**:
  ```bash
  sudo apt install meson ninja-build gcc git libgusb-dev libpixman-1-dev libssl-dev libsystemd-dev libglib2.0-dev
  ```
- **Arch Linux**:
  Available via the AUR as [`libfprint-elanpress-git`](https://aur.archlinux.org/packages/libfprint-elanpress-git).

### 2. Build the Library

```bash
git clone -b elan-0c6e-touch-support https://github.com/FCsab/Elan-04f3-0c6e-fingerprint-reader-fix.git
cd Elan-04f3-0c6e-fingerprint-reader-fix
meson setup build --prefix=/usr -Ddoc=false -Dintrospection=false
ninja -C build
```

### 3. Install & Restart Service

> [!IMPORTANT]
> **Keep backups outside of `/usr/lib64/`:** Do **not** store backup copies (such as `.stock` or `.bak`) directly in `/usr/lib64/`. Because shared libraries contain an internal SONAME (`libfprint-2.so.2`), `ldconfig` automatically scans all files in `/usr/lib64/` and will repoint the active symlink back to the stock library on the next system update! Always store backups in `/var/backups/libfprint/`.

```bash
# 1. Safely backup stock library outside the library linker search path
sudo mkdir -p /var/backups/libfprint
sudo cp /usr/lib64/libfprint-2.so.2.0.0 /var/backups/libfprint/ 2>/dev/null || \
sudo cp /usr/lib/x86_64-linux-gnu/libfprint-2.so.2.0.0 /var/backups/libfprint/ 2>/dev/null

# 2. Install compiled library
sudo ninja -C build install

# 3. Update linker cache and restart fprintd
sudo ldconfig
sudo systemctl restart fprintd
```

---

## Enrolling & Testing

1. **Delete any old prints:**
   ```bash
   fprintd-delete $USER
   ```

2. **Enroll your fingerprint:**
   ```bash
   fprintd-enroll
   ```
   > [!TIP]
   > **Enrollment Tips & Responsiveness:**
   > - **Lift your finger completely between presses for about 1 second.** The driver requires 10 consecutive empty polls to ensure you have lifted your finger so the same press isn't sampled twice. Tapping too quickly before the sensor registers a full release will ignore the press.
   > - **Press firmly.** Light or partial touches with low contrast are automatically filtered out to ensure template quality, requiring an extra press.
   > - **Vary your position slightly** (center, left side, right side, fingertip) across the 12 stages to allow SIFT to map the entire finger pad.

3. **Verify:**
   ```bash
   fprintd-verify
   ```
   Touch the sensor once to confirm recognition.

4. **Lock Screen:**
   Lock your screen (`Super + L`) and unlock using your fingerprint sensor.

---

## Troubleshooting & Surviving System Updates

### If an update reverts the fix or `fprintd` reports a protocol error:
1. **Check where the symlink points:**
   ```bash
   ls -l /usr/lib64/libfprint-2.so.2
   ```
   If it points to a `.stock` or `.bak` file, move the backup file out of `/usr/lib64/`:
   ```bash
   sudo mkdir -p /var/backups/libfprint
   sudo mv /usr/lib64/libfprint-2.so.2.0.0.* /var/backups/libfprint/ 2>/dev/null || true
   sudo ldconfig
   sudo systemctl restart fprintd
   ```

2. **If a Fedora update overwrites `libfprint-2.so.2.0.0` with upstream stock:**
   Simply re-install from your build directory:
   ```bash
   cd ~/git/libfprint  # or wherever your repo is cloned
   sudo ninja -C build install
   sudo ldconfig
   sudo systemctl restart fprintd
   ```

---

## AI Disclosure

In the spirit of transparency, **AI tools were used in the development of this repository**:
- Codebase research, driver porting/integration onto modern `libfprint`, meson build configurations, protocol debugging, and documentation were performed with the assistance of **Google DeepMind's Antigravity AI coding assistant**.
- All changes were compiled, inspected, and tested on actual hardware (`04f3:0c6e` on ASUS laptop) before committing.

---

## Credits & Acknowledgements

- Dedicated driver logic and SIFT matcher based on the work by [Filip Spanne](https://github.com/filip-rs/libfprint) (`elanpress`).
- Upstream driver framework provided by the [libfprint](https://gitlab.freedesktop.org/libfprint/libfprint) project.
