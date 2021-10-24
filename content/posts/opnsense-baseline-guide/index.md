---
title: "OPNsense Baseline Guide"
date: 2021-10-23T23:37:35+02:00
cover: "img/cover.svg"
useRelativeCover: true
draft: true
hideReadMore: true
comments: true
tags:
  - mullvad
  - networking
  - opnsense
  - routing
  - vpn
  - wireguard
---

## Interface Creation And Configuration

### Create VLANs

We need to identify the physical parent interface which transfers all VLAN traffic. Typically an assignment to the logical `LAN` interface already exists, connecting your switch to the OPNsense router. In my case that's `igb0`.

Navigate to `Interfaces` &rarr; `Other Types` &rarr; `VLAN`.

#### Management VLAN

1. Click `+`:
2. `Parent Interface`: the parent interface, e.g. `igb0`
3. `VLAN tag`: `10`
4. `Description`: `VLAN10_MANAGE`
5. Click `Save`

#### VPN VLAN

1. Click `+`:
2. `Parent Interface`: the parent interface, e.g. `igb0`
3. `VLAN tag`: `20`
4. `Description`: `VLAN20_VPN`
5. Click `Save`

#### Clear VLAN

1. Click `+`:
2. `Parent Interface`: the parent interface, e.g. `igb0`
3. `VLAN tag`: `30`
4. `Description`: `VLAN30_CLEAR`
5. Click `Save`

#### Guest VLAN

1. Click `+`:
2. `Parent Interface`: the parent interface, e.g. `igb0`
3. `VLAN tag`: `40`
4. `Description`: `VLAN40_GUEST`
5. Click `Save`

### Create Logical Interfaces

For each VLAN, create a logical interface. Navigate to `Interfaces` &rarr; `Assignments`

1. Select `vlan 10`, add the description `VLAN10_MANAGE`, and click `+`
2. Select `vlan 20`, add the description `VLAN20_VPN`, and click `+`
3. Select `vlan 30`, add the description `VLAN30_CLEAR`, and click `+`
4. Select `vlan 40`, add the description `VLAN40_GUEST`, and click `+`
5. Click `Save`

### Configure Interface IP Addresses

To easier remember which IP range belongs to which VLAN, a common approach is to match an octet of the IP range with the VLAN ID. E.g., the VLAN with the ID **10** would have a range of 192.168.**10**.0/24.

![Screenshot of VLAN logical interface configuration](img/configure-interface-ip-addresses.png)

Navigate to `Interfaces` &rarr; `Assignments`.

#### VLAN10_MANAGE Interface

1. Navigate to `VLAN10_MANAGE`
2. `Enable Interface`
3. `IPv4 Configuration Type`: `Static IPv4`
4. `IPv4 Address`: `192.168.10.1/24`
5. Click `Save` and `Apply changes` when prompted

#### VLAN20_VPN Interface

1. Navigate to `VLAN20_VPN`
2. `Enable Interface`
3. `IPv4 Configuration Type`: `Static IPv4`
4. `IPv4 Address`: `192.168.20.1/24`
5. Click `Save` and `Apply changes` when prompted

#### VLAN30_CLEAR Interface

1. Navigate to `VLAN30_CLEAR`
2. `Enable Interface`
3. `IPv4 Configuration Type`: `Static IPv4`
4. `IPv4 Address`: `192.168.30.1/24`
5. Click `Save` and `Apply changes` when prompted

#### VLAN40_GUEST Interface

1. Navigate to `VLAN40_GUEST`
2. `Enable Interface`
3. `IPv4 Configuration Type`: `Static IPv4`
4. `IPv4 Address`: `192.168.40.1/24`
5. Click `Save` and `Apply changes` when prompted

### Configure Interface DHCP

Depending on your needs, you might have to adjust the DHCP ranges. For my homelab I chose `x.x.x.100-199` for dynamic addresses and `x.x.x.10.10-99` for static addresses.

![Screenshot of VLAN interface DHCP configuration](img/vlan-interface-dhcp-configuration.png)

Navigate to `Services` &rarr; `DHCPv4`.

#### VLAN10_MANAGE DHCP

1. Select `VLAN10_MANAGE`
2. Check `Enable DHCP server on the VLAN10_MANAGE interface`
3. Configure the `Range` from `192.168.10.100` to `192.168.10.199`
4. Click `Save`

