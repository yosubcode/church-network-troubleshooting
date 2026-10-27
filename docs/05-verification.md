# 5. Verification — Proving the Fix with a Second Capture

[← Back to README](../README.md)

## Why a Second Capture

"The internet seems fine now" is not proof. Intermittent problems can disappear for a while on their own. After all the [reconfiguration](03-reconfiguration.md) was done, I captured again from the same wired workstation (`192.168.1.100`) and compared it against the [original capture](02-wireshark-analysis.md).

## Before vs. After

| Check | Before | After |
|---|---|---|
| `duplicate use of 192.168.1.1 detected!` warnings | **3** (frames 45, 46, 126) | **0** |
| Devices sending ARP as 192.168.1.1 | Verizon (Arcadyan) **and** TP-Link | **Verizon (Arcadyan) only** |
| ICMP `Destination unreachable (Network unreachable)` from the gateway | 4 (frames 108, 137, 151, 153) | **0** |
| Outbound HTTPS from the workstation | SYN → ICMP error → retransmission | **Full TCP handshake + TLS 1.3 sessions completed** |
| UDP 7437 broadcast (TP-Link Archer C9) | Sent from **192.168.1.1** | Sent from **192.168.1.251** |

## Key Evidence

### 1. The gateway address now belongs to one device

Every ARP message sent from 192.168.1.1 in the second capture came from the Verizon router's MAC address (Arcadyan OUI), for example:

| Frame | Source | Info |
|---|---|---|
| 1 | Arcadyan_xx:xx:xx | Who has 192.168.1.229? Tell 192.168.1.1 |
| 3 | Arcadyan_xx:xx:xx | Who has 192.168.1.203? Tell 192.168.1.1 |
| 6 | Arcadyan_xx:xx:xx | Who has 192.168.1.223? Tell 192.168.1.1 |

No TP-Link MAC appears as 192.168.1.1, and Wireshark raised no duplicate-address warning.

### 2. The conflicting device moved to its new address

The Archer C9 periodically broadcasts on UDP port 7437. This same broadcast is a fingerprint that can be followed across both captures:

| Capture | Frame | Source → Destination | Info |
|---|---|---|---|
| Before | 9, 73 | **192.168.1.1** → 255.255.255.255 | UDP 53409 → 7437 |
| After | 18 | **192.168.1.251** → 255.255.255.255 | UDP 59444 → 7437 |

The device that had been impersonating the gateway was now broadcasting from the address assigned to it during reconfiguration. This ties the root cause and the fix to the same physical device.

### 3. Internet traffic completes normally

Where the first capture showed connection attempts answered by `Network unreachable`, the second capture shows the workstation resolving and connecting to an external site, with the TCP handshake and TLS 1.3 negotiation completing:

| Frame | Source → Destination | Info |
|---|---|---|
| 36 | 192.168.1.100 → external host | TCP SYN to port 443 |
| 37 | external host → 192.168.1.100 | **SYN, ACK** |
| 38 | 192.168.1.100 → external host | ACK |
| 39 | 192.168.1.100 → external host | TLS 1.3 Client Hello |
| 41 | external host → 192.168.1.100 | **TLS 1.3 Server Hello** |

## Limits of This Verification

- The second capture covers only a few seconds (~250 frames). A longer capture, ideally during a busy service when many devices are connected, would be stronger evidence.
- It was taken from one wired workstation. Traffic from wireless clients on each AP was not captured separately.
- Other traffic is still present and expected, such as multicast from the Dante audio equipment on 192.168.50.x. These are covered in [Open Items](06-open-items.md).

## Re-checking in the Future

The same checks can be repeated in a few minutes on any new capture:

| Filter | Expected result on a healthy network |
|---|---|
| `arp.duplicate-address-detected` | No frames |
| `arp.src.proto_ipv4 == 192.168.1.1` | One MAC address only (the Verizon router) |
| `icmp.type == 3 && ip.src == 192.168.1.1` | No frames during normal browsing |

Next: [Open items & recommendations](06-open-items.md).
