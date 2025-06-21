---
publish: true
---

# Nut-Monitor

## Introduction

I've been doing system maintenance and discovered a few issues with my current setup, so I've decided to document the process and share it. 

I'm Using a CTB-1200 UPS. It's connected via usb. After following the [wiki](https://wiki.archlinux.org/title/Network_UPS_Tools#) instruction and set up a basic configuration, the following files need to be modified for the installation to be effective.

## Configuration

### Driver configuration

Not every model is compatible, I was lucky enough to find my hardware listed in [Network UPS Tools](https://networkupstools.org/stable-hcl.html) page, it provides the required configuration parameters.

There are several files that need to be modified according to the user's UPS model residing in the `/etc/nut` folder. 

```shell
/etc/nut
├── hosts.conf
├── nut.conf
├── ups.conf # Driver Settings
├── upsd.conf
├── upsd.users # Required to send commands to the daemon
├── upsmon.conf # Daemon configuration
├── upssched.conf # Timer and command schedule
├── upssched-execscript.sh # Custom user script
├── upsset.conf
├── upsstats.html
└── upsstats-single.html
```

#### **nut.conf**
 
 In this file, change the following line from 'none' to 'standalone' to enable the monitor.

```sh
# Valores por defecto omitidos
MODE=standalone 
```

#### **ups.conf**

This file is where the driver configuration is stored. Parameters compatibility will depend on the driver. The following are general purpose so they should work for most drivers.

```shell
# Default values and file comments ommitted, 
# append configuration to desired driver.
[ctb-1200]
    driver = blazer_usb
    port = auto
    offdelay = 20
    ondelay = 30
    ignorelb
    override.battery.charge.low = 20
    override.battery.charge.warning = 40
```

#### **ups.users**

This file is where the user configuration is stored. It's required to send commands to the daemon.

```sh
[dnloop]
     password = 1212
     upsmon master
     actions = SET
     instcmds = ALL
```

#### **upsmon.conf**

This file is where the daemon configuration is stored. It provides an interface to send commands to the device.

```sh
# Default values ommitted
SHUTDOWNCMD "/sbin/shutdown -h +0"
NOTIFYCMD /usr/sbin/upssched
NOTIFYFLAG ONLINE      SYSLOGW+WALL
NOTIFYFLAG ONBATT      SYSLOG+WALL
MONITOR ctb-1200@localhost 1 dnloop 1212 master # same credentials as ups.users
```

#### **upssched.conf**

This file is where the timer and command schedule is stored. Here we can tune the time to wait until power reconnection or shutdown. It's advisable to not wait too much in a power outage due to aging of battery, it's better to have a quick shutdown than a long one.

```sh
# Default values ommitted
CMDSCRIPT /etc/nut/upssched-execscript.sh

PIPEFN /etc/nut/upssched.pipe
LOCKFN /etc/nut/upssched.lock

AT ONBATT * START-TIMER shutdown_onbatt 40 # value in seconds
AT ONBATT * EXECUTE info_onbatt

AT ONLINE * CANCEL-TIMER shutdown_onbatt
AT ONLINE * EXECUTE ups-back-on-power

AT LOWBATT * EXECUTE shutdown_lowbatt

AT REPLBATT * EXECUTE replace_batt

```

#### Custom user script

This script is where the custom commands are stored. It's called by `upssched.conf`. It can be modified to execute appropriate handling of services and notify event loggers. I'm only logging and forcing a shutdown for now.

**upssched-execscript.sh**

```sh
#! /bin/sh
case $1 in
    shutdown_onbatt)
        logger -t upsmon[upssched] "shutdown_onbatt): Triggering shutdown after being on battery"
        /sbin/shutdown -h +0
        ;;

    shutdown_lowbatt)
        logger -t upsmon[upssched] "shutdown_lowbatt): Triggering shutdown when battery.charge.low is under 50%"
        /sbin/shutdown -h +0
        ;;

    info_onbatt)
        logger -t upsmon[upssched] "info_onbatt): Now on battery"
        ;;

    ups-back-on-power)
        logger -t upsmon[upssched] "ups-back-on-power): UPS back on power"
        ;;

    replace_batt)
        message="Quick self-test indicates battery requires replacement"
        logger -t upsmon[upssched] "replace_batt): $message"
        ;;

    *)
        logger -t upsmon[upssched] "*) = Unrecognized command: $1"
        ;;
esac

```

### Systemd Targets

These are systemd targets extracted from Arch wiki, they should be present after following along the guide.

```sh
╭─ /etc/systemd/system/

nut-driver@ctb-1200.service.d
├── nut-driver-enumerator-generated-checksum.conf
├── nut-driver-enumerator-generated.conf
├── nut-driver-enumerator-generated-devicename.conf
└── nut-driver-enumerator-generated-doclink.conf
nut-driver@.service.d
└── nut-driver-enumerator-generated-checksum.conf
nut-driver.target.wants
└── nut-driver@ctb-1200.service -> /usr/lib/systemd/system/nut-driver@.service
nut.target.wants
├── nut-driver-enumerator.service -> /usr/lib/systemd/system/nut-driver-enumerator.service
├── nut-driver.target -> /usr/lib/systemd/system/nut-driver.target
└── nut-server.service -> /usr/lib/systemd/system/nut-server.service

```

### Summary of services

* nut-server.service
* nut-monitor.service
* nut-driver-enumerator.service

### Post Installation

A working installation can be checked with: 

```sh
sudo upsd
```

should output something similar to:

```sh
Network UPS Tools upsd 2.8.3 release
not listening on ::1 port 3493
not listening on 127.0.0.1 port 3493
no listening interface available
```

A proper configured server should show the following similar entries, the most recent is when the ups is plugged off the wall. The older ones shows a successful server start:

**`journalctl -rxb` output**

```sh
Jun 21 16:02:36 lyra nut-monitor[1353]: UPS ctb-1200@localhost on battery
Jun 21 16:02:36 lyra kernel: dragonrise 0003:0079:0006.0006: Force Feedback for DragonRise Inc. game controllers by Richard Walmsley <richwalm@gmail.com>
Jun 21 16:02:36 lyra kernel: dragonrise 0003:0079:0006.0006: input,hidraw1: USB HID v1.10 Joystick [DragonRise Inc.   Generic   USB  Joystick  ] on usb-0000:06:00.3-1/input0
Jun 21 16:02:36 lyra kernel: input: DragonRise Inc.   Generic   USB  Joystick   as /devices/pci0000:00/0000:00:08.1/0000:06:00.3/usb3/3-1/3-1:1.0/0003:0079:0006.0006/input/input26
Jun 21 16:02:36 lyra kernel: usb 3-1: Manufacturer: DragonRise Inc.  
Jun 21 16:02:36 lyra kernel: usb 3-1: Product: Generic   USB  Joystick  
Jun 21 16:02:36 lyra kernel: usb 3-1: New USB device strings: Mfr=1, Product=2, SerialNumber=0
Jun 21 16:02:36 lyra kernel: usb 3-1: New USB device found, idVendor=0079, idProduct=0006, bcdDevice= 1.07
Jun 21 16:02:36 lyra kernel: usb 3-1: new low-speed USB device number 4 using xhci_hcd
Jun 21 16:02:36 lyra kernel: usb 3-1: USB disconnect, device number 2
Jun 21 16:01:51 lyra nut-monitor[1353]: Communications with UPS ctb-1200@localhost established
Jun 21 16:01:51 lyra upsd[7763]: User dnloop@::1 logged into UPS [ctb-1200]
Jun 21 16:01:51 lyra nut-server[7763]: User dnloop@::1 logged into UPS [ctb-1200]
Jun 21 16:01:49 lyra sudo[7776]: pam_unix(sudo:session): session closed for user root
Jun 21 16:01:49 lyra sudo[7776]: pam_unix(sudo:session): session opened for user root(uid=0) by dnloop(uid=1000)
Jun 21 16:01:49 lyra sudo[7776]:   dnloop : TTY=pts/0 ; PWD=/etc/nut ; USER=root ; COMMAND=/usr/bin/systemctl status nut-server.service
Jun 21 16:01:46 lyra nut-monitor[1353]: Communications with UPS ctb-1200@localhost lost
Jun 21 16:01:46 lyra nut-monitor[1353]: Poll UPS [ctb-1200@localhost] failed - Driver not connected
Jun 21 16:01:42 lyra upsd[7763]: upsnotify: logged the systemd watchdog situation once, will not spam more about it
Jun 21 16:01:42 lyra upsd[7763]: upsnotify: failed to notify about state NOTIFY_STATE_READY_WITH_PID: no notification tech defined, will not spam more about it
Jun 21 16:01:42 lyra upsd[7763]: upsnotify: notify about state NOTIFY_STATE_READY_WITH_PID with libsystemd: was requested, but not running as a service unit now, will not spam more about it
Jun 21 16:01:42 lyra nut-server[7763]: upsnotify: logged the systemd watchdog situation once, will not spam more about it
Jun 21 16:01:42 lyra nut-server[7763]: upsnotify: failed to notify about state NOTIFY_STATE_READY_WITH_PID: no notification tech defined, will not spam more about it
Jun 21 16:01:42 lyra upsd[7763]: Running as foreground process, not saving a PID file
Jun 21 16:01:42 lyra nut-server[7763]: upsnotify: notify about state NOTIFY_STATE_READY_WITH_PID with libsystemd: was requested, but not running as a service unit now, will not spam more about it
Jun 21 16:01:42 lyra nut-server[7763]: Running as foreground process, not saving a PID file
Jun 21 16:01:42 lyra sudo[7755]: pam_unix(sudo:session): session closed for user root
Jun 21 16:01:42 lyra blazer_usb[1343]: sock_connect: enabling asynchronous mode (auto)
Jun 21 16:01:42 lyra upsd[7763]: Found 1 UPS defined in ups.conf
Jun 21 16:01:42 lyra upsd[7763]: Connected to UPS [ctb-1200]: blazer_usb-ctb-1200
Jun 21 16:01:42 lyra nut-server[7763]: Found 1 UPS defined in ups.conf
Jun 21 16:01:42 lyra nut-server[7763]: Connected to UPS [ctb-1200]: blazer_usb-ctb-1200
Jun 21 16:01:42 lyra upsd[7763]: listening on 127.0.0.1 port 3493
Jun 21 16:01:42 lyra upsd[7763]: listening on ::1 port 3493
Jun 21 16:01:42 lyra nut-server[7763]: listening on 127.0.0.1 port 3493
Jun 21 16:01:42 lyra nut-server[7763]: listening on ::1 port 3493
Jun 21 16:01:42 lyra nut-server[7763]: Network UPS Tools upsd 2.8.3 release
Jun 21 16:01:42 lyra systemd[1]: Started Network UPS Tools - power devices information server.

```

To get information about the ups run `upsc <upsname>`

```she
upsc ctb-1200
battery.charge: 100
battery.charge.low: 20
battery.charge.warning: 40
battery.voltage: 27.10
battery.voltage.high: 26.00
battery.voltage.low: 20.80
battery.voltage.nominal: 24.0
device.type: ups
driver.debug: 0
driver.flag.allow_killpower: 0
driver.flag.ignorelb: enabled
driver.name: blazer_usb
driver.parameter.offdelay: 20
driver.parameter.ondelay: 30
driver.parameter.override.battery.charge.low: 20
driver.parameter.override.battery.charge.warning: 40
driver.parameter.pollinterval: 2
driver.parameter.port: auto
driver.parameter.synchronous: auto
driver.state: quiet
driver.version: 2.8.3
driver.version.internal: 0.21
driver.version.usb: libusb-1.0.29 (API: 0x0100010A)
input.current.nominal: 5.0
input.frequency: 50.0
input.frequency.nominal: 50
input.voltage: 224.4
input.voltage.fault: 224.4
input.voltage.nominal: 220
output.voltage: 226.5
ups.beeper.status: enabled
ups.delay.shutdown: 18
ups.delay.start: 1800
ups.load: 5
ups.productid: 5161
ups.status: OL
ups.type: offline / line interactive
ups.vendorid: 0665
```

## References

* [Network UPS Tools](https://wiki.archlinux.org/title/Network_UPS_Tools#NUT-Monitor)
* [NUT & UPS, howto shut down client after 2 min on battery?](https://community.ipfire.org/t/nut-ups-howto-shut-down-client-after-2-min-on-battery/9096/6)
* [nut – testing the shutdown mechanism](https://dan.langille.org/2020/09/10/nut-testing-the-shutdown-mechanism/)
* [6. Configuration notes](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html)
