# 6. Open Items & Recommendations

[← Back to README](../README.md)

The immediate problem is fixed and [verified](05-verification.md). This page records the risks and weaknesses I found along the way that are **not yet resolved**, ranked by priority. Some are quick configuration changes; others need a decision from the church.

## Summary

| # | Item | Risk | Effort | Priority |
|---|---|---|---|---|
| 1 | Static IP .251 may sit inside the DHCP pool | Future IP conflict | Low | **High** |
| 2 | Archer C9 allows legacy WPA (TKIP) | Weaker Wi-Fi encryption | Low | **High** |
| 3 | Legacy Cisco APs are unmanaged | Unknown security state, separate SSID | Medium | **High** |
| 4 | Staff, guests, and AV equipment share one flat network | No isolation between trust levels | Medium | Medium |
| 5 | One shared Wi-Fi password for everyone | Hard to revoke access | Low | Medium |
| 6 | Multicast / SSDP noise | Performance | Low | Low |
| 7 | Considered: expanding the subnet to /22 | Larger broadcast domain | Low | Evaluate first |

---

## 1. Keep the C9's static address out of the DHCP pool

The Archer C9 was given a static address, **192.168.1.251**, by hand. If the Verizon router's DHCP pool includes .251, it could one day hand that same address to a phone or laptop, recreating the exact kind of conflict this project fixed.

**Recommendation:** check the Verizon DHCP range and either shrink the pool so it ends below .251, or create a DHCP reservation for the C9's MAC address. While there, add reservations for the other APs too, so each AP's management address never changes.

## 2. Switch the Archer C9 to WPA2-only (AES)

The C9 is set to **WPA/WPA2-Personal**, which still allows the original WPA with TKIP, a deprecated protocol with known weaknesses. All modern devices support WPA2-AES.

**Recommendation:** change the C9's security mode to **WPA2-Personal (AES)** on both bands. The C9 is also an older model without fast-roaming support; when budget allows, replacing it with an AP that supports WPA3 would remove this item entirely.

## 3. Bring the Cisco ceiling APs under management, or remove them

Cisco Aironet APs mounted on the ceiling still broadcast their **own separate SSID**. I identified them in the main router's device list by their MAC prefixes (Cisco OUI) and tried to reach them, but they expose no web interface. That behavior is typical of **lightweight (controller-based) APs**, which take their configuration from a wireless LAN controller, or of units configured for console/CLI access only.

Why this matters from a security standpoint:

- Nobody currently on site can see or change their configuration, encryption settings, or firmware version.
- They are active devices on the main LAN that no one is managing.
- Their separate SSID also prevents roaming with the rest of the network.

**Recommendation:**
1. Ask whether the church ever had a Cisco wireless controller or an outside vendor managing these APs.
2. If not, either factory-reset them and configure them directly through the console port, or **unplug them**, after confirming the new APs cover the same areas.
3. Until then, list them in the network inventory as unmanaged devices.

## 4. Separate guests, staff, and AV equipment

Everything currently shares one network: staff computers, visitors' phones, printers, and Dante audio-over-IP equipment (seen on 192.168.50.x in both captures). A visitor's infected phone sits on the same network as staff PCs.

**Recommendation (in order of effort):**
1. **Guest Wi-Fi with client isolation.** Most of the existing routers offer a guest network that blocks guests from reaching local devices. This is the quickest win.
2. **Keep the Dante audio network on its own switch.** Audio-over-IP is sensitive to multicast and latency; isolating it protects both the audio and the main network.
3. **VLANs for staff / guest / AV** if a managed switch is added later. This fits naturally with any future expansion to other floors.

## 5. Rotate the shared Wi-Fi password

One password is shared by everyone, and it has been written down during setup. Anyone who has ever connected keeps access indefinitely.

**Recommendation:** change the password after the guest network (item 4) is in place, give the main network's password only to staff, and rotate it on a schedule (for example, yearly or when staff change).

## 6. Reduce multicast noise

The first capture showed repeated SSDP announcements and constant multicast traffic. Some of it came from routers that are now APs, and some is normal (Dante, device discovery).

**Recommendation:** disable UPnP on the APs where it is not needed, and enable IGMP snooping where the firmware supports it, so multicast only reaches devices that asked for it.

## 7. Subnet expansion to /22: evaluated, not applied

I considered changing the subnet mask from **/24** (254 usable addresses) to **/22** (1,022 usable addresses) to make room for more devices.

| For | Against |
|---|---|
| More addresses during crowded events | A larger broadcast domain: every ARP, SSDP, and multicast packet reaches up to 4× as many devices |
| | Every device with a manual mask (such as the C9) must be updated, or it will mis-route |
| | Does not address the real constraint if few devices are connected at once |

**Decision:** not applied. First, check the Verizon router's DHCP lease table during a busy service. If the pool is genuinely running out, **shortening the DHCP lease time** for the guest network is a simpler fix. If more than ~200 devices are regularly online at once, revisit /22 together with the VLAN plan in item 4.

---

[← Back to README](../README.md)
