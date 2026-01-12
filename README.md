# Fresco Logic FL6000 Keyboard Driver for Linux

This is a modified driver for the Fresco Logic FL6000 USB 3.0 Hub Controller, specifically adapted for the **Acer Aspire Switch 12 S** (and similar devices) detachable keyboard dock.

The FL6000 chip acts as a USB bridge between the tablet and the detachable keyboard, exposing keyboard and touchpad as USB HID devices through a virtual xHCI host controller.

## Supported Devices

- Acer Aspire Switch 12 S (SW7-272)
- Other devices using the Fresco Logic FL6000 chip (VID: 0x1D5C, PID: 0x6000)

## Requirements

- Linux kernel 6.x (tested on 6.14)
- Kernel headers installed
- Build tools (gcc, make)

### Install build dependencies (Debian/Ubuntu):

```bash
sudo apt update
sudo apt install build-essential linux-headers-$(uname -r)
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

### Step 4: Install the Driver (Permanent)

To make the driver load automatically on boot:

```bash
# Copy the module to the kernel modules directory
sudo cp src/ehub.ko /lib/modules/$(uname -r)/kernel/drivers/usb/host/

# Update module dependencies
sudo depmod -a

# Load the module
sudo modprobe ehub
```

To load the driver automatically at boot, create a modules-load config:
```bash
echo "ehub" | sudo tee /etc/modules-load.d/ehub.conf
```

### Step 5: Blacklist Conflicting Module (if needed)

If the system has a non-working ehub module, blacklist it:

```bash
echo "blacklist ehub" | sudo tee /etc/modprobe.d/blacklist-ehub.conf
```

Then install your working module as shown above.

## Uninstallation

```bash
sudo rmmod ehub
sudo rm /lib/modules/$(uname -r)/kernel/drivers/usb/host/ehub.ko
sudo rm /etc/modules-load.d/ehub.conf
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

This driver is based on the original Fresco Logic FL6000 driver and is licensed under GPL v2.

## Credits

- Original driver: Fresco Logic, Inc. (2014-2017)
- Kernel 6.x adaptation and fixes: Community contribution
