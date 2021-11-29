---
title: "Configure NUT for OPNsense and TrueNAS with the CyberPower PR750ERT2U UPS"
date: 2021-11-28T03:50:22+01:00
comments: true
toc: false
cover:
  src: cover.jpg
toc: true
tags:
  - CyberPower
  - Homelab
  - NUT
  - OPNsense
  - Self-host
  - TrueNAS
  - UPS
aliases:
  - /posts/configure-nut-for-opnsense-and-truenas-with-the-cyberpower-pr750ert2u-ups
---

For storage in my homelab, I use [TrueNAS](https://www.truenas.com/). Additionally, I run a couple of apps on top of it as [jails](https://docs.freebsd.org/en/books/handbook/jails/). For over a year, I've been using an [uninterruptible power supply (UPS)](https://en.wikipedia.org/wiki/Uninterruptible_power_supply) to protect my TrueNAS from possible data loss in case of a power failure. What I've been missing throughout that time are the monitoring and management tools to shut down everything gracefully when the battery of the UPS runs low. In the event of a power outage lasting longer than 30 minutes, the battery would run out of juice. Everything attached to the UPS would be powered off immediately, and data loss might occur. Luckily I live in an area where power outages rarely happen. I also have backups I could restore if my TrueNAS data gets corrupted. Still, doing this right and configuring [Network UPS Tools (NUT)](https://networkupstools.org/) to orchestrate shutdowns has been on my to-do list for way too long. It's time to tackle the issue!

<!--more-->

As always, I'll only mention settings deviating from the defaults.

## Requirements

My server rack contains the following hardware.

1. [OPNsense](https://opnsense.org/) firewall and router
2. TrueNAS storage and jail apps
3. [Mikrotik CRS328-24P-4S+RM](https://mikrotik.com/product/crs328_24p_4s_rm#fndtn-downloads) core switch running [SwOS](https://help.mikrotik.com/docs/pages/viewpage.action?pageId=76415036)
4. [CyberPower PR750ERT2U](https://www.cyberpower.com/global/en/product/sku/pr750ert2u) UPS
5. ISP modem

Of the above, only OPNsense and TrueNAS are susceptible to data corruption in case of power loss. If a shutdown due to low battery power is required, I want to shut down TrueNAS first. After, OPNsense shuts down. To achieve this with NUT, I configure OPNsense as NUT **master** and TrueNAS as NUT **slave**. Only one master must exist. However, multiple slaves may exist.

## CyberPower PR750ERT2U

The CyberPower PR750ERT2U has about everything I'd ever want from a UPS.

- HID-compliant USB port to connect OPNsense
- Fanless operation in utility mode. It isn't apparent from the specifications, but the CyberPower support kindly provided me with this information
- 750VA capacity allowing me to add another server
- Pure sine wave output
- Line-interactive topology
- Energy-saving capabilities

Initially, I looked at similar used APC units on eBay. At the time, I couldn't find any good deals on eBay meeting all of the requirements above. I then found the CyberPower unit selling new for under $400 and grabbed one. I've been a happy CyberPower customer ever since. To be clear, I don't affiliate with CyberPower.

## NUT on OPNsense

### NUT Service Configuration

To configure the NUT service on OPNsense, we do the following.

1. Install the `os-nut` plugin under {{< breadcrumb "System" "Firmware" "Plugins" >}}
2. Refresh the browser and enable the USB HID driver under {{< breadcrumb "Services" "Nut" "Configuration" "UPS Type" "USBHID-Driver" >}}
3. Reboot OPNsense
4. Connect the UPS to OPNsense via USB
5. Set the **Monitor Password** and **Admin Password** under {{< breadcrumb "Services" "Nut" "Configuration" "Nut Account Settings" >}}
6. Set a **Name** (e.g., `cyberpower`) and **Enable Nut** under {{< breadcrumb "Services" "Nut" "Configuration" "General Settings" "General Nut Settings" >}}

We should see the data of the UPS under {{< breadcrumb "Services" "Nut" "Diagnostics" >}}.

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

To allow TrueNAS to communicate with the NUT server on OPNsense, we add the following port forward under {{< breadcrumb "Firewall" "NAT" "Port Forward" >}}. I assume that TrueNAS is part of the `LAN` network.

|                        |                                    |
| ---------------------- | ---------------------------------- |
| Interface              | `LAN`                              |
| Source                 | TrueNAS IP                         |
| Destination            | `LAN address`                      |
| Destination port range | from `3493` to `3493`              |
| Redirect target IP     | `127.0.0.1`                        |
| Redirect target port   | `3493`                             |
| Description            | `Redirect NUT traffic to OPNsense` |

To see if it works, we SSH into TrueNAS and run the following to retrieve the UPS status from OPNsense.

```shell
upsc cyberpower@192.168.1.1
```

## TrueNAS

![Screenshot of TrueNAS NUT settings](truenas-nut.png)

Set the following options under {{< breadcrumb "Services" "UPS" "Configure" >}}.

| General Options |                                                             |
| --------------- | ----------------------------------------------------------- |
| Identifier      | The **Name** we earlier set in OPNsense, e.g., `cyberpower` |
| UPS Mode        | `Slave`                                                     |
| Remote Host     | `<IP or hostname of OPNsense>`                              |
| Remote Port     | `3493`                                                      |
| Identifier      | `auto`                                                      |

| Shutdown         |                           |
| ---------------- | ------------------------- |
| Shutdown Mode    | `UPS reaches low battery` |
| Shutdown Command | `/sbin/poweroff`          |

| Monitor          |                                                     |
| ---------------- | --------------------------------------------------- |
| Monitor User     | `monuser`                                           |
| Monitor Password | The **Monitor Password** we earlier set in OPNsense |

Clicking **Save** returns us to the **Services** menu. We need to enable the **UPS** service and check **Start Automatically** to start the service at boot time.

![Screenshot of TrueNAS Services settings](truenas-services.png)

## Test

We could disconnect the UPS from power and let the battery drain until the UPS reaches _low battery_ status. At some point, it's probably a good idea to do. But repeatedly doing that unnecessarily wears down the battery. Another way is to run the **F**orce **S**hut **D**own command. SSH into the NUT master (OPNsense) and run the following.

```shell
upsmon -c fsd
```

If we configured everything correctly, TrueNAS and OPNsense shut down.
