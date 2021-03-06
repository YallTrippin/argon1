# argon1

![LXPanel plugin default view](../assets/lxplug-argon1-default.png?raw=true) &nbsp;
![LXPanel plugin medium speed](../assets/lxplug-argon1-medium.png?raw=true) &nbsp;
![LXPanel plugin high speed](../assets/lxplug-argon1-high.png?raw=true)


This package provides temperature-based fan control and power button monitoring for the Argon One case for Raspberry Pi 4.  It is a complete re-write and does not share any code with the official Argon40 packages (which were only used to figure out relatively simple hardware protocols).  


> ## DISCLAIMER
> 
> **The package is in no way affiliated or endorsed by Argon40, and is _not_ officially supported.**
> Use it at your own discretion and risk.

I just got an Argon One case recently, I had a free weekend, and this provided an excellent excuse to play with setuptools and dpkg (which I always wanted but never got around to doing).  This was a personal "distraction" project, which you _may_ find useful.  As such, feel free to do as you wish with it, but do **not** expect any support or serious maintenance from me!

# Differences from official scripts

The main differences from the Argon40 script are:

* The daemon is much more configurable, via `/etc/argonone.yaml`.
* The daemon registers a system D-Bus service, which publishes notification signals, as well as methods to query and control it.
* An LXPanel plugin is available, to show fan status and easily pause/unpause fan from a desktop session.
* The daemon does not run as root.
* Users can be selectively granted permission to control the daemon (all users can query it).
* The `argonctl` commandline utility provides an easy way to query/control over D-Bus.
* The daemon is installed as a systemd service.
* The package is "debianized" natively, and can be easily installed via `dpkg` or `apt`.

# Installation

Simply download the `.deb` file and install it via

```shell
sudo apt install argon1_x.y.z_all.deb
```

where `x.y.z` is the package version.

If the installer detects user accounts other than `pi`, it will prompt you to grant permission to control the daemon.  If you wish to add users yourself (e.g., if you want to selectively add a subset of user accounts, or if you create additional user accounts at a later time), you simply need to add them to the `argonone` group via

```shell
sudo usermod -a -G argonone username
```

where `username` should be replaced with the actual username of the user you wish to grant control permissions to.

## LXPanel plugin

If you also wish to install the LXPanel plugin, then *after* installing the `argon1_x.y.z_all.deb` package, also install the corresponding `lxplug-argon1_x.y.z_armhf.deb` package, in the same way.

> **Please make sure that you install matching `x.y.z` versions of `argon1` and `lxplug-argon1` DEBs.**  

Package dependencies *should* ensure minimum for compatibility, but it's always better to keep versions in-sync. 


## `apt` vs `dpkg`

You can also use `dpkg` to install the `.deb` packages.  However, in that case, you must manually install any missing dependencies.  The easiest way to do that is *after* installing the `.deb`, e.g.,

```shell
sudo dpkg -i packagefile.deb
sudo apt install -f
```

Alternatively, you can list the dependencies by, e.g.,
```shell
dpkg-deb --show --showformat='${Depends}\n' packagefile.deb
```
and then manually install the ones missing, in advance.

Of course, if you are installing `lxplug-argon1`, then `apt` cannot install the `argon1` dependency, since it's not in any APT repositories.

# Usage

## Command line

You can query the daemon using the `argonctl` utility command:

* `argonctl speed` shows the current fan speed setting.
* `argonctl temp` shows the last CPU temperature measurement.
* `argonctl pause` pauses temperature-based fan control; the fan will stay at whatever speed it was at the time the command was executed.
* `argonctl resume` resumes temperature-based fan control.
* `argonctl set_speed NNN` will set the fan speed to the requested value (must be between 0..100); if temperature-based fan control is not paused, then the daemon may change it the next time the temperature is measured (by default, this happens every 10 seconds).
* `argonctl lut` shows the currently configured fan speed lookup table (LUT).

There are a few additional commands that are probably less useful.  If you wish to shutdown the daemon, please do so via systemd, e.g., `sudo systemctl stop argonone`.  If you use `argonctl shutdown` directly, systemd will think the daemon crashed and will attempt to restart it.

## LXPanel plugin UI

![LXPanel plugin screenshot](../assets/lxplug-argon1.png?raw=true)

Use of the panel plugin should be self-explanatory.  It can be added just like any other plugin (right-click on panel and select "Add/Remove Panel Items").

