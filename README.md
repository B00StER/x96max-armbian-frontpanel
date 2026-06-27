# X96 Max front-panel display on ophub Armbian

Gets the 4-digit front-panel VFD/LED clock display working on an X96 Max (Amlogic
S905X2) running [ophub](https://github.com/ophub/amlogic-s9xxx-armbian) Armbian,
where it stays dark out of the box and `armbian-openvfd` doesn't help.

This repo is a thin recipe and does not contain the display driver. The driver is
[jefflessard/tm16xx-display](https://github.com/jefflessard/tm16xx-display)
(GPL-2.0, currently [in mainline review](https://lists.openwall.net/linux-kernel/2025/11/21/1304)).
What's here is the glue to build it on ophub kernels, package it as DKMS so it
survives kernel updates, and rebuild it automatically after `armbian-update`.

## Why the panel is dark

ophub's device tree already describes the front panel: an FD628/TM1628 controller
on a bit-banged SPI bus, with the device created at `/sys/bus/spi/devices/spi0.0`
(`compatible = "fdhisi,fd628","titanmec,tm1628"`). The ophub kernel ships no
driver that binds to it (no `CONFIG_TM16XX`, no module), so the device just sits
there with nothing driving it.

`armbian-openvfd`, which ophub bundles, doesn't help on these boxes. On current
kernels its modprobe-parameter device creation never fires, so
`/sys/class/leds/openvfd` is never created even though `armbian-openvfd` reports
`SUCCESS`. It also wants the same GPIO lines the device tree's `spi-gpio` node
already owns. The working fix is the in-kernel-style `tm16xx` driver, which binds
to the existing device-tree node with no device-tree changes.

## Assumptions (what this was verified on)

| | |
|---|---|
| Box | Vontar **X96 Max**, 4/64 |
| SoC | Amlogic **S905X2** (`meson-g12a`) |
| OS image | **ophub Armbian** (`https://github.com/ophub/amlogic-s9xxx-armbian`), installed to eMMC |
| Kernel | `6.18.33-ophub` (aarch64); also expected to work on other ophub 6.x kernels |
| Panel controller | **FD628 / TM1628** on `spi-gpio` (CLK/DAT/STB = GPIO 64/63/10) |

Other Amlogic boxes (S905X2/X3) that use the same `fdhisi,fd628` / `titanmec,tm1628`
`spi-gpio` binding will probably work too, though I haven't tested them. Run Step 0
to check before assuming.

> ⚠️ Don't use this on boxes whose panel genuinely needs `armbian-openvfd`. It
> applies only where the device tree already instantiates an `fd628`/`tm1628` SPI
> device that has no driver bound.

## Prerequisites

- Root / `sudo` access.
- About 500 MB free on `/`, for `build-essential`, DKMS, and the build itself.
- Kernel headers for the running kernel. ophub usually puts them in
  `/usr/src/linux-headers-$(uname -r)` (Step 1 verifies this). If they're missing,
  reinstall the kernel together with its header package; ophub's `armbian-update`
  ships a `header-*.tar.gz`. DKMS can't build without them.
- Internet access, for `apt` and to clone the driver.
- Packages `git`, `dkms`, `build-essential` (Step 1 installs them).
- The driver source, `jefflessard/tm16xx-display` (Step 2 clones it).

## Step 0. Confirm your box matches

```bash
# Should report an X96 Max / S905X2-class board:
cat /proc/device-tree/model; echo

# The panel device must exist and report an fd628/tm1628 compatible:
cat /sys/bus/spi/devices/spi0.0/of_node/compatible | tr '\0' ' '; echo
# expected: fdhisi,fd628 titanmec,tm1628
```

If `/sys/bus/spi/devices/spi0.0` doesn't exist, your device tree doesn't
instantiate the panel this way and the recipe won't apply, so stop here. If the
`compatible` string differs, the driver may still cover your controller (it
handles tm1618/tm1620/tm1628/tm1650, fd620/fd628/fd650, aip1628, pt6964, hbs658,
and more), but the steps below assume the fd628/tm1628 layout the X96 Max ships.

## Step 1. Install build prerequisites and check headers

```bash
sudo apt update
sudo apt install -y git dkms build-essential

# Verify kernel headers are present for the RUNNING kernel:
ls -d /usr/src/linux-headers-$(uname -r) && \
ls /lib/modules/$(uname -r)/build/Makefile
```

Both commands must succeed. If the headers are missing, restore them before
continuing (see Prerequisites).

## Step 2. Get the sources

```bash
cd ~
git clone --depth 1 https://github.com/jefflessard/tm16xx-display
git clone --depth 1 https://github.com/B00StER/x96max-armbian-frontpanel
```

## Step 3. Stage the DKMS source tree

We assemble a self-contained DKMS package at `/usr/src/tm16xx-1.0` from the driver
sources plus this repo's `Makefile` and `dkms.conf`. The keypad sub-module is left
out on purpose.

```bash
sudo rm -rf /usr/src/tm16xx-1.0
sudo mkdir -p /usr/src/tm16xx-1.0/include

# driver sources (note: no tm16xx_keypad.c)
cd ~/tm16xx-display/drivers/auxdisplay
sudo cp line-display.c line-display.h tm16xx.h tm16xx_compat.h \
        tm16xx_core.c tm16xx_spi.c tm16xx_i2c.c /usr/src/tm16xx-1.0/
sudo cp -r ~/tm16xx-display/include/. /usr/src/tm16xx-1.0/include/

# this repo's ophub-tailored Kbuild + DKMS config
sudo cp ~/x96max-armbian-frontpanel/dkms/Makefile  /usr/src/tm16xx-1.0/Makefile
sudo cp ~/x96max-armbian-frontpanel/dkms/dkms.conf /usr/src/tm16xx-1.0/dkms.conf
```

## Step 4. Build and install the module via DKMS

```bash
sudo dkms add     -m tm16xx -v 1.0
sudo dkms build   -m tm16xx -v 1.0
sudo dkms install -m tm16xx -v 1.0 --force

sudo dkms status tm16xx
# expect: tm16xx/1.0, <your-kernel>, aarch64: installed
```

This installs `line-display.ko`, `tm16xx.ko`, `tm16xx_spi.ko`, and `tm16xx_i2c.ko`
into `/lib/modules/$(uname -r)/updates/dkms/` (where the bundled `line-display`
overrides the kernel's stub) and runs `depmod`.

## Step 5. Load the module at boot

The panel device's modalias (`spi:fd628`) doesn't match the module's
(`spi:tm1628`), so udev coldplug won't auto-load it. Force-load it at boot, then
load it now:

```bash
sudo cp ~/x96max-armbian-frontpanel/system/etc/modules-load.d/tm16xx.conf \
        /etc/modules-load.d/tm16xx.conf
sudo modprobe tm16xx_spi

# the panel device should now have a driver + LED nodes:
ls /sys/class/leds/ | grep display
# expect: display  display::usb  display::apps  display::setup  display::sd  display::hd  display::cvbs  display::colon
```

## Step 6. Install the boot-time self-heal (survives kernel updates)

`armbian-update` ships new-kernel headers but doesn't run DKMS, so a freshly
updated kernel boots with the module missing. This service rebuilds it on the next
boot whenever it's absent.

```bash
cd ~/x96max-armbian-frontpanel
sudo install -m 0755 system/usr/sbin/tm16xx-dkms-ensure /usr/sbin/tm16xx-dkms-ensure
sudo install -m 0644 system/etc/systemd/system/tm16xx-dkms.service \
        /etc/systemd/system/tm16xx-dkms.service
sudo mkdir -p /etc/systemd/system/display.service.d
sudo install -m 0644 system/etc/systemd/system/display.service.d/10-dkms-ensure.conf \
        /etc/systemd/system/display.service.d/10-dkms-ensure.conf
sudo systemctl daemon-reload
sudo systemctl enable tm16xx-dkms.service
```

## Step 7. Install the clock/status daemon and enable it

The display content (clock plus status icons) is driven by jefflessard's
`display-service`:

```bash
cd ~/tm16xx-display
sudo install -m 0755 display-service /usr/sbin/display-service
sudo install -m 0644 display.service /lib/systemd/system/display.service
sudo systemctl daemon-reload
sudo systemctl enable --now display.service
```

The panel should now show the time.

## Step 8. Verify

```bash
systemctl is-active display tm16xx-dkms          # both: active
cat /sys/class/leds/display/message              # current text (e.g. the time, "2209")
basename "$(readlink /sys/bus/spi/devices/spi0.0/driver)"   # tm16xx-spi
```

Optional: `sudo reboot`, then check the panel shows the clock again with no manual
steps.

## After a kernel update (`armbian-update`)

Nothing to do. On the next boot, `tm16xx-dkms.service` sees the module isn't built
for the new kernel and rebuilds it via DKMS (the new kernel's headers come with
`armbian-update`), then loads it so `display.service` can start. If a kernel
release ever ships without headers, the panel stays dark until you install them
and run `sudo dkms autoinstall`.

## Uninstall

```bash
sudo systemctl disable --now display.service tm16xx-dkms.service
sudo rm -f /usr/sbin/display-service /lib/systemd/system/display.service \
           /usr/sbin/tm16xx-dkms-ensure /etc/systemd/system/tm16xx-dkms.service \
           /etc/modules-load.d/tm16xx.conf
sudo rm -rf /etc/systemd/system/display.service.d
sudo dkms remove -m tm16xx -v 1.0 --all
sudo rm -rf /usr/src/tm16xx-1.0
sudo systemctl daemon-reload
```

## Credits and license

The display driver is [jefflessard/tm16xx-display](https://github.com/jefflessard/tm16xx-display)
(GPL-2.0). All credit for making the panel work belongs there, and this repo
doesn't redistribute it. The diagnosis came out of an
[ophub](https://github.com/ophub/amlogic-s9xxx-armbian) image issue,
[#3567](https://github.com/ophub/amlogic-s9xxx-armbian/issues/3567). The glue here
(DKMS packaging, the Kbuild tweaks, the boot self-heal) is MIT-licensed; see
`LICENSE`.

This is a stopgap. Once `tm16xx` lands in mainline and reaches ophub's kernel,
none of it will be needed; if you can, support the
[upstream series](https://lists.openwall.net/linux-kernel/2025/11/21/1304).

Tested on a single X96 Max (S905X2). No warranty, best-effort. Issues and PRs are
welcome, though I maintain it as time allows.
