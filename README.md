# Fresco Logic FL6000 Keyboard Driver for Linux

This is a modified driver for the Fresco Logic FL6000 USB 3.0 Hub Controller, specifically adapted for the **Acer Aspire Switch 12 S** (and similar devices) detachable keyboard dock.

The FL6000 chip acts as a USB bridge between the tablet and the detachable keyboard, exposing keyboard and touchpad as USB HID devices through a virtual xHCI host controller.

## Supported Devices

- Acer Aspire Switch 12 S (SW7-272)
- Other devices using the Fresco Logic FL6000 chip (VID: 0x1D5C, PID: 0x6000)

## Requirements

- Linux kernel 6.x (tested on 6.17)
- Kernel headers installed
- Build tools (gcc, make)

### Install build dependencies (Debian/Ubuntu):

```bash
sudo apt update
sudo apt install build-essential linux-headers-$(uname -r) dkms
```

## Installation

### Step 1: Configure USB Quirks

Add USB quirks to GRUB to ensure proper USB communication with the FL6000 device.

Edit `/etc/default/grub` and add the following to `GRUB_CMDLINE_LINUX_DEFAULT`:

```
usbcore.quirks=1d5c:5050:k,1d5c:6000:kgn
```

For example, your line might look like:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash usbcore.quirks=1d5c:5050:k,1d5c:6000:kgn"
```

Then update GRUB and reboot:
```bash
sudo update-grub
sudo reboot
```

### Step 2: Build the Driver

```bash
cd src
make
```

### Step 3: Load the Driver (Manual)

```bash
sudo insmod ehub.ko
```

To verify the driver loaded correctly:
```bash
dmesg | tail -30
```

You should see messages about USB bus 5 being registered and ITE keyboard/touchpad devices being detected.

### Step 4: Install the Driver (DKMS - Recommended)

DKMS (Dynamic Kernel Module Support) automatically rebuilds the driver when you upgrade your kernel. This is the recommended installation method.

```bash
# Copy the driver source to /usr/src
sudo cp -r /path/to/fl6000-keyboard-driver /usr/src/ehub-1.0.0

# Register with DKMS
sudo dkms add -m ehub -v 1.0.0

# Build and install for current kernel
sudo dkms build -m ehub -v 1.0.0
sudo dkms install -m ehub -v 1.0.0

# Load the module
sudo modprobe ehub
```

To load the driver automatically at boot:
```bash
echo "ehub" | sudo tee /etc/modules-load.d/ehub.conf
```

Create a udev rule to ensure the driver loads when the keyboard dock is connected:
```bash
echo 'ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="1d5c", ATTR{idProduct}=="6000", RUN+="/sbin/modprobe ehub"' | sudo tee /etc/udev/rules.d/99-fl6000-keyboard.rules
sudo udevadm control --reload-rules
```

### Step 4 (Alternative): Manual Installation

If you prefer not to use DKMS, you can install manually. Note that you'll need to rebuild and reinstall after each kernel upgrade.

```bash
# Copy the module to the kernel modules directory
sudo cp src/ehub.ko /lib/modules/$(uname -r)/kernel/drivers/usb/host/

# Update module dependencies
sudo depmod -a

# Load the module
sudo modprobe ehub

# Auto-load at boot
echo "ehub" | sudo tee /etc/modules-load.d/ehub.conf

