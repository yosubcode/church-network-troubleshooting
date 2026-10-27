# 2. Wireshark Analysis — Finding the Duplicate Gateway

[← Back to README](../README.md)

## Goal

Users reported that the internet would work, drop, and come back, with speeds swinging between fast and unusable. A `ping` to an outside host showed the pattern clearly: one normal reply, followed by `Destination net unreachable` from the gateway address.

```
Reply from 140.82.114.4: bytes=32 time=12ms TTL=52
Reply from 192.168.1.1: Destination net unreachable.
Reply from 192.168.1.1: Destination net unreachable.
Reply from 192.168.1.1: Destination net unreachable.
```

A packet that reached the internet and a "no route" answer from the *same* gateway address a moment later suggested that **more than one device was answering as the gateway**. I captured traffic on a wired workstation to confirm.

## Capture Setup

| Item | Value |
|---|---|
| Capture point | Wired Windows workstation on the main LAN (`192.168.1.100`) |
| Tool | Wireshark |
| Capture window | ~6 seconds (~280 frames) recorded while the problem was occurring |

> MAC addresses below are masked to the vendor prefix (e.g. `TP-Link_xx:xx:xx`). Vendors were identified from the MAC OUI.

## Finding 1 — Wireshark flags a duplicate 192.168.1.1

Wireshark's ARP analysis raised an expert warning three times:

| Frame | Source | Info |
|---|---|---|
| 45 | Arcadyan_xx:xx:xx | Who has 192.168.1.200? Tell 192.168.1.1 **(duplicate use of 192.168.1.1 detected!)** |
| 46 | Arcadyan_xx:xx:xx | Who has 192.168.1.201? Tell 192.168.1.1 **(duplicate use of 192.168.1.1 detected!)** |
| 126 | Arcadyan_xx:xx:xx | Who has 192.168.1.208? Tell 192.168.1.1 **(duplicate use of 192.168.1.1 detected!)** |

Wireshark raises this warning when it sees the same IP address claimed by **two different MAC addresses**. Arcadyan is the manufacturer of the Verizon Fios router, which is the real gateway.

## Finding 2 — A TP-Link router is also answering as the gateway

When the workstation asked for the gateway's MAC address, the reply did **not** come from the Verizon router:

| Frame | Source | Info |
|---|---|---|
| 19 | Workstation | Who has 192.168.1.1? Tell 192.168.1.100 |
| 21 | **TP-Link_xx:xx:xx** | **192.168.1.1 is at TP-Link_xx:xx:xx** |

The same TP-Link device was also sending its own ARP requests from 192.168.1.1 (frames 140–148), including malformed requests for `254.128.0.0`. It was not just answering by accident; it was configured with the gateway's address.

**Conclusion:** two devices were claiming 192.168.1.1:

| Device | Vendor (OUI) | Role |
|---|---|---|
| Verizon Fios router | Arcadyan | Real gateway, connected to the internet |
| Secondary router | TP-Link | Still configured as a router with LAN IP 192.168.1.1, no internet uplink |

## Finding 3 — Why the symptoms were intermittent

Clients keep the gateway's MAC address in their ARP cache. Depending on **which device answered last**, a client sent its internet traffic to:

- the **Verizon router** → traffic reached the internet normally, or
- the **TP-Link router** → it had no route to the internet, so connections failed.

The capture shows the second case directly. The workstation tried to open HTTPS connections and immediately received ICMP errors back from "192.168.1.1":

| Frame | Source → Destination | Info |
|---|---|---|
| 107 | 192.168.1.100 → external host | TCP SYN to port 443 |
| 108 | 192.168.1.1 → 192.168.1.100 | **ICMP Destination unreachable (Network unreachable)** |
| 136 | 192.168.1.100 → external host | TCP SYN **retransmission** |
| 137 | 192.168.1.1 → 192.168.1.100 | **ICMP Destination unreachable (Network unreachable)** |

This matches the `ping` output exactly: some packets reached the internet, others were answered by a gateway that could not route them. Whichever ARP reply a client had cached decided whether the internet "worked" at that moment.

## Finding 4 — Other routers still running their own networks

The capture also showed SSDP announcements from addresses outside the main subnet:

| Source | Observation | Likely meaning |
|---|---|---|
| 192.168.0.1 | Repeated SSDP NOTIFY bursts | A router on its factory-default TP-Link subnet, still acting as its own router |
| 192.168.50.1 | Repeated SSDP NOTIFY bursts | A router on its factory-default ASUS subnet (an ASUS RT-AC86U also identified itself via LLDP, frame 91) |
| 192.168.50.89 | Multicast audio traffic | Dante audio-over-IP equipment (Audinate OUI) on its own subnet |

At least two "secondary" routers were therefore still behaving as independent routers on the same physical LAN instead of acting as access points. This explained the mixed subnets and added unnecessary broadcast and multicast noise.

## What Was Not the Root Cause

The capture also contained `[Malformed Packet]` (IPv6) and `[TCP ACKed unseen segment]` warnings. These appeared on otherwise healthy connections and are commonly caused by NIC offloading or capture timing rather than by the network itself, so they were noted but not treated as the cause.

## Useful Display Filters

To reproduce this analysis on another capture:

| Filter | Shows |
|---|---|
| `arp.duplicate-address-detected` | Frames where Wireshark detected an IP claimed by two MACs |
| `arp.src.proto_ipv4 == 192.168.1.1` | Every ARP message sent as the gateway, to see which MACs use it |
| `icmp.type == 3` | ICMP Destination Unreachable messages |
| `ssdp` | SSDP announcements, useful for spotting stray routers and their subnets |

## Result

The analysis pointed to a specific device and a specific misconfiguration: a TP-Link router still set up as a router at 192.168.1.1. That led directly to the fix in [Reconfiguration](03-reconfiguration.md), and the follow-up capture in [Verification](05-verification.md) confirmed it.
