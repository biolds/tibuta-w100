This project contains notes about setting up Debian Buster on the Tibuta MasterPad W100 tablet (<https://www.amazon.com/Tibuta-Masterpad-Computer-1536%C3%972048-Keyboard/dp/B09LS6Y2KT>).
It is highly recomended to backup the Windows partition, or make a dual boot when installing Linux.

It aims to cover system setup and [IceWM](https://ice-wm.org/) lightweight window manager setup. Not yet working:

- Sound card
- Volume button
- Wifi
- Display power management bug (sometime after going into power saving mode, the screen won't light back)
- Screen sometime blinking in console mode
- Accelerometers (automatically switch between landscape and portrait modes)
- Suspend on lid close

Feel free to post a pull request to improve this doc or open a discussion.

# Touchscreen

Using the touchscreen requires rebuilding the kernel to enable custom settings, and update the `silead` module. The updated kernel can be downloaded from this repository: [linux-image-5.10.162+_5.10.162+-1_amd64.deb](/uploads/Home/linux-image-5.10.162+_5.10.162+-1_amd64.deb)

The [firmware](/uploads/Home/gsl1680-tibuta-w100.fw) needs to be copied manually into `/lib/firmware/silead/`.

```sh
export PATH=/sbin:/usr/sbin:$PATH
dpkg -i linux-image-5.10.158+_5.10.158+-1_amd64.deb
cp gsl1680-tibuta-w100.fw /lib/firmware/silead/
update-grub
```

## Kernel building

The kernel can also be built manually, based on [gls-firmware Github page](https://github.com/onitake/gsl-firmware).

A [patch](/uploads/Home/0001-tibuta-touchpad-module.patch) is required to load the new firmware, updated debian kernel conf and cert, etc.

```sh
apt-get install build-essential linux-source bc kmod cpio flex libncurses5-dev libelf-dev libssl-dev dwarves bison python3
cd /root
tar xaf /usr/src/linux-source-5.10.tar.xz
cd linux-source-5.10
patch --dry-run < 0001-tibuta-touchpad-module.patch # try with -p1 or -p0 in case of error, then remove dry-run
make -j2 deb-pkg
```

## Setup/integration

Using the kernel above the touchscreen does not match the screen matrix, this can adapted using the following scripts (which also handle screen rotation, and resizing), and installing the `xinput` package:

`/usr/local/bin/screen-landscape`
```sh
#!/usr/bin/bash
xrandr --output eDP-1 --scale 1x1
xrandr -o left
xinput set-prop silead_ts 'Coordinate Transformation Matrix' 0 -1.17 1.17 1.32 0 -0.32 0 0 1
xrandr --output eDP-1 --scale 0.5x0.5
```

`/usr/local/bin/screen-portrait`
```sh
#!/usr/bin/bash
xrandr --output eDP-1 --scale 1x1
xrandr -o normal
xinput set-prop silead_ts 'Coordinate Transformation Matrix' 1.32 0 -0.32 0 1.17 -0.17 0 0 1
xrandr --output eDP-1 --scale 0.5x0.5
```

`/usr/local/bin/screen-switch`
```sh
#!/usr/bin/bash
if xrandr | grep '^eDP-1 connected primary 1024x768+0+0 left'
then
    /usr/local/bin/screen-portrait
else
    /usr/local/bin/screen-landscape
fi
```

_Note: the matrix coordinates are usable as is but could be improved, please make a PR if you fine-tune them_

# HDMI output

HDMI output works (the sound output does not), the following script will setup HDMI output while keeping the tablet screen in landscape scaled mode, but this breaks the touchscreen scaling done above (I did not bother building the correct xinput commands, since a mouse is required anyway to access the external screen):

`/usr/local/bin/screen-hdmi`
```sh
#!/bin/bash
xrandr --output HDMI-1 --off # Reset the output in case it's in a bad state
xrandr --output eDP-1 --scale 1x1 --auto
xrandr --output HDMI-1 --auto --mode 1920x1080
xrandr --output HDMI-1 --auto --right-of eDP-1
xrandr --output eDP-1 --mode 1536x2048 -o left --scale 0.5x0.5
```

`/usr/local/bin/screen-hdmi-off`
```
#!/bin/bash
xrandr --output HDMI-1 --off
```

# Touchscreen gestures

[Touchegg](https://github.com/JoseExposito/touchegg) can be installed to handle right-click with 2 fingers.

Scrolling with 2 fingers requires using `xdotool` and some  custom configuration, as described on <https://wiki.archlinux.org/title/Touchegg> and setting the [thresholds parameters](https://github.com/JoseExposito/touchegg#daemon-configuration) in systemd's configuration. This gave some results with:

```
ExecStart=/usr/bin/touchegg --daemon 10 15
```

_Note: this conflicts with Chromium built-in's multi-touch scrolling / zooming_

On IceWM, `touchegg-client` needs to be run at startup, in `.icewm/startup` with `/usr/bin/touchegg --client &`.

# Login screen

[LightDM](https://doc.ubuntu-fr.org/lightdm) setup for tablet that don't handle screen rotation:

- screen scaling, and landscape mode: [lightdm.conf](/uploads/Home/lightdm.conf)
- conf to have [Onboard](https://launchpad.net/onboard) in the accesibility menu: [lightdm-gtk-greeter.conf](/uploads/Home/lightdm-gtk-greeter.conf)
(to be put in `/etc/lightdm`)


# Backlight

The backlight can be changed through `/sys/class/backlight/intel_backlight`, and can be updated with keyboard shortcuts using `xbindkeys`:

`/usr/local/bin/backlight-dec`
```sh
#!/bin/bash
set -e

PERCENT=10
SYSFS_BL=/sys/class/backlight/intel_backlight
MAX="$(cat $SYSFS_BL/max_brightness)"
CURRENT="$(cat $SYSFS_BL/brightness)"

STEP="$(($MAX * $PERCENT / 100))"
NEW="$((CURRENT - STEP))"

if [ "$NEW" -lt 0 ]
then
	NEW=0
fi

echo "$NEW" > $SYSFS_BL/brightness
```

`/usr/local/bin/backlight-inc`
```sh
#!/bin/bash
set -e

PERCENT=10
SYSFS_BL=/sys/class/backlight/intel_backlight
MAX="$(cat $SYSFS_BL/max_brightness)"
CURRENT="$(cat $SYSFS_BL/brightness)"

STEP="$(($MAX * $PERCENT / 100))"
NEW="$((CURRENT + STEP))"

if [ "$NEW" -gt "$MAX" ]
then
	NEW="$MAX"
fi

echo "$NEW" > $SYSFS_BL/brightness
```

`~/.xbindkeysrc`
```
# Increase backlight
"/usr/local/bin/backlight-inc"
   XF86MonBrightnessUp

# Decrease backlight
"/usr/local/bin/backlight-dec"
   XF86MonBrightnessDown
```

`xbindkeys` then needs to be started from `~/.icewm/startup`

# Suspend on power button

Set `HandlePowerKey=suspend` in `/etc/systemd/login.conf`

# Bluetooth

Works out of the box (tested as a network adapter), see <https://wiki.debian.org/BluetoothUser>.

# Debian packages

```
apt install lightdm icewm libpugixml1v5 xinput xdotool xbindkeys \
	firefox-esr pcmanfm onboard sakura papirus-icon-theme bluetooth rfkill blueman
```
