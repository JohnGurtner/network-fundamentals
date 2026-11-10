# Network Fundamentals: Physical Layer & Ethernet

> Built a three-switch, three-host network from factory-reset hardware and hand-terminated cable, then
> used Wireshark and a **SPAN (port mirroring)** session to watch the traffic on the wire. Also measured
> how link speed and duplex affect real transfer times.

*CNIT 344, Lab 1, Group 7 (with Ethan Price).*

## Planning

### Topology

```
              Rack switch (uplink / WAN)
                        │
                 ┌──────┴──────┐
                 │ Cisco 3750A │  G7SW1, console ◄── PC1
                 └──┬───────┬──┘
     SPAN: src Gi3/0/3 ─► dst Gi3/0/1 (monitor PC)
                    │       │
          ┌─────────┴─┐   ┌─┴──────────────┐
          │Cisco 3750B│   │ HP ProCurve    │  console ◄── PC3
          └───────────┘   └────────────────┘
          console ◄── PC2
```

### Addressing

| Host | OS | IP | Mask | Gateway | DNS |
| --- | --- | --- | --- | --- | --- |
| PC1 | Windows 11 | 44.34.7.2 | 255.255.255.0 | 44.34.7.1 | 44.2.1.44, 44.2.1.45 |
| PC2 | Linux | 44.34.7.3 | 255.255.255.0 | 44.34.7.1 | 44.2.1.44, 44.2.1.45 |
| PC3 | Windows 11 | 44.34.7.4 | 255.255.255.0 | 44.34.7.1 | 44.2.1.44, 44.2.1.45 |

The default gateway is needed because traffic to anything outside `44.34.7.0/24`, including the
internet, has to be routed.

## Procedure

### Phase 1: Build and verify the network

**1. Factory-reset the Cisco 3750s over the console (PuTTY, serial).**
Power-cycled while holding **Mode** to drop into the boot loader, then:

```
switch: flash_init
switch: delete flash:vlan.dat
switch: delete flash:config.text
switch: boot
Switch# write erase
Switch# reload
Switch# copy running-config startup-config
```

Deleting `vlan.dat` matters. VLANs live outside the startup config and survive `write erase` on their own.

**2. Static IPs.** Set on Windows through *Network and Sharing Center → IPv4 properties*. On Linux, through
a new wired connection profile with Manual IPv4 (see Problem 3).

**3. Connectivity checks.** `ping 1.1.1.1` from every host, and added the lab printer by IP.

**4. SMB file share.** Created `G7Share` on PC1 and opened it from Linux in Dolphin
(`smb://K206B02PC01/G7Share`), then did the same between PC3 and PC2.

**5. Recorded switch state** for documentation:

```
show mac address-table
show running-config
show startup-config
show vlan brief
```

Result: one VLAN (VLAN 1) on all ports, and only static CPU entries in the MAC table on a freshly reset switch.

### Phase 2: Packet analysis

**6. Captured an SMB file transfer.** Started Wireshark on PC1 with display filter `tcp.port == 445`,
copied a ~600-word file through the share, and saved the capture.

**7. Set up a SPAN session** on the 3750A to mirror the uplink to a monitoring port:

```
Switch# configure terminal
Switch(config)# monitor session 1 source interface gi3/0/3
Switch(config)# monitor session 1 destination interface gi3/0/1
Switch(config)# end
Switch# show monitor session 1
```

**8. Watched another host's web traffic.** With Wireshark on the monitoring PC (filter `dns`), loaded
`espn.com` on PC1, which never ran a capture itself. The capture still showed the DNS lookups, ARP for
the gateway, the TCP handshake and the TLS session. Then removed the session with `no monitor session 1`.

**9. Built and tested cables.** Terminated a **TIA-568B straight-through** and a **568A/568B crossover**
patch cable and checked both with a cable tester.

### Phase 3: Expand the topology and measure performance

**10. Added the second 3750 and the HP ProCurve** (reset with the front-panel *Reset + Clear* buttons),
cabled to match the planned hierarchy.

**11. Made exact-size test files:**

```powershell
fsutil file createnew 1MB.txt   1048576
fsutil file createnew 100MB.txt 104857600
fsutil file createnew 250MB.txt 261440000
```

**12. Timed transfers at each NIC speed/duplex setting** (*NIC properties → Advanced → Speed & Duplex*):

```powershell
Measure-Command { Copy-Item -Path 100MB.txt -Destination \\K206B02PC03\G7Share\ }
```

Repeated the uploads to Google Drive at each setting.

**13. Changed PC3's MAC address** (*NIC → Advanced → Locally Administered Address* = `001122334455`),
checked it with `getmac -v`, and watched how the host and switch reacted.

**14. Captured a 250 MB transfer** from PC3 to PC2 and counted frames and retransmissions.

## Findings

### Transfer times: host to internet (Google Drive upload)

