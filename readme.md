# Adding support for the A7CMII Camera to the ASIAIR MINI

## Introduction

The lack of support for the Sony A7C II on ASIAIR is not due to a limitation of the device itself.

ASIAIR relies on the libgphoto2 library for camera communication, and at the time of this setup, the specific camera model (ILCE-7CM2) was not properly recognized in the version included in the system.

This means the issue is related to missing or incomplete support in the underlying library, rather than a deliberate restriction or hardware incompatibility from ASIAIR.

By patching and rebuilding libgphoto2, full functionality can be achieved.

## ⚠️ SSH Access Requirement

Before starting, you need SSH access to the ASIAIR device.

This can be achieved using the jailbreak script from:

https://github.com/open-astro/rk-flashtool/tree/master/jailbreak

If the jailbreak does not work on your device:

1. Reset the ASIAIR firmware to the original factory version
2. Run the jailbreak process again

Once completed, you should be able to connect via:

## On the ASIAIR, search for the camera in the system (7CM2)
```bash
lsusb
```

Bus 002 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub Bus 001 Device 043: ID 054c:0e8c Sony Corp. 

Bus 001 Device 045: ID 054c:094e Sony Corp. ILCE-6000 (aka Alpha-6000) in PC Remote mode Bus 001 Device 002: ID 04b4:6572 Cypress Semiconductor Corp. 

Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

054c:0e8c Sony Corp. --> This is my A7CMII Camera (THIS NUMBERS ARE IMPORTANT TO IDENTIFY THE CAMERA)

054c:094e Sony Corp. ILCE-6000 in PC Remote mode --> This is my A6000 Camera

## Understanding the Camera Entry (Vendor/Product IDs) - We need these numbers to identify the camera in the camera library
Vendor: 054c
Product: 0e8c

## Search for the camera in the system (ILCE) in the camera library
```bash
strings /usr/local/lib/libgphoto2/2.5.31/ptp2.so | grep -i ILCE
```

ILCE-7RM4 ILCE-7RM4A ILCE-7RM5 ILCE-7M3 ILCE-7SM3 ILCE-7M4 ILCE-9M2 ILCE-1 ILCE-7C Sony:ILCE-7R M2 (MTP) Sony:ILCE-7M2 (Control) Sony:ILCE-6400 (PC Control) Sony:ILCE-1 (Control) Sony:ILCE-7C (Control) Sony:ILCE-7RM3A (PC Control) Sony:ILCE-7RM4A (PC Control) Sony:ILCE-7RM5 (PC Control)

No ILCE-7CM2, ILCE-7C II, 054c:0e8c in the camera library

## List supported cameras in the camera library
```bash
gphoto2 --list-cameras | grep -i 7C
```
Sony ILCE-7C (Control)  --> Not A7CMII

The camera is not recognized in this build of libgphoto2,
even though it may be supported in newer upstream versions.

## Fetch the camera library in your local machine
```bash
curl -LO https://downloads.sourceforge.net/project/gphoto/libgphoto2/2.5.31/libgphoto2-2.5.31.tar.bz2
```

## Copy to the ASIAIR
```bash
scp libgphoto2-2.5.31.tar.* pi@10.0.0.1:/home/pi/
```

## In the ASIAIR, extract the library
```bash
tar -xvf libgphoto2-2.5.31.tar.bz2
```

## Go to the directory
```bash
cd libgphoto2-2.5.31
```

## Apply the patch to in order to support the A7CMII Camera (Add new entry for the A7CMII Camera)
```bash
sed -i '/ILCE-7C (Control)/a\ \ \ \ {"Sony:ILCE-7CM2 (Control)", 0x054c, 0x0e8c, PTP_CAP|PTP_CAP_PREVIEW},' camlibs/ptp2/library.c
```

## Delete cache
```bash
rm -rf autom4te.cache
find . -name "*.lo" -delete
find . -name "*.o" -delete
```

## Compile the library
```bash
./autogen.sh
./configure --prefix=/usr/local
make -j4
```

## Mount filesystem (write mode)
```bash
sudo mount -o remount,rw /
```

## Backup the original library
```bash
cp /usr/local/lib/libgphoto2/2.5.31/ptp2.so \
   /usr/local/lib/libgphoto2/2.5.31/ptp2.so.bak
```

## Install patched version
```bash
cp camlibs/ptp2/.libs/ptp2.so \
   /usr/local/lib/libgphoto2/2.5.31/ptp2.so
```

## Fix USB permissions
```bash
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="054c", ATTR{idProduct}=="0e8c", MODE="0666"' \
| sudo tee /etc/udev/rules.d/99-sony-a7cii.rules > /dev/null
```

## 3 Reload udev
```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

## Restore read-only
```bash
sync
sudo mount -o remount,ro /
```

## Reboot the ASIAIR and test the camera

## Commands to test the camera in the ASIAIR
```bash
gphoto2 --auto-detect
gphoto2 --summary
gphoto2 --capture-image
gphoto2 --capture-image-and-download
```

## In the ASIAIR App

The exposure time is limited to 30 seconds. I'll try to unlock that later. In the meantime I think we can use a shutter release cable.

**Update 1**: The 30 sec limit is only for the preview mode. In Autorun mode you can set as much as you want.

<img width="1200" height="872" alt="WhatsApp Image 2026-05-04 at 19 40 16" src="https://github.com/user-attachments/assets/a13a6e9d-682d-4742-a6e7-a2a862b2b1cd" />


