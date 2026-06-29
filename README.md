# ubuntu_hdmi_not_working_solution
---

# Ubuntu HDMI Not Working (NVIDIA + Secure Boot)

## Problem

* HDMI cable connected.
* External monitor not detected or shows **No Signal**.
* Ubuntu only displays the laptop screen.
* Laptop: **Acer Nitro AN515-52**
* GPU:

  * Intel UHD 630
  * NVIDIA GTX 1060 Mobile
* Ubuntu 24.04

---

# Step 1: Check if HDMI is detected

```bash
xrandr
```

### Example Output (Problem)

```text
Screen 0: minimum 16 x 16, current 1920 x 1080, maximum 32767 x 32767

eDP-1 connected primary 1920x1080+0+0
```

Notice:

* ✅ `eDP-1` exists
* ❌ No `HDMI-1`
* ❌ No `HDMI-A-1`

This means Ubuntu cannot see the HDMI output.

---

# Step 2: Check DRM connectors

```bash
for f in /sys/class/drm/*/status; do
    echo "$f: $(cat "$f")"
done
```

### Example Output

```text
/sys/class/drm/card1-eDP-1/status: connected
```

Only the laptop display exists.

---

# Step 3: Check available display connectors

```bash
ls /sys/class/drm/
```

### Example Output

```text
card1
card1-eDP-1
renderD128
version
```

Problem:

No HDMI connector exists.

---

# Step 4: Check GPU drivers

```bash
lspci -k | grep -EA3 'VGA|3D|Display'
```

### Example Output

```text
00:02.0 VGA compatible controller:
Intel UHD Graphics 630

Kernel driver in use: i915

01:00.0 VGA compatible controller:
NVIDIA GeForce GTX 1060 Mobile

Kernel modules:
nvidiafb
nouveau
nvidia_drm
nvidia
```

Notice:

Intel has

```text
Kernel driver in use: i915
```

But NVIDIA does **NOT** have

```text
Kernel driver in use: nvidia
```

This means the NVIDIA driver is not loaded.

---

# Step 5: Verify NVIDIA

```bash
nvidia-smi
```

### Example Output

```text
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.
```

---

# Step 6: Check if the NVIDIA module is loaded

```bash
lsmod | grep nvidia
```

### Example Output

```text
(no output)
```

Meaning:

The NVIDIA kernel module is not loaded.

---

# Step 7: Try loading the driver

```bash
sudo modprobe nvidia
```

### Example Output

```text
modprobe: ERROR: could not insert 'nvidia':
Key was rejected by service
```

This is the key clue.

---

# Step 8: Check Secure Boot

```bash
mokutil --sb-state
```

### Example Output

```text
SecureBoot enabled
```

Root Cause:

Secure Boot is rejecting the NVIDIA kernel module.

---

# Step 9: Verify MOK files

```bash
sudo ls -l /var/lib/shim-signed/mok/
```

### Example Output

```text
MOK.der
MOK.priv
```

Ubuntu already generated signing keys.

---

# Step 10: Enroll the MOK

```bash
sudo mokutil --import /var/lib/shim-signed/mok/MOK.der
```

It asks for a password.

Example:

```text
input password:
confirm password:
```

Choose any password (8–16 characters).

---

# Step 11: Reboot

After reboot, a blue screen appears.

Choose:

```
Enroll MOK
```

↓

```
Continue
```

↓

```
Yes
```

↓

Enter the password

↓

```
Reboot
```

---

# Step 12: Verify NVIDIA

```bash
nvidia-smi
```

Expected Output

Instead of an error, you should now see something like:

```text
+------------------------------------------------------+
| NVIDIA-SMI 535.xx                                    |
| GPU Name: GeForce GTX 1060 Mobile                    |
+------------------------------------------------------+
```

---

# Step 13: Verify HDMI

```bash
xrandr
```

Expected Output

```text
eDP-1 connected

HDMI-1 connected
```

or

```text
HDMI-A-1 connected
```

Your monitor should now be detected.

---

# Root Cause Summary

Symptoms:

* HDMI not detected
* `xrandr` only shows `eDP-1`
* `nvidia-smi` fails
* `lsmod | grep nvidia` returns nothing
* `modprobe nvidia` returns **Key was rejected by service**
* `mokutil --sb-state` reports **SecureBoot enabled**

Cause:

* Secure Boot prevented the unsigned NVIDIA kernel module from loading.
* Because the HDMI port is wired to the NVIDIA GPU on the Acer Nitro AN515-52, the HDMI connector never became available.

Solution:

* Enroll the Machine Owner Key (MOK) using:

```bash
sudo mokutil --import /var/lib/shim-signed/mok/MOK.der
```

* Reboot.
* Enroll the key through the MOK Manager.
* The NVIDIA driver loads successfully, and HDMI works again.

---
