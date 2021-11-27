---
title: "Configure NUT for OPNsense and TrueNAS with the CyberPower PR750ERT2U UPS"
date: 2021-11-25T15:10:22+01:00
draft: true
comments: true
toc: false
cover:
  src: "cover.jpg"
toc: true
tags:
  - CyberPower
  - Homelab
  - NUT
  - OPNsense
  - TrueNAS
  - UPS
---

For storage in my homelab, I use [TrueNAS](https://www.truenas.com/). Additionally, I run a couple of apps on top of it as [jails](https://docs.freebsd.org/en/books/handbook/jails/). For over a year, I've been using an [uninterruptible power supply (UPS)](https://en.wikipedia.org/wiki/Uninterruptible_power_supply) to protect my TrueNAS from possible data loss in case of a power failure. What I've been missing throughout that time are the monitoring and management tools to shut down everything gracefully when the battery of the UPS runs low. In the event of a power outage lasting longer than 30 minutes, the battery would run out of juice. Everything attached to the UPS would be powered off immediately, and data loss might occur. Luckily I live in an area where power outages rarely happen. I also have backups I could restore if my TrueNAS data gets corrupted. Still, doing this right and configuring [Network UPS Tools (NUT)](https://networkupstools.org/) has been on my to-do list for way too long.

<!--more-->

## Requirements

My server rack contains the following hardware:

1. [OPNsense](https://opnsense.org/) firewall and router
2. TrueNAS storage and jail apps
3. [Mikrotik CRS328-24P-4S+RM](https://mikrotik.com/product/crs328_24p_4s_rm#fndtn-downloads) core switch running [SwOS](https://help.mikrotik.com/docs/pages/viewpage.action?pageId=76415036)
4. [CyberPower PR750ERT2U](https://www.cyberpower.com/global/en/product/sku/pr750ert2u) UPS
5. ISP modem

Of the above, only OPNsense and TrueNAS are susceptible to data corruption in case of power loss. If a shutdown due to low battery power is required, I want to shut down TrueNAS first. After, OPNsense will shut down and turn off the UPS. To achieve this with NUT, I configure OPNsense as NUT **master** and TrueNAS as NUT **slave**. Only one master must exist. However, multiple slaves may exist.

## CyberPower PR750ERT2U

This UPS ticks a lot of boxes for me:

- HID-compliant USB port to connect OPNsense
- Fanless operation during utility mode. It isn't apparent from the specifications, but the CyberPower support kindly provided me with this information
- 750VA capacity allowing me to add another server)
- Pure sine wave output
- Line-interactive topology
- Energy-saving capabilities

Initially, I looked at similar used APC units on eBay. At the time, I couldn't find any good deals on eBay meeting all of the above requirements. I then found the CyberPower unit selling new for under $400 and got one. I've been a happy CyberPower customer ever since. To be clear, I'm don't affiliate with CyberPower.

## NUT on OPNsense

Only a few steps are required to configure NUT for OPNsense.

### Service Configuration

1. Install the `os-nut` plugin under {{< breadcrumb "System" "Firmware" "Plugins" >}}
2. Refresh the browser and enable the USB HID driver under {{< breadcrumb "Services" "Nut" "Configuration" "UPS Type" "USBHID-Driver" >}}
3. Reboot OPNsense
4. Connect the UPS to OPNsense via USB
5. Set the **Monitor Password** and **Admin Password** for good measure under {{< breadcrumb "Services" "Nut" "Configuration" "Nut Account Settings" >}}
6. Set a **Name** and **Enable Nut** under {{< breadcrumb "Services" "Nut" "Configuration" "General Settings" "General Nut Settings" >}}

{{< breadcrumb "Services" "Nut" "Diagnostics" >}} should output something like the following:

```text
battery.charge: 100
battery.charge.low: 0
battery.charge.warning: 35
battery.mfr.date: CPS
battery.runtime: 5020
battery.runtime.low: 300
battery.type: PbAcid
battery.voltage: 0.6
battery.voltage.nominal: 22
device.mfr: CPS
device.model: PR750ERT2U
device.serial: XXXXXXXXXXXX
device.type: ups
driver.name: usbhid-ups
driver.parameter.pollfreq: 30
driver.parameter.pollinterval: 2
driver.parameter.port: auto
driver.parameter.synchronous: no
driver.version: 2.7.4
driver.version.data: CyberPower HID 0.4
driver.version.internal: 0.41
input.voltage: 235.0
input.voltage.nominal: 230
output.voltage: 253.0
ups.beeper.status: enabled
ups.delay.shutdown: 20
ups.delay.start: 30
ups.load: 17
ups.mfr: CPS
ups.model: PR750ERT2U
ups.productid: 0601
ups.realpower.nominal: 750
ups.serial: XXXXXXXXXXXX
ups.status: OL
ups.test.result: No test initiated
ups.timer.shutdown: 0
ups.timer.start: 0
ups.vendorid: 0764
```

### Port Forward

To allow TrueNAS to communicate with the NUT server on OPNsense, we need to add the following port forward under {{< breadcrumb "Firewall" "NAT" "Port Forward" >}}. I assume that TrueNAS is connected to the `LAN` interface.

|                        |                                    |
| ---------------------- | ---------------------------------- |
| Interface              | `LAN`                              |
| Source                 | `LAN net`                          |
| Destination            | `LAN address`                      |
| Destination port range | from `3493` to `3493`              |
| Redirect target IP     | `127.0.0.1`                        |
| Redirect target port   | `3493`                             |
| Description            | `Redirect NUT traffic to OPNsense` |

After this you should be able to SSH into your TrueNAS and run the following to retrieve the UPS status from OPNsense:

```shell
upsc ups@192.168.1.1
```

## TrueNAS

Navigate to {{< breadcrumb "Services" "UPS" "Configure" >}}.
