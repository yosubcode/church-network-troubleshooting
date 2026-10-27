# 3. Reconfiguration — One Gateway, Many Access Points

[← Back to README](../README.md)

## Design Goal

The [Wireshark analysis](02-wireshark-analysis.md) showed secondary routers acting as independent routers on the same LAN, one of them even claiming the gateway address. The target design was simple:

| Rule | Why |
|---|---|
| **One router, one gateway, one DHCP server** | Only the Verizon Fios router (192.168.1.1) routes traffic and hands out addresses. Every other device follows it. |
| **Every other router becomes an access point** | Secondary routers only provide Wi-Fi and switch ports. No routing, no DHCP, no separate subnet. |
| **One SSID everywhere** | Same network name and password on every AP and every band, so devices can roam between APs. |

## Device-by-Device Configuration

Each secondary router needed a different approach, depending on whether its firmware had a real AP mode.

| Device | AP support | Method | Address | Uplink port |
|---|---|---|---|---|
| Verizon Fios router | — (main router) | Left as router and sole DHCP server | 192.168.1.1 (static) | Internet |
| TP-Link Archer BE800 | Native AP mode | Operation mode → Access Point | DHCP from main router | WAN / Internet port |
| ASUS RT-AC86U | Native AP mode | Operation mode → Access Point (AP) mode, "Automatic IP" | DHCP from main router | WAN port |
| TP-Link Archer C9 | **No AP mode** | Manual: DHCP server disabled, LAN IP set by hand | **192.168.1.251** (static) | **LAN port** |

### Native AP mode (BE800, RT-AC86U)

Switching the operation mode to Access Point turns the router into a bridge: routing and the DHCP server are disabled, and the device requests its own management address from the main router.

- **ASUS:** chose *Access Point (AP) mode*, not *AiMesh*. AiMesh only works when the main router is also ASUS; here the main router is Verizon.
- **TP-Link BE800:** after switching modes, the DHCP server setting showed *Auto* instead of *Off*. Auto disables itself only when it detects an upstream DHCP server, so I set it explicitly to **Off** rather than relying on detection.
- Before the uplink was working, the BE800 showed a fallback address of 192.168.0.254. Once connected to the main router it received a 192.168.1.x address, and a laptop plugged into it got gateway **192.168.1.1** from the Verizon router, confirming that DHCP was passing through instead of being served locally.

### Manual AP configuration (Archer C9)

The older Archer C9 firmware had no operation-mode setting, so it was converted into an AP by hand:

1. **Isolated it first.** Connected a laptop directly to the C9 with all other cables unplugged, so its admin page could be reached without colliding with the main network.
2. **Moved its LAN IP into the main subnet, away from the gateway:** `192.168.1.251`. Addresses near the top of the range are rarely handed out by DHCP.
3. **Disabled its DHCP server.**
4. **Connected the uplink to a LAN port, leaving WAN empty.** Without AP mode, the WAN port would still be treated as a routed internet port and would not bridge the main network to the C9's clients.

> **Evidence this was the conflicting device:** in the first capture, a broadcast on UDP port 7437 came from 192.168.1.1. In the follow-up capture, the same broadcast came from **192.168.1.251**. The device that had been impersonating the gateway was now on its new address. See [Verification](05-verification.md).

> **Lesson:** before assigning .251, I pinged the address to check it was free. The laptop was still on the C9's own 192.168.0.x network, so the reply (`Destination net unreachable`) proved nothing about the main subnet. The address was only truly confirmed by the follow-up capture.

## Wi-Fi and Roaming

| Setting | Value | Reason |
|---|---|---|
| SSID | Same name on all APs | Devices only roam between APs that broadcast the same network |
| Password and security type | Identical on all APs | A mismatch makes devices treat them as different networks |
| Band steering | *Smart Connect* enabled on BE800 and RT-AC86U; *Separate 2.4/5 GHz* left unchecked | One name for all bands; the AP steers devices between 2.4 GHz and 5/6 GHz |
| Fast roaming | BE800: no separate fast-roaming toggle in this firmware's wireless settings; RT-AC86U offers *Roaming Assistant* (disconnects clients below a signal threshold, default −70 dBm) | Helps devices move to the nearest AP instead of clinging to a weak one |

**Limitation:** the Archer C9 does not support fast-roaming standards. Devices roam away from it only when its signal becomes weak, which can cause a brief reconnect.

## Device Hardening

- Set unique administrator credentials on the routers instead of factory defaults.
- Updated router firmware.
- WPA2 encryption on all Wi-Fi networks. The Archer C9 is currently in WPA/WPA2 mixed mode; switching it to WPA2-only (AES) is listed in [Open Items](06-open-items.md).

## Resulting Layout

| Address | Device | Role |
|---|---|---|
| 192.168.1.1 | Verizon Fios router | Gateway, router, sole DHCP server |
| 192.168.1.x (DHCP) | TP-Link Archer BE800 | Access point |
| 192.168.1.x (DHCP) | ASUS RT-AC86U | Access point |
| 192.168.1.251 (static) | TP-Link Archer C9 | Access point (manual) |
| 192.168.1.x | Cisco Aironet ceiling APs | Legacy APs, separate SSID, left in place (see [Open Items](06-open-items.md)) |

Next: getting the uplinks to actually pass traffic → [Physical-layer troubleshooting](04-physical-layer.md).