# Udev rule for device detection
echo 'ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="1d5c", ATTR{idProduct}=="6000", RUN+="/sbin/modprobe ehub"' | sudo tee /etc/udev/rules.d/99-fl6000-keyboard.rules
sudo udevadm control --reload-rules
```

### Step 5: Blacklist Conflicting Module (if needed)

If the system has a non-working ehub module, blacklist it:

```bash
echo "blacklist ehub" | sudo tee /etc/modprobe.d/blacklist-ehub.conf
```

Then install your working module as shown above.

## Uninstallation

### DKMS Uninstallation

```bash
sudo rmmod ehub
sudo dkms remove -m ehub -v 1.0.0 --all
sudo rm -rf /usr/src/ehub-1.0.0
sudo rm /etc/modules-load.d/ehub.conf
sudo rm /etc/udev/rules.d/99-fl6000-keyboard.rules
sudo udevadm control --reload-rules
```

### Manual Installation Uninstallation

```bash
sudo rmmod ehub
sudo rm /lib/modules/$(uname -r)/kernel/drivers/usb/host/ehub.ko
sudo rm /etc/modules-load.d/ehub.conf
sudo rm /etc/udev/rules.d/99-fl6000-keyboard.rules
sudo udevadm control --reload-rules
sudo depmod -a
```

## Troubleshooting

### USB Protocol Errors (-71 EPROTO)

If you see protocol errors in dmesg, ensure USB quirks are properly configured:
```bash
cat /proc/cmdline | grep quirks
```

Should show: `usbcore.quirks=1d5c:5050:k,1d5c:6000:kgn`

### Module Won't Load

Check for kernel version compatibility:
```bash
uname -r
modinfo src/ehub.ko | grep vermagic
```

If versions don't match, rebuild the driver.

### DMA Errors

If you see DMA-related warnings, ensure the driver is the patched version with proper sysdev support for DMA operations.

### Keyboard Keys Output Numbers Instead of Letters (NumLock Issue)

The ITE keyboard (VID:06CB PID:7BA3) connected via the FL6000 defaults to NumLock ON. This causes certain keys to output numbers instead of their normal characters (e.g., U=4, I=5, O=6, J=1, K=2, L=3).

**Quick fix** (temporary, until reboot):
```bash
# Find and disable NumLock on the ITE keyboard
for led in /sys/class/leds/input*::numlock; do
    input_num=$(basename "$led" | cut -d: -f1)
    input_name=$(cat "/sys/class/input/$input_num/name" 2>/dev/null)
    if echo "$input_name" | grep -qi "ITE"; then
        echo 0 | sudo tee "$led/brightness" > /dev/null
    fi
done
```

**Permanent fix** (auto-disable NumLock on keyboard connect):

1. Create the script `/usr/local/bin/fl6000-numlock-off.sh`:
```bash
sudo tee /usr/local/bin/fl6000-numlock-off.sh > /dev/null << 'EOF'
#!/bin/bash
sleep 1
for led in /sys/class/leds/input*::numlock; do
    if [ -f "$led/brightness" ]; then
        input_num=$(basename "$led" | cut -d: -f1)
        input_name=$(cat "/sys/class/input/$input_num/name" 2>/dev/null)
        if echo "$input_name" | grep -qi "ITE"; then
            echo 0 > "$led/brightness" 2>/dev/null
        fi
    fi
done
EOF
sudo chmod +x /usr/local/bin/fl6000-numlock-off.sh
```

2. Create a udev rule `/etc/udev/rules.d/99-fl6000-numlock.rules`:
```bash
echo 'ACTION=="add", SUBSYSTEM=="input", ATTRS{idVendor}=="06cb", ATTRS{idProduct}=="7ba3", RUN+="/usr/local/bin/fl6000-numlock-off.sh"' | sudo tee /etc/udev/rules.d/99-fl6000-numlock.rules
sudo udevadm control --reload-rules
```

## Technical Details

### Changes from Original Driver

This driver has been modified for Linux kernel 6.x compatibility:

1. **Timer API**: Updated from `setup_timer()` to `timer_setup()` with new callback signatures
2. **DMA Operations**: Fixed to use `hcd->self.sysdev` instead of `hcd->self.controller` for proper DMA support on USB devices
3. **Kernel API Updates**:
   - `probe_kernel_read/write` → `copy_from_kernel_nofault/copy_to_kernel_nofault`
   - `asm/unaligned.h` → `linux/unaligned.h`
4. **Function Signatures**: Updated `address_device` to include `timeout_ms` parameter
5. **DMA Mapping**: Fixed URB DMA mapping to use proper device with DMA capabilities

### USB Quirks Explanation

- `1d5c:5050:k` - FL6000 Hub: Disable Link Power Management (LPM)
- `1d5c:6000:kgn` - FL6000 Device:
  - `k` = Disable LPM
  - `g` = Delay initialization
  - `n` = Delay control messages

## License

This driver is licensed under the GNU General Public License v2 (GPL-2.0).
See the [LICENSE](LICENSE) file for the full license text.

## Credits

- Original xHCI driver: Intel Corp. (2008), Author: Sarah Sharp
- FL6000 driver: Fresco Logic, Inc. (2014-2017)
- Linux kernel 6.x adaptation and DMA fixes: Philipp Thiele
