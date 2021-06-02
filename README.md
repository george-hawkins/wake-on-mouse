Disabling wake on mouse
=======================

**TLDR;** assuming that the file `/etc/udev/rules.d/10-wakeup.rules` doesn't exist. Just run the included script like so:

```
$ ./wakeup-rule | sudo tee /etc/udev/rules.d/10-wakeup.rules > /dev/null
```

And then plug your USB mouse in and out to activate the rule.

The above script works for Ubuntu 20.04 LTS. If you run it on its own you'll see it produces output like this:

```
# Udev rule to disable wakeup for Logitech_USB_Receiver.
ACTION=="bind", ATTRS{idVendor}=="046d", ATTRS{idProduct}=="c52b", ATTR{power/wakeup}=="*", ATTR{power/wakeup}="disabled"
```

The `idVendor` and `idProduct` values are the USB vendor and product IDs specific to your particular make of USB mouse. In the case above, they're actually the values for the USB bluetooth receiver for a wireless Logitech mouse. If you've got multiple devices connected to such a receiver, e.g. mouse and keyboard, then wakeup will be disabled for all these devices. In this situation, you can still wake your computer by e.g. quickly pressing the power button.

For other setups, it may be necessary to change the `ACTION` above from `bind` to e.g. `add`. Finding the appropriate action is discussed below.

Details
-------

Find the mouse USB vendor and product ID:

    $ lsusb
    Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
    Bus 001 Device 006: ID 413c:2107 Dell Computer Corp. 
    Bus 001 Device 029: ID 046d:c52b Logitech, Inc. Unifying Receiver
    Bus 001 Device 004: ID 05e3:0610 Genesys Logic, Inc. 4-port hub
    Bus 001 Device 005: ID 8087:0026 Intel Corp. 
    Bus 001 Device 002: ID 103c:84fd HP TracerLED
    Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
    Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
    Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

In my case above, it's the `Logitech` device and its ID is shown as `046d:c52b`. The vendor ID is `046d` and the product ID is `c52b`.

An alternative way to find this values is to look for `idVendor` and `idProduct` in the output of:

    $ udevadm info --name=/dev/input/mouse0 --attribute-walk

Create a udev rule file:

    $ sudo vim /etc/udev/rules.d/10-wakeup.rules

And add the line:

    ACTION=="bind", ATTRS{idVendor}=="046d", ATTRS{idProduct}=="c52b", ATTR{power/wakeup}=="*", ATTR{power/wakeup}="disabled"

Change `046d` and `c52b` to match your device.

That's it - just plug the device in and out and the rule will activate.

Notes
-----

There are lots of pages on the web with differing suggestions as to how to do this. For nearly everything I tried, I ended up with a line like this in `/var/log/syslog`:

    systemd-udevd[88948]: 1-7.1: /etc/udev/rules.d/10-wakeup.rules:1 Failed to write ATTR{/sys/devices/pci0000:00/0000:00:14.0/usb1/1-7/1-7.1/power/wakeup}, ignoring: No such file or directory

It turned out there were two issues:

* Most pages list the relevant `ACTION` as `add` whereas on my system this attribute is only available when `bind` occurs.
* The matching rules match device children, so even though the `ATTRS{idProduct}=="c52b"` matched the correct device, it also matched children that didn't have the `power/wakeup` attribute. Adding the check `ATTR{power/wakeup}=="*"` resolved this.

To find the correct action, I added the following temporary rule to `/etc/udev/rules.d/10-wakeup.rules`:

    ATTRS{idVendor}=="046d", ATTRS{idProduct}=="c52b", ATTR{power/wakeup}=="*" RUN+="/bin/sh -c 'echo $(date --rfc-3339=seconds) $env{DEVPATH} $env{ACTION} >> /tmp/udev-wakeup.log 2>&1'"

I.e. a very unconstrained rule, with just checks for the particular device and the `power/wakeup` attribute. Then after plugging the device in and out, I could find the relevant action in `/tmp/udev-wakeup.log`:

    $ cat /tmp/wkup 
    2021-06-02 09:41:25+02:00 /devices/pci0000:00/0000:00:14.0/usb1/1-7/1-7.1 bind
    2021-06-02 09:51:05+02:00 /devices/pci0000:00/0000:00:14.0/usb1/1-7/1-7.1/1-7.1:1.2/0003:046D:C52B.0051/0003:046D:101B.0052/power_supply/hidpp_battery_18 change

Note that this also picked up a battery-related child device - but I could see that if I checked for just the `bind` action I would get the device I wanted.

You can check the current `power/wakeup` value by taking the device path shown above and doing:

    $ dev=/devices/pci0000:00/0000:00:14.0/usb1/1-7/1-7.1
    $ cat /sys$dev/power/wakeup
    enabled

You might wonder how you know what environment values, e.g. `DEVPATH` and `ACTION` above, you can check for. You can discover this with:

    $ udevadm monitor --environment --udev

Just plug the device in and out and you'll see it display the relevant events and the related environment values.

Note: `--environment` is not mentioned at all in the `man` page for `udevadm` on my system.

And if you want to see what else you can match on you can do:

    $ udevadm info --name=/dev/bus/usb/001/029 --attribute-walk

Where `001/029` are the values you saw for `Bus` and `Device` when using `lsusb` up above.

Other interesting alternatives to `--attribute-walk` are `--query=all` and `--query=property`.

An alternative to using `lsusb` at all is to use `udevadm info` about your mouse device:

    $ udevadm info --name=/dev/input/mouse0 --attribute-walk

You'll see the relevant `idVendor` and `idProduct` values listed for one of its parents.

You can also use `udevadm info` with a `/devices` path (like above, i.e. something under `/sys`) using `--path` instead of `--name`:

    $ udevadm info --path=/devices/pci0000:00/0000:00:14.0/usb1/1-7/1-7.1 --attribute-walk

Another way to find the relevant USB device, if you have the product ID, is like so:

    $ grep c52b /sys/bus/usb/devices/*/idProduct

You can then see if wakeup is enabled for this device like so:

    $ dev=$(fgrep -l c52b /sys/bus/usb/devices/*/idProduct)
    $ cat ${dev%/*}/power/wakeup
    enabled
