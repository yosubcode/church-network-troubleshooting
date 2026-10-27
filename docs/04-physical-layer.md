# 4. Uplink Troubleshooting — Working Layer by Layer

[← Back to README](../README.md)

## The Problem

After converting the TP-Link Archer BE800 to an access point, it still would not pass internet traffic:

- The BE800's front display showed a **"!" warning** (no internet detected).
- A laptop plugged into the BE800 got an address and a default gateway of **192.168.1.1**, but web pages would not load.

So something was reaching the laptop, but the path to the internet was broken.

## Approach: Check One Layer at a Time

| Step | Check | Result | What it ruled out |
|---|---|---|---|
| 1 | `ipconfig` on the laptop | Got a 192.168.1.x address and gateway 192.168.1.1 | Not a dead cable; DHCP answers were reaching the laptop |
| 2 | Confirmed the BE800 was in AP mode and its own DHCP server was **Off** | Correct | The BE800 itself was not handing out addresses |
| 3 | Moved the uplink between the BE800's ports (LAN vs. WAN/Internet, 10 Gbps vs. 2.5 Gbps combo ports) and rebooted the AP | Still "!" | Not just the AP's port choice |
| 4 | Traced the cable back to the **main router** and found it in the Verizon router's **10 Gbps WAN/LAN port** | — | Found a likely suspect on the other end |
| 5 | Moved the cable to a **standard LAN port** on the Verizon router | Warning cleared; internet worked | **Fix** |

## Confirmation

After the move, `ipconfig` on the laptop connected through the BE800:

```
IPv4 Address . . . . . . : 192.168.1.231
Default Gateway  . . . . : 192.168.1.1
DNS Suffix . . . . . . . : (Verizon network suffix)
```

The address came from the Verizon router's DHCP pool, the gateway was the real gateway, and web traffic flowed normally.

## What I Am Not Certain Of

I want to be precise about root cause here:

- **Most likely:** the link between the Verizon router's 10 Gbps port and the BE800 did not negotiate cleanly, so traffic did not pass reliably. Moving to a standard port fixed it immediately.
- **Also possible:** this work happened while the [duplicate gateway](02-wireshark-analysis.md) (a TP-Link router answering as 192.168.1.1 with DHCP still on) was still active. "Got a gateway of 192.168.1.1 but no internet" is exactly what a client would see if its address came from that device instead of the Verizon router. Step 1 checked the gateway *address*, not *which device* owned it.

**What I would do differently:** run `arp -a` on the laptop and compare the gateway's MAC address against the Verizon router's. That one command would have separated "bad link" from "wrong gateway" in seconds.

## Takeaways

- Work bottom-up: link → addressing → gateway → internet. Each check rules out a layer.
- A correct-looking gateway **IP** does not prove you have the correct gateway **device**. Check the MAC.
- High-speed multi-gig ports on consumer routers can be picky; a standard 1 Gbps / 2.5 Gbps LAN port is the safer choice for an AP uplink unless the extra speed is needed.

Next: [Verification](05-verification.md) of the whole network after all changes.
