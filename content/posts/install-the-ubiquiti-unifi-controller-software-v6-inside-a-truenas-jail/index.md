---
title: "Install the Ubiquiti Unifi Controller Software v6 Inside a TrueNAS Jail"
date: 2021-11-04T23:15:27+01:00
cover:
  src: "cover.png"
hideReadMore: true
comments: true
tags:
  - FreeBSD
  - Jails
  - TrueNAS
  - Ubiquiti
  - UniFi
---

To manage [Ubiquiti UniFi devices](https://www.ui.com/wi-fi), a UniFi controller is required. Over a year ago, I initially installed the controller software inside a Ubuntu VirtualBox VM. Now that version 6 of the UniFi controller software is released, it's time to upgrade. I decided to reinstall the controller inside a TrueNAS jail instead of a VirtualBox VM. When searching the interwebs, I only found lots of outdated instructions. It turns out that it's very straightforward, so here are my quick notes on how to do it.

<!--more-->

I tested the following on TrueNAS version `12.0-U6`.

## Install UniFi

Connect to a shell via Web GUI or SSH and fetch the latest FreeBSD release available. At the time of this writing, it was `12.2-RELEASE`.

<!-- markdownlint-disable MD033 -->
<pre class="command-line language-bash" data-user="root" data-host="truenas">
  <code>
    iocage fetch
  </code>
</pre>
<!-- markdownlint-enable MD033 -->

Connect to TrueNAS via shell and create the jail.

<!-- markdownlint-disable MD033 -->
<pre class="command-line language-bash" data-user="root" data-host="truenas">
  <code>
    iocage create --name unifi --release 12.2-RELEASE dhcp=1 boot=1
  </code>
</pre>
<!-- markdownlint-enable MD033 -->

I use DHCP reservations to manage my server IPs. Setting `dhcp=1` also sets `vnet=1` and `bpf=1`. Network configuration is out of scope for this guide. Please consult the `iocage` manual (`man iocage`) or the [TrueNAS jails documentation](https://www.truenas.com/docs/core/applications/jails/) for more info. Setting `boot=1` enables auto-start at boot time.

Connect to the jail.

<!-- markdownlint-disable MD033 -->
<pre class="command-line language-bash" data-user="root" data-host="truenas">
  <code>
    sudo iocage console unifi
  </code>
</pre>
<!-- markdownlint-enable MD033 -->

Once connected, run the following commands.

<!-- markdownlint-disable MD033 -->
<pre class="command-line language-bash" data-user="root" data-host="unifi">
  <code>
    pkg update && pkg upgrade -y  # install updates
    pkg install unifi6            # install unifi
    sysrc unifi_enable=YES        # auto-start at boot
    service unifi start           # start unifi
  </code>
</pre>
<!-- markdownlint-enable MD033 -->

Connect to the UniFi controller at `https://<jail IP>:8443`.

## Updating

Updating requires very few steps. **Please backup the jail before updating** to be able to roll back if something goes wrong.

Connect to the jail.

<!-- markdownlint-disable MD033 -->
<pre class="command-line language-bash" data-user="root" data-host="truenas">
  <code>
    sudo iocage console unifi
  </code>
</pre>
<!-- markdownlint-enable MD033 -->

Once connected, run the following commands.

<!-- markdownlint-disable MD033 -->
<pre class="command-line language-bash" data-user="root" data-host="unifi">
  <code>
    service unifi stop            # stop unifi
    pkg update && pkg upgrade -y  # install updates
    service unifi start           # start unifi
  </code>
</pre>
<!-- markdownlint-enable MD033 -->

And that's it!