| Link setting | 1 MB | 100 MB | Theoretical minimum, 100 MB |
| --- | --- | --- | --- |
| 100 Mbps full duplex | 1.28 s | 13.5 s | ~8.4 s |
| 100 Mbps half duplex | 1.52 s | 15.7 s | ~8.4 s |
| 10 Mbps full duplex | 2.35 s | 1 min 37 s | ~84 s |
| 10 Mbps half duplex | 2.82 s | 1 min 52 s | ~84 s |

- Dropping from 100 to 10 Mbps made the 100 MB upload about **7× slower**, close to the expected 10×.
  The overhead from TCP, TLS and the internet path explains the gap.
- **Half duplex was slower than full duplex at every speed** (about 15% slower at 10 Mbps for 100 MB).
  Half duplex can't send and receive at the same time, so TCP's ACKs compete with the data stream.
- Small (1 MB) transfers are dominated by latency and setup, so link speed matters less there.

> **Data note:** The LAN-to-LAN timings in the original report (100 MB in under half a second at
> 10 Mbps) are faster than the link physically allows. Looking back at the procedure, the timed copy ran
> on PC3 *to PC3's own share* (`\\K206B02PC03\...`), so the data never crossed the wire. Those results
> are left out here. Next time, the copy should run from a different host.

### Frame math for the 250 MB transfer
- File size 261,440,000 bytes ÷ 1,500-byte Ethernet payload = **at least 174,294 frames**.
- The capture showed **more** than that, because of TCP ACKs, retransmissions, and ARP/DNS/SMB control
  traffic that isn't part of the payload.
- In practice the TCP payload per frame is about 1,460 bytes after the IP and TCP headers, so the real
  minimum is even higher.

### Encapsulation, read from one SMB2 packet

| TCP/IP layer | What the packet showed |
| --- | --- |
| Application | SMB2 data, the file contents |
| Transport | TCP header, port 445 |
| Internet | IP header, source and destination host IPs |
| Network access | Ethernet header, source and destination MACs |

Protocols involved: **ARP** (find the destination MAC), **DNS** (resolve names), **TCP** (reliable
delivery), **SMB2** (the file share itself).

**Could someone rebuild the file from the capture?** Yes, if every TCP segment was captured. SMB over
port 445 here is unencrypted, so anyone with access to a SPAN port or tap can read file contents
straight off the wire.

### Cabling reference
- **Straight-through** (568B on both ends) connects *different* device types, like PC to switch.
- **Crossover** (568A on one end, 568B on the other) connects *like* devices, like switch to switch.
- **Rollover** is the Cisco console cable. It carries management, not Ethernet frames.
- 568B pin order: white-orange, orange, white-green, blue, white-blue, green, white-brown, brown.
- These are better called "twisted-pair patch cables" than "Ethernet cables." Ethernet is the L1/L2
  standard, and the cable is defined by its pinout.

### MAC address behavior
The switch builds its MAC table by learning **source** MACs on each port, forwards known destinations out
the matching port, and floods unknown ones. When a host's MAC changes, the switch learns the new
address on the same port as soon as frames arrive, and the stale entry ages out. Changing a MAC this way is legitimate in some cases (for
example, matching a port-security allow-list). It's also exactly how **MAC spoofing** gets around naive
port security.

## Problems and fixes

**1. The 3750 kept the previous group's config after a reset.**
Releasing the Mode button after the first round of flashing LEDs wasn't enough.
**Fix:** Hold Mode through the *second* flash cycle. Then the switch fully dropped to factory defaults.

**2. New files in the SMB share didn't show up until the folder was reopened.**
**Fix:** Connecting to the share by **IP address** instead of hostname made changes appear right away.
That points to name-resolution caching on the hostname path.

**3. The Linux PC kept asking for an admin password when setting a static IP.**
Editing the existing system-wide connection required root.
**Fix:** Created a **new** wired connection and unchecked *"All users may connect to this network,"* so
the profile was scoped to the current user and needed no elevation.

## Why it's in a security portfolio

- **SPAN/port mirroring is how IDS sensors and packet-capture appliances get their data.** This lab
  set one up on real hardware and pulled another host's DNS and TLS traffic from it.
- Reading a capture layer by layer and rebuilding a file from TCP segments is a core SOC and
  network-forensics skill. It also shows why unencrypted SMB is a data-exposure risk.
- Knowing how switches learn MACs and how MACs can be changed is the basis for understanding
  port security, MAC spoofing and CAM-table attacks.
- Getting a device back to a known-good state (including deleting `vlan.dat`) is a basic step before
  deploying network hardware.

## Skills demonstrated

Cisco IOS · console access & switch recovery · SPAN / port mirroring · Wireshark (display filters,
protocol analysis) · TCP/IP encapsulation · SMB · static IPv4 on Windows & Linux · TIA-568A/B cable
termination & testing · speed/duplex performance testing · PowerShell `Measure-Command` · troubleshooting

## Screenshots

See [`screenshots/`](screenshots/) for switch configs, the SPAN session, the Wireshark captures and the
cable tests from the original report.