Clicking on the plugin icon will toggle fan control (specifically, if fan control was on, then it will pause fan control and set it's speed to zero, otherwise it will unpause fan control, which should eventually adjust speed according to current temperature). A context menu with a few other common actions is accessible via right-click.

Configuration of the plugin is fairly basic: you can choose whether you want a label in the panel (if hidden, you have to mouse over for tooltip) and whether you want temperature information to be displayed as well.

### Restarting `lxpanel`

Please note that, as `lxpanel` is a monolithic app (which dynamically loads plugins from `.so` files at startup), you may need to restart it for any changes to be reflected.  You can always log out and back in, but a less annoying way is via

```
lxpanelctl restart
```

> You may have to do this after installing or upgrading the `lxplug-argon1` package.  
> You may also have to do this if you (re)start the `argonone` system daemon.


# Daemon configuration

All configuration can be found in `/etc/argonone.yaml`, which should be self-explanatory.  

The default values should be fine and should not need to be adjusted.  The setting you are more likely to want to experiment with is the temperature-based fan control lookup table (LUT).  

If you modify the configuration file, then you need to restart the daemon for the changes to take effect, via 

```shell
sudo systemctl argonone restart
```

Finally, note that the `enabled` configuration values simply determine the _initial_ "paused"/"unpaused" state of each daemon component each time the daemon starts up.  However, this state can be toggled while the server is running, via the `argonctl` utility. For all other settings you _must_ restart the daemon (after editing `/etc/argonone.yaml`) to change them.

# Troubleshooting and monitoring

As stated earlier, you are on your own here! :)  However, you may wish to start by inspecting the systemd logs, e.g., via

```shell
systemctl status argonone
```

Furthermore, if you wish to monitor all daemon events, you can do so via, e.g.,

```shell
dbus-monitor --system "sender='net.clusterhack.ArgonOne'"
```

# Hardware protocol

The hardware protocol is not officially documented but can be inferred from the official scripts.  Some aspects are rather awkward (probably this is a "home-brew" protocol, not based on some standard IC for e.g., PWM control, and not intended for public consumption?).  In particular:

* The hardware uses both a GPIO pin (as an interrupt signal, from board to Pi communication), as well as I2C (device is a slave, for Pi to board communication).

* A signal is sent over GPIO whenever the power button is pressed.  This is a pulse, whose width encodes the press-type parameter: 10-30msec means "reboot" while 30-50msec means "shutdown".  Note that pulse widths do not have anything to do with physical button presses ("reboot" is triggered by a double-press, whereas "shutdown" is triggered by a press between 3-5 sec).
  * After a "reboot" request, the board will not do anything else.  However, it seems it will remember this and, after a "power off request" (see below) is received, it will restore power after cutting it briefly.
  * After a "shutdown" request, the board will wait for a brief period of time and then cut off power anyway.  Therefore, it is critical that the system `shutdown` command is issued ASAP.
  Unfortunately, it seems that this poweroff delay is hardcoded into the board's firmware.

  > If your shutdown sequence takes longer than 1-2sec (e.g., if you mount network drives that need to be flushed and umounted, for instance), it would be better to avoid using the case's power button to initiate a shutdown.

* On I2C, the device address is 0x1a (26) and only register 0x00 is used for everything.  The register can only be written and it is "overloaded" for various things:
  * Values between 0-100 (inclusive) are interpreted as fan speeds.
  * The value 0xff (255) is interpreted as a "power off request".  This is apparently intended to be sent by a systemd/initd shutdown script, so that the board knows to cut power even if the shutdown is performed by issuing a `shutdown` command manually (but, see above for exception).

&lt;rant&gt; For what it's worth, IMHO the hardware protocol incorporates unnecessary, hardcoded logic.  It would be better to delegate all logic to the Raspberry Pi:

* Button presses should merely convey information about the event, but **not** initiate any actions.  Those should be initiated by the Pi.  In fact, I see no reason why the button press patterns should be hardcoded in the firmware; the button signal could be passed verbatim to the Pi's GPIO without any interpretation (maybe debouncing, but that's it).  This would further allow easily changing or adding button press patterns in software.
* Instead of the doubly-overloaded I2C register 0x00 and command 0xFF (with semantics of "turn off power and stay off, except if a "reboot" button press was sent \[how long ago?\], in which case turn power off and then back on"), the board could use another register (e.g., 0x01) for simple power-related commands.  These should include at least a "power off and stay off" command code, as well as a "power off then back on" command.  If implementing some minimal logic was desired (e.g., so that customers don't get disappointed if the button does nothing without software installed), then perhaps a "disable power actions" command code should also be included (which control software would issue immediately upon starting up -- and, perhaps, a separate "enable power actions" command could be optionally issued when control software exits).

But, anyway, the protocol is what it is, and the overall experience is not at all bad, despite some minor shortcomings!  The case design itself (and it's cooling capacity) is great, and that's what really matters. &lt;/rant&gt;