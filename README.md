# Network Fundamentals — Physical Layer &amp; Ethernet

> Built a small switched network from bare switches and cable, then used Wireshark and a SPAN port to
> actually watch the traffic — the groundwork for understanding network-based detection.

*CNIT 344 — Lab 1, Group 7 (with Ethan Price).*

## Summary

Hands-on layer-1/layer-2 networking: factory-reset and configured Cisco 3750 and HP ProCurve
switches, built a three-PC network, captured and analyzed traffic with Wireshark, mirrored a port
with SPAN, built and tested Ethernet cables, and measured how NIC speed/duplex affect real transfer
times.

## Lab setup

- 2× Cisco 3750 switches, 1× HP ProCurve switch, Cat6 cabling
- 2× Windows 11 PCs, 1× Linux PC, static IP addressing
- PuTTY (console), Wireshark, a cable tester

## What I did &amp; found

| Phase | Work | Takeaway |
| --- | --- | --- |
| Switch prep | `flash_init`, wiped `vlan.dat`/`config.text`, verified factory reset | Clean, known-good switch state |
| Addressing | Static IPv4 on all three hosts; verified with `ping 1.1.1.1` | Confirmed L3 reachability |
| Capture | Transferred large files while capturing in Wireshark | Saw real frames/packets on the wire |
| SPAN | Mirrored the uplink to a third PC and loaded a web page | **Port mirroring is how an IDS/analyst taps traffic** |
| Cabling | Built TIA-568A/B straight-through + crossover, tested | Verified physical-layer correctness |
| Performance | Compared speed/duplex combos vs theoretical times | Duplex mismatches wreck throughput |
| MAC behavior | Reassigned a MAC, observed host/switch response | How switches learn/age MAC entries |

## Why it's in a security portfolio

Network security starts at the physical and data-link layers. SPAN/port-mirroring is exactly how
packet-capture sensors and IDS get their feed, and reading a capture in Wireshark is a core analyst
skill — this lab is where I did both on real hardware.

## Screenshots

See [`screenshots/`](screenshots/) for switch configs, the Wireshark captures, the SPAN session, and
the cable tests.

## Skills demonstrated

Cisco IOS · switch configuration · VLANs · SPAN / port mirroring · Wireshark packet analysis ·
Ethernet cabling · duplex/speed troubleshooting
