---
title: "Use Mullvad Port Forwarding to Connect to Your OPNsense Home Network with WireGuard"
date: 2021-11-28T21:18:30+01:00
comments: true
socialShare: true
toc: false
cover:
  src: cover.png
tags:
  - Firewall
  - Mullvad
  - Network
  - OPNsense
  - VPN
  - WireGuard
aliases:
  - /posts/use-mullvad-port-forwarding-to-connect-to-your-opnsense-home-network-with-wireguard
---

In this quick guide, I'll show you how to use [Mullvad](https://mullvad.net/) port forwarding and [OPNsense](https://opnsense.org/) to create a [WireGuard](https://www.wireguard.com/) VPN "tunnel-inside-a-tunnel" configuration, to be able to connect to your home network from the outside. It's pretty nifty because you won't have to expose your public IP address. This time, I'll give you more of a high-level overview and reference the relevant documentation instead of a detailed step-by-step guide.

<!--more-->

## "Outer" WireGuard Tunnel

I won't cover the configuration steps of the "outer" tunnel leading from your OPNsense router to a Mullvad VPN server in this post. [I already cover this topic in-depth in my OPNsense baseline guide.](/posts/opnsense-baseline-guide-with-vpn-guest-and-vlan-support#wireguard-vpn-with-mullvad)

## "Inner" WireGuard Tunnel

Configuring the "inner" tunnel is also covered by the [WireGuard Road Warrior Setup guide from the official OPNsense documentation](https://docs.opnsense.org/manual/how-tos/wireguard-client.html). Make sure to set it up and verify it's working before you continue.

## Why Mullvad Port Forwarding?

While you set up the inner tunnel, you might have noticed that the external road warrior clients need to know the public IP of your OPNsense host. If you have a residential internet subscription it's likely your ISP provides you with a _dynamic_ IP address. So every time your public IP address changes, you need to update the VPN client configurations. A typical solution for this issue is to use [Dynamic DNS (DDNS)](https://en.wikipedia.org/wiki/Dynamic_DNS). You can configure a DDNS client to monitor your dynamic IP address. And as soon it changes, it associates the IP address with a public DNS record, e.g., `vpn.example.com`. You can then use the DNS hostname in your WireGuard client configurations instead of an explicit IP. But what if you don't want to expose your "real" IP address to the public? Cloudflare proxying comes to mind, [but it operates on layer 7](https://developers.cloudflare.com/load-balancing/understand-basics/proxy-modes), so it doesn't work with WireGuard.

Mullvad port forwarding to the rescue! It allows us to forward any traffic through the "outer" tunnel. [Read the official port forwarding with Mullvad VPN guide to find out how to configure your ports.](https://mullvad.net/en/help/port-forwarding-and-mullvad/)

## Configure OPNsense

Only a few steps are needed to configure OPNsense. In the following, I'll assume the following.

- The interface of the outer WireGuard tunnel is named `WAN_VPN1`
- The randomly generated Mullvad port number is `61234`
- The WireGuard local peer for external clients listens to port `51888`

First, we allow inbound traffic for the Mullvad port on the WireGuard interface of the outer tunnel. Navigate to {{< breadcrumb "Firewall" "Rules" "WAN_VPN1" >}} and add the following rule.

![Screenshot of firewall rule](rule.png)

|                        |                              |
| ---------------------- | ---------------------------- |
| Action                 | `Pass`                       |
| Interface              | `WAN_VPN1`                   |
| Protocol               | `UDP`                        |
| Source                 | `any`                        |
| Destination            | `WAN_VPN1 address`           |
| Destination port range | from `61234` to `61234`      |
| Description            | `Allow Mullvad port forward` |

Secondly, we redirect the traffic to the WireGuard local peer for external clients. Navigate to {{< breadcrumb "Firewall" "NAT" "Port Forward" >}} and add the following rule.

![Screenshot of firewall port-forward](port-forward.png)

|                        |                                              |
| ---------------------- | -------------------------------------------- |
| Interface              | `WAN_VPN1`                                   |
| Protocol               | `UDP`                                        |
| Source                 | `any`                                        |
| Destination            | `WAN_VPN1 address`                           |
| Destination port range | from `61234` to `61234`                      |
| Redirect target IP     | `127.0.0.1`                                  |
| Redirect target port   | `51888`                                      |
| Description            | `Redirect Mullvad port forward to WireGuard` |

Optionally, you can configure a public DNS record pointing to the exit IP of your Mullvad VPN tunnel.

And that's it for this little guide. Let me know what you think on Twitter or in the comments below!
