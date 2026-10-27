# 1. Initial Assessment — What I Walked Into

[← Back to README](../README.md)

## Why This Project Started

The church asked me to clean up its network. Over time, routers had been added wherever coverage was needed, and the result was:

- Many routers, each running its own network with its own IP range, not connected to each other.
- Printers that were only reachable from some parts of the network.
- Too many Wi-Fi networks (SSIDs) to choose from.

While I was working on this, a second problem appeared: the internet **dropped in and out**, and speed swung between fast and unusable. Finding the cause of that became the core of this project.

## Existing Environment

The network had grown over time, with routers added wherever coverage was needed. Before changing anything, I inventoried what was on the floor.

| Equipment | Notes |
|---|---|
| Verizon Fios router | ISP-provided router; the actual internet gateway (192.168.1.1) |
| TP-Link Archer BE800 | Wi-Fi router added for coverage |
| TP-Link Archer C9 | Older Wi-Fi router added for coverage |
| ASUS RT-AC86U | Wi-Fi router added for coverage |
| Cisco Aironet AP | Ceiling-mounted, older installation, separate SSID |
| Printers | Several office printers and one commercial printer |
| AV equipment | Dante audio-over-IP devices (identified later in the capture) |

**Physical path** from the main router to a typical workstation:

```
Verizon router → switch → secondary router → long cable runs joined with RJ45 couplers → PC
```

## Initial Observations

| Observation | Why it mattered |
|---|---|
| Several routers each running **their own network**, with their own IP ranges and SSIDs | Devices on different routers were effectively on different networks and could not reliably reach each other or shared printers |
| **Many SSIDs** broadcast in the same space | Devices stuck to whichever network they joined first instead of the nearest AP |
| Long cable runs with **couplers** between segments | A classic source of intermittent link problems |
| `ping` to an outside host returned a mix of normal replies and `Destination net unreachable` **from 192.168.1.1** | The gateway address itself was giving inconsistent answers |

Example from a workstation:

```
Pinging github.com [140.82.112.3] with 32 bytes of data:
Request timed out.
Request timed out.
Reply from 192.168.1.1: Destination net unreachable.
Reply from 192.168.1.1: Destination net unreachable.
```

## Possible Causes Considered

| Hypothesis | Evidence for | Evidence against / how to test |
|---|---|---|
| Loose cable or bad coupler | Long runs and couplers; intermittent symptoms | A bad cable usually causes timeouts, not an explicit "unreachable" answer from the gateway |
| ISP outage | Internet-only symptoms | Other times the same host replied in 12 ms; an outage would not alternate that quickly |
| Secondary router uplink on the wrong port (WAN instead of LAN) | Common mistake with routers used as APs | Checked: the uplink was already on a LAN port |
| **More than one device acting as gateway or DHCP server** | Multiple routers still running their own networks; "unreachable" coming from the gateway address | **Test with a packet capture** |

The last hypothesis explained the strange pattern best: one packet reached the internet, the next was refused by "the gateway." The fastest way to confirm it was to look at the traffic directly.

Next: [Wireshark analysis](02-wireshark-analysis.md).
