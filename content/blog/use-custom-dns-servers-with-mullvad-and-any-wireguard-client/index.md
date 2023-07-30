---
title: "Use Custom DNS Servers With Mullvad And Any WireGuard Client"
date: 2021-10-31T08:19:52+01:00
cover:
  src: cover.jpg
comments: true
socialShare: true
tags:
  - DNS
  - Mullvad
  - Network
  - VPN
  - WireGuard
aliases:
  - /posts/use-custom-dns-servers-with-mullvad-and-any-wireguard-client
---

I've been using [Mullvad VPN](https://mullvad.net/) for a while now but only
ever used it with the official client on my workstation. I use DNS extensively
in my home network, so as soon as I activate Mullvad, I can't resolve DNS names
locally. Of course, this is by design and expected. I own an
[OPNsense](https://opnsense.org/) appliance, so the natural solution is to move
the tunnel there.

## TL;DR

Use the following shell command to request an IP with no DNS hijacking:

```shell
# Authenticate to Mullvad and store the returned access token
access_token=$( \
  curl -X 'POST' 'https://api.mullvad.net/auth/v1/token' \
    -H 'accept: application/json' -H 'content-type: application/json' \
    -d '{ "account_number": "YOUR MULLVAD ACCOUNT NUMBER" }' \
  | jq -r .access_token)

# Create a new Mullvad Device with DNS hijacking disabled
curl -X POST https://api.mullvad.net/accounts/v1/devices \
  -H "Authorization: Bearer $access_token" -H 'content-type: application/json' \
  -d '{"pubkey":"YOUR WIREGUARD PUBLIC KEY","hijack_dns":false}'
```

## Mullvad Hijacks DNS Queries Over WireGuard Tunnels

Instead of using the OpenVPN protocol, I decided to go with the latest and
greatest: [WireGuard](https://www.wireguard.com/). OPNsense is a fork of FreeBSD
but lacks a kernel implementation of WireGuard, requiring a plugin. It's good
enough for me to try, and I hope WireGuard will be natively supported soon.

During my research on how to best configure this, there seemed to be the
[caveat of Mullvad hijacking DNS traffic going through WireGuard tunnels](https://forum.netgate.com/topic/166804/unbound-dns-resolver-through-wireguard-tunnel-mullvad-vpn),
redirecting it to their DNS servers. It decreases the likelihood of DNS leaks,
but power users like you and I might not want that. What if I want to query DNS
root servers through the VPN tunnel because I use my own DNS resolver? Mullvad
hijacking DNS queries would make my DNS resolver trip up.

What a bummer, right? Looking at the
[Mullvad FAQ](https://mullvad.net/en/help/faq/), it seemed the only solution was
to resort to OpenVPN:

> Ports 1400 UDP and 1401 TCP do not have DNS hijacking enabled, which might
> work better for pfSense users

[But Mullvad launched support for custom DNS servers on the Mullvad VPN app back in April 2021](https://mullvad.net/ar/blog/2021/4/15/support-custom-dns-servers-launched/).
It also works for WireGuard, so what's the secret?

## Reverse-engineering the Mullvad App

Searching through the docs, I found the
[WireGuard on a router article explaining how to get an IP to use with Mullvad via API](https://mullvad.net/es/help/running-wireguard-router/)
(note that
[this snippet is outdated](https://github.com/mullvad/mullvadvpn-app/issues/473#issuecomment-1500461798)):

```shell
curl https://api.mullvad.net/wg/ -d account=YOURMULLVADACCOUNTNUMBER --data-urlencode pubkey=YOURPUBLICKEY
```

Next, let's look at how the app requests IPs.
[Fortunately it's open-source and available on GitHub](https://github.com/mullvad/mullvadvpn-app/issues/473#issuecomment-852064948),
so the only reverse-engineering we're going to be doing is reading some code. It
turns out that
[the app uses the `https://api.mullvad.net/app/v1/devices` endpoint to create a _Mullvad Device_ with DNS hijacking disabled](https://github.com/mullvad/mullvadvpn-app/blob/f300cda8767f3e07b62471677c5ea729d8a0dff0/mullvad-api/src/device.rs#L58-L66).

## Testing Both APIs

For testing, we'll be using the
[official WireGuard client](https://www.wireguard.com/install/). Let's open the
client, click `Add empty tunnel...`, and give it a name:

![Screenshot of WireGuard client "Add Tunnel" context menu](wireguard-add-tunnel-menu.png)

The tunnel will initially look like this:

![Screenshot of initial tunnel configuration](wireguard-initial-tunnel-configuration.png)

Copy the public key and execute the following to request our Mullvad IPs
(because the snippet is outdated, this might not work in the future):

```shell
curl https://api.mullvad.net/wg/ -d account=YOURMULLVADACCOUNTNUMBER --data-urlencode pubkey=YOURPUBLICKEY
```

The response will return an IPv4 and IPv6 address. Add the following to the
configuration file:

```text
[Interface]
PrivateKey = <PRIVATE KEY>
Address = <IPv4 ADRESS>
DNS = 9.9.9.9

[Peer]
PublicKey = bmy9vGzMqc0yS3IiMMyOONyXRwPCMiyhR/bnNQ2LsCE=
AllowedIPs = 0.0.0.0/0
Endpoint = 31.7.59.250:51820
```

We use the `ch2-wireguard` Mullvad server as peer and
[Quad9](https://quad9.org/) as DNS server.

Let's activate the tunnel and browse to
[Mullvad's connection check](https://mullvad.net/en/check):

![Screenshot of Mullvad connection check without leak](mullvad-connection-check-no-leak.png)

As expected, the Quad9 DNS server is not leaking through because Mullvad hijacks
our DNS requests and redirects them to their DNS servers.

Next, we use the API the app uses to request the Mullvad IPs. Before we can do
this, we create a new public key with the WireGuard client because Mullvad
doesn't allow using the same public key more than once.

Then we authenticate to Mullvad and store the returned access token:

```shell
access_token=$( \
  curl -X 'POST' 'https://api.mullvad.net/auth/v1/token' \
    -H 'accept: application/json' -H 'content-type: application/json' \
    -d '{ "account_number": "YOUR MULLVAD ACCOUNT NUMBER" }' \
  | jq -r .access_token)
```

Next, we create a new _Mullvad Device_ with DNS hijacking disabled:

```shell
curl -X POST https://api.mullvad.net/accounts/v1/devices \
  -H "Authorization: Bearer $access_token" -H 'content-type: application/json' \
  -d '{"pubkey":"YOUR NEW WIREGUARD PUBLIC KEY","hijack_dns":false}'
```

Next, we replace the IP in `Address` field of the WireGuard config with the new
IP we received. Then we activate the tunnel and visit
[Mullvad's connection check](https://mullvad.net/en/check):

![Screenshot of Mullvad connection check with leak](mullvad-connection-check-leak.png)

Hooray, the Quad9 DNS servers leak through, so Mullvad is not hijacking our DNS
traffic for this tunnel!