#### VLAN20_VPN DHCP

1. Select `VLAN20_VPN`
2. Check `Enable DHCP server on the VLAN20_VPN interface`
3. Configure the `Range` from `192.168.20.100` to `192.168.20.199`
4. Click `Save`

#### VLAN30_CLEAR DHCP

1. Select `VLAN30_CLEAR`
2. Check `Enable DHCP server on the VLAN30_CLEAR interface`
3. Configure the `Range` from `192.168.30.100` to `192.168.30.199`
4. Click `Save`

#### VLAN40_GUEST DHCP

1. Select `VLAN40_GUEST`
2. Check `Enable DHCP server on the VLAN40_GUEST interface`
3. Configure the `Range` from `192.168.40.100` to `192.168.40.199`
4. Click `Save`

## VPN Configuration

The past couple of years, I've been using [Mullvad](https://mullvad.net/) as VPN provider and I'm a very happy customer. Back when _That One Privacy Site_ was still a thing, Mullvad earned a very good review which was when I decided to try it out. I have never looked back since. No personally idenfiable information is required to register, and paying cash via mail works perfectly.

I decided to go with the [WireGuard Road Warrior](https://docs.opnsense.org/manual/how-tos/wireguard-client.html) setup since I think WireGuard is the VPN protocol of the future. For more detailed step, check the [official OPNsense documentation on how to setup WireGuard with Mullvad](https://docs.opnsense.org/manual/how-tos/wireguard-client-mullvad.html). We'll also configure failover with two separate tunnels, in case one goes down.

WireGuard is not (yet) a native part of FreeBSD, so you must install it as plugin. Navigate to `System` &rarr; `Firmware` &rarr; `Plugins` and install `os-wireguard`.

After refreshing the browser, navigate to `VPN` &rarr; `WireGuard`.

### Setup the Endpoint (Client Peer)

![Screenshot of WireGuard Endpoint configuration](img/wireguard-endpoint-configuration.png)

Select one or more WireGuard servers from [Mullvad's server list](https://mullvad.net/en/servers/) and take note of their name and public key.

1. Select the `Endpoints` tab
2. Click `Add`
3. `Name`: e.g., `mullvad-ch5-wireguard`
4. `Public Key`: e.g, `/iivwlyqWqxQ0BVWmJRhcXIFdJeo0WbHQ/hZwuXaN3g=`
5. `Allowed IPs`: `0.0.0.0/0`, `::/0`
6. `Endpoint Address`: `<server name>.mullvad.net` e.g., `ch5-wireguard.mullvad.net`
7. `Endpoint Port`: `51820`
8. Click `Save`

Repeat the steps above and add another server, e.g., `ch6-wireguard`. If you want to mitigate the risk against DNS poisoning, enter the resolved IP as `Endpoint Address` instead of the hostname.

### Setup the Local Peer (Server)

![Screenshot of WireGuard Local Peer configuration](img/wireguard-local-peer-configuration.png)

1. Select the `Local` tab
2. Click `Add`
3. Enable `advanced mode`
4. `Name`: e.g., `mullvad0`
5. `Listen Port`: `51820`
6. `DNS Server`: `193.138.218.74` (public Mullvad DNS)
7. `Tunnel Address`: leave empty for now
8. `Peers`: e.g., `ch5-wireguard`
9. Click `Save`
10. Click `Edit` and copy the generated `Public Key`
11. Check `Disable Routes`

The following command will output an IPv4 and IPv6 IP address:

```shell
curl -sSL https://api.mullvad.net/wg/ -d account=<account number> --data-urlencode pubkey=<public key>
```

Replace the `Tunnel Address` with these IP addresses.

Repeat the steps above to create a second local peer named `mullvad1`. Don't forget to use a different `Listen Port`, e.g. `51821`.

When you're done, select the `General` tab and check `Enable WireGuard`. Shortly after you should see two handshakes for the `wg0` and `wg1` tunnels on the `Handshakes` tab.

### WireGuard Interface Assignments and Addressing

Next, we assign the wireguard tunnels to interfaces.

![Screenshot of WireGuard interface configuration](img/wireguard-interface-configuration.png)

Navigate to `Interfaces` &rarr; `Assignments`.

1. Select `wg0`, add the description `VPN0_WAN`, and click `+`
2. Select `wg1`, add the description `VPN1_WAN`, and click `+`

#### VPN0 Interface

1. Navigate to `VPN0_WAN`
2. `Enable Interface`
3. `IPv4 Configuration Type`: `Static IPv4`
4. `IPv4 Address`: IPv4 address of the `mullvad0` local peer
5. `IPv4 Upstream Gateway`
   1. Click `+`
   2. `Gateway Name`: `VPN0_WAN`
   3. `Gateway IPv4`: IPv4 address of the `mullvad0` local peer
   4. Click `Save` and select the new gateway
6. `IPv6 Configuration Type`: `Static IPv6`
7. `IPv6 Address`: IPv6 address of the `mullvad0` local peer
8. `IPv6 Upstream Gateway`
   1. Click `+`
   2. `Gateway Name`: `VPN0_WAN6`
   3. `Gateway IPv6`: IPv6 address of the `mullvad0` local peer
   4. Click `Save` and select the new gateway
9. Click `Save` and `Apply changes`

#### VPN1 Interface

1. Navigate to `VPN1_WAN`
2. `Enable Interface`
3. `IPv4 Configuration Type`: `Static IPv4`
4. `IPv4 Address`: IPv4 address of the `mullvad1` local peer
5. `IPv4 Upstream Gateway`
   1. Click `+`
   2. `Gateway Name`: `VPN1_WAN`
   3. `Gateway IPv4`: IPv4 address of the `mullvad1` local peer
   4. Click `Save` and select the new gateway
6. `IPv6 Configuration Type`: `Static IPv6`
7. `IPv6 Address`: IPv6 address of the `mullvad1` local peer
8. `IPv6 Upstream Gateway`
   1. Click `+`
   2. `Gateway Name`: `VPN1_WAN6`
   3. `Gateway IPv6`: IPv6 address of the `mullvad1` local peer
   4. Click `Save` and select the new gateway
9. Click `Save` and `Apply changes`

### Configure Gateway Monitoring

Since the gateway address uses the local address of the tunnel, the gateway will always be online. Hence, failover will never occur, even if a WireGuard endpoint is unreachable. To mitigate this issue, configure a remote monitor IP for the gateway. It's best to use highly reliable services like Cloudflare for this.

Navigate to `System` &rarr; `Gateways` &rarr; `Single`

1. `Edit` the `VPN0_WAN` gateway, enter `1.1.1.1` as `Monitor IP`, and click `Save`
2. `Edit` the `VPN1_WAN` gateway, enter `1.0.0.1` as `Monitor IP`, and click `Save`
3. `Edit` the `VPN0_WAN6` gateway, enter `2606:4700:4700::1111` as `Monitor IP`, and click `Save`
4. `Edit` the `VPN1_WAN6` gateway, enter `2606:4700:4700::1001` as `Monitor IP`, and click `Save`
5. Click `Apply changes`

### Configure Gateway Failover

Navigate to `System` &rarr; `Gateways` &rarr; `Group`

#### IPv4 Gateway Failover

1. Click `+`
2. `Group Name`: `VPN_GROUP`
3. `VPN0_WAN`: `Tier 1`
4. `VPN1_WAN`: `Tier 1`
5. `Trigger Level`: `Packet Loss or High Latency`
6. Click `Save` and `Apply changes`

#### IPv6 Gateway Failover

1. Click `+`
2. `Group Name`: `VPN_GROUP6`
3. `VPN0_WAN6`: `Tier 1`
4. `VPN1_WAN6`: `Tier 1`
5. `Trigger Level`: `Packet Loss or High Latency`
6. Click `Save` and `Apply changes`

## DNS

We make use of three DNS resolvers to provide name resolution across the network:

- **Cloudflare DNS** for the guest network, supporting WAN failover
- **DNS Forwarder (Dnsmasq DNS)** for the Clear network. Unbound handles local lookups and Quad9 handles external lookups with some basic privacy
- **DNS Resolver (Unbound)** will be the authoritative name server for the private `internal.example.com` domain, so names as part of that domain are not forwarded to external DNS preventing information leakage

The design of the system:

- Support multiple gateways
- Enable local device lookups for all non-guest interfaces
- Prevent information leaking to the ISP
- Prevent IP leaks by using VPN
- Keep DNS queries within the VPN tunnel from secured networks
- Optimize local performance with DNS lookup caching

This requires that local devices only use OPNsense as DNS server. For cached and local names lookup results from Unbound, unknown names will be resolved externally with Quad9 or the Mullvad DNS server through the VPN tunnel. Unbound will only use the VPN_WAN interface, so DNS lookups won't be possible, which is why Clear and Guest networks serve as backups.

### Configure DNS Resolver (Unbound)

Navigate `Services` &rarr; `Unbound DNS` &rarr; `General`

- `Show advanced options`
- `Enable Unbound`
- `Listen Port`: `53`
- `Network Interfaces`: `LAN`, `VLAN10_MANAGE`, `VLAN20_VPN`
- Check `Enable DNSSec Support`
- Check `Register DHCP static mappings` to make using DHCP reservations more convenient
- `Local Zone Type`: `static`
- `Outgoing Network Interfaces`: `VPN0_WAN`, `VPN1_WAN`

Navigate to `Services` &rarr; `Unbound DNS` &rarr; `Advanced`

- Check `Prefetch Support`
- Check `Prefetch DNS Key Support`
- Check `Harden DNSSEC data`

We also need to tell Unbound not to query external name servers for the private `internal.example.com` domain. We need to add a custom [SOA record](https://www.cloudflare.com/learning/dns/dns-records/dns-soa-record/) overriding the authoritative name server. We'll make use of [templates](https://docs.opnsense.org/development/backend/templates.html) to for [advanced Unbound configuration](https://docs.opnsense.org/manual/unbound.html#advanced-configurations). To configure them, you need to access OPNsense through either SSH or the serial console.

Add a `+TARGETS` file by running `vi /usr/local/opnsense/service/templates/sampleuser/Unbound/+TARGETS` with the content:

```text
private_domains.conf:/usr/local/etc/unbound.opnsense.d/private_domains.conf
```

Add the template file by running `vi /usr/local/opnsense/service/templates/sampleuser/Unbound/private_domains.conf` with the content:

```text
server:
  private-domain: internal.example.com
  local-data: "internal.example.com. 10800 IN SOA opnsense.internal.example.com. root.example.com. 1 3600 1200 604800 10800"
```

Run the following to verify the configuration:

```shell
# generate template
configctl template reload sampleuser/Unbound
# show generated file
cat /usr/local/etc/unbound.opnsense.d/private_domains.conf
# check if configuration is valid
configctl unbound check
```

## NAT

NAT translates private to public IP addresses. We need to set this up for both the WAN and the VPN gateways:

- All VLANs are translated to the WAN address range
- VPN VLAN is translated to VPN_WAN and WAN ranges (selective routing)

Navigate to `Firewall` &rarr; `NAT` &rarr; `Outbound`

- Select `Manual outbound NAT rule generation`
- Click `Save` and `Apply changes`

If there are any rules, go ahead and delete them. Then add the following rules:

### localhost to WAN

- Click `+`
- `Interface`: `WAN`
- `Source address`: `Loopback net`
- `Description`: `localhost to WAN`

### LAN to WAN

- Click `+`
- `Interface`: `WAN`
- `Source address`: `LAN net`
- `Description`: `LAN to WAN`

### VLAN10_MANAGE to WAN

- Click `+`
- `Interface`: `WAN`
- `Source address`: `VLAN10_MANAGE net`
- `Description`: `VLAN10_MANAGE to WAN`

### VLAN20_VPN to WAN

- Click `+`
- `Interface`: `WAN`
- `Source address`: `VLAN20_VPN net`
- `Description`: `VLAN20_VPN to WAN`

### VLAN30_CLEAR to WAN

- Click `+`
- `Interface`: `WAN`
- `Source address`: `VLAN30_CLEAR net`
- `Description`: `VLAN30_CLEAR to WAN`

### VLAN40_GUEST to WAN

- Click `+`
- `Interface`: `WAN`
- `Source address`: `VLAN40_GUEST net`
- `Description`: `VLAN40_GUEST to WAN`

### VLAN20_VPN to VPN0_WAN

- Click `+`
- `Interface`: `WAN`
- `Source address`: `VLAN20_VPN net`
- `Description`: `VLAN20_VPN to VPN0_WAN`

### VLAN20_VPN to VPN1_WAN

- Click `+`
- `Interface`: `WAN`
- `Source address`: `VLAN20_VPN net`
- `Description`: `VLAN20_VPN to VPN1_WAN`
