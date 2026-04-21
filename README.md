# CCNA MEGA LAB — ICS445 Network Management & Security

> **King Fahd University of Petroleum & Minerals (KFUPM)**
> Course: ICS445 — Network Management & Security
> Instructor: Dr. Farag Azzedin
> Term: 252 (April 2026)
> https://ics445project.replit.app/ics445-presentation/

---

## Team Members

| Name | ID |
|------|-----|
| Abdullah Basel AlQalalweh | 202251980 |
| Mohamed Ashraf Serag | 202183250 |
| Ibrahim Yousef Abdoh | 202171350 |

---

## Project Overview

This repository contains the full implementation of a multi-site enterprise network across **3 phases and 9 parts**, designed and configured in **Cisco Packet Tracer**. The network connects two offices (Office A and Office B) through a hierarchical core/distribution/access architecture, with a shared edge router (R1) connecting to dual ISPs.

The `slides-website/` directory contains a live interactive presentation deck built with **React + Vite** covering all phases, commands, and design justifications.

---

## Network Topology

```
                    Internet (ISP1 + ISP2)
                          |
                         R1
                    /         \
                 CSW1 ======= CSW2   (L3 EtherChannel)
               /    \       /    \
          DSW-A1  DSW-A2  DSW-B1  DSW-B2
           |  \   / |      |  \  / |
         ASW-A1 A2 A3    ASW-B1 B2 B3

Office A (Yellow):                Office B (Cyan):
  WLC1, LWAP1                       LWAP2, SRV1
  Phone1, Phone2                    Phone3
  PC1, PC2, Laptop1                 PC3, Laptop2
```

### Device Inventory

| Layer | Devices |
|-------|---------|
| Edge | R1 (Dual ISP) |
| Core | CSW1, CSW2 |
| Distribution A | DSW-A1, DSW-A2 |
| Distribution B | DSW-B1, DSW-B2 |
| Access A | ASW-A1, ASW-A2, ASW-A3 |
| Access B | ASW-B1, ASW-B2, ASW-B3 |
| Wireless | WLC1, LWAP1, LWAP2 |
| Server | SRV1 (Office B) |

### VLAN Design

| VLAN | Name | Purpose |
|------|------|---------|
| 10 | PCs | Workstation traffic |
| 20 | Phones | Voice/VoIP |
| 30 | Servers | SRV1 + Wi-Fi Office B |
| 40 | Wi-Fi | Wireless Office A |
| 99 | Management | Device management |
| 1000 | Native | Untagged (no data) |

---

## Phase 1

### Part 1 — Initial Setup & Security Baseline
Applied to all 13 devices.

```ios
hostname R1
enable secret 9 <scrypt-hash>
username admin secret 9 <scrypt-hash>
service password-encryption

line console 0
 login local
 exec-timeout 30 0
 logging synchronous

line vty 0 15
 login local
 transport input ssh
 exec-timeout 30 0
```

**Why:** Type 9 (scrypt) is the strongest Cisco password hash — resistant to brute force even if configs are leaked. `exec-timeout` closes idle sessions. `logging synchronous` prevents syslog messages from interrupting command input.

---

### Part 2 — VLANs & Layer-2 EtherChannel

**Office A — PAgP:**
```ios
interface range Gi0/1-2
 channel-group 1 mode desirable
interface port-channel 1
 switchport mode trunk
 switchport trunk native vlan 1000
 switchport nonegotiate
```

**Office B — LACP:**
```ios
interface range Gi0/1-2
 channel-group 1 mode active
interface port-channel 1
 switchport mode trunk
 switchport trunk native vlan 1000
 switchport nonegotiate
```

**VTP:**
```ios
vtp mode server
vtp version 2
```

**Why:** EtherChannel doubles bandwidth and adds link redundancy. Native VLAN 1000 prevents VLAN hopping attacks. DTP disabled (`nonegotiate`) stops rogue trunking. VTPv2 propagates VLAN database automatically.

---

## Phase 2

### Part 3 — IPv4 Addressing, L3 EtherChannel & HSRPv2

**L3 EtherChannel between CSW1–CSW2:**
```ios
interface range Gi1/0/1-2
 no switchport
 channel-group 1 mode active
interface port-channel 1
 ip address 10.X.X.X 255.255.255.0
ip routing
```

**HSRPv2 — Active (DSW-A1 / DSW-B1):**
```ios
interface vlan 10
 standby version 2
 standby 10 ip 10.A.10.1
 standby 10 priority 150
 standby 10 preempt
```

**HSRPv2 — Standby (DSW-A2 / DSW-B2):**
```ios
interface vlan 10
 standby 10 ip 10.A.10.1
 standby 10 priority 100
 standby 10 preempt
```

**Why:** L3 EtherChannel allows both core links to carry traffic simultaneously without STP blocking. HSRPv2 provides a virtual gateway — if the active switch fails, standby takes over in seconds with no end-user reconfiguration needed.

---

### Part 4 — Rapid PVST+ Spanning Tree

```ios
spanning-tree mode rapid-pvst

! On DSW-A1 (root for VLAN 10, 99):
spanning-tree vlan 10,99 priority 0

! On DSW-A2 (root for VLAN 20, 40):
spanning-tree vlan 20,40 priority 0

! Access ports:
interface range Fa0/1-24
 spanning-tree portfast
 spanning-tree bpduguard enable
```

**Why:** Rapid PVST+ converges in under 1 second vs. 30–50 seconds for classic STP. Root bridge is aligned with HSRP active to keep L2 and L3 traffic on the same switch. PortFast lets PCs connect instantly. BPDU Guard shuts down the port if a rogue switch is plugged in.

---

## Phase 3

### Part 5 — OSPFv2 Routing

```ios
! R1 — Dual ISP failover:
ip route 0.0.0.0 0.0.0.0 [ISP1-GW]
ip route 0.0.0.0 0.0.0.0 [ISP2-GW] 10

! All devices:
router ospf 1
 router-id X.X.X.X
 network X.X.X.X 0.0.0.255 area 0
 passive-interface Vlan10
 default-information originate

! CSW point-to-point links:
interface Gi1/0/1
 ip ospf network point-to-point
 ip ospf 1 area 0
```

**Why:** OSPFv2 dynamically discovers all internal routes and self-heals on link failure. Floating static (AD=10) provides ISP failover. Passive interfaces hide routing from end users. Point-to-point skips DR/BDR election on switch links.

---

### Part 6 (1/2) — DHCP, NTP & NAT

```ios
! DHCP on R1:
ip dhcp excluded-address 10.A.10.1 10.A.10.10
ip dhcp pool VLAN10-A
 network 10.A.10.0 255.255.255.0
 default-router 10.A.10.1
 dns-server [SRV1-IP]

! Relay on DSW SVIs:
interface vlan 10
 ip helper-address [R1-IP]

! NTP:
ntp master 5
ntp authentication-key 1 md5 [key]
ntp authenticate
ntp trusted-key 1

! NAT/PAT:
ip nat pool POOL1 203.0.113.200 203.0.113.207 netmask 255.255.255.248
ip nat inside source list 2 pool POOL1 overload
ip nat inside source static [SRV1-Private] [SRV1-Public]
```

**Why:** Centralized DHCP on R1 simplifies IP pool management. `ip helper-address` forwards broadcast DHCP requests across VLAN boundaries. NTP auth prevents rogue time servers. Static NAT exposes SRV1 to the internet; PAT allows all hosts to share a small public IP pool.

---

### Part 6 (2/2) — SSH, SNMP, Syslog & FTP

```ios
! SSHv2:
ip domain-name corp.local
crypto key generate rsa modulus 2048
ip ssh version 2

! VTY ACL (Office A PCs only):
ip access-list standard SSH-ACL
 permit [OfficeA-subnet]
line vty 0 15
 access-class SSH-ACL in
 transport input ssh
 login local

! SNMP:
snmp-server community [string] RO

! Syslog:
logging buffered 8192
logging host [SRV1-IP]

! FTP IOS upgrade:
ip ftp username [user]
ip ftp password [pass]
copy ftp://[SRV1]/[image].bin flash:
boot system flash:[image].bin
```

**Why:** SSHv2 encrypts all management traffic (Telnet is plaintext). RSA 2048-bit is minimum recommended key size. VTY ACL restricts SSH access to trusted subnets only. SNMP read-only lets NMS monitor without configuration access. Centralized syslog on SRV1 enables audit trails and incident response.

---

### Part 7 — ACL, Port Security, DHCP Snooping & DAI

```ios
! Extended ACL:
ip access-list extended OfficeA_to_OfficeB
 permit icmp [OfficeA] [OfficeB]
 deny ip [OfficeA] [OfficeB]
 permit ip any any
interface Gi0/1
 ip access-group OfficeA_to_OfficeB in

! Port Security:
switchport port-security
switchport port-security maximum 2
switchport port-security violation restrict
switchport port-security mac-address sticky

! DHCP Snooping:
ip dhcp snooping
ip dhcp snooping vlan 10,20,40,99
no ip dhcp snooping information option
interface Gi0/1
 ip dhcp snooping trust

! DAI:
ip arp inspection vlan 10,20,40,99
ip arp inspection validate src-mac dst-mac ip
interface Gi0/1
 ip arp inspection trust
```

**Why:** Extended ACL allows ping between offices but blocks all other inter-office IP traffic. Placed near source (best practice). Port security sticky MAC restricts each port to 2 known devices. DHCP Snooping blocks rogue DHCP servers (MITM). DAI validates ARP against the snooping binding table, preventing ARP spoofing.

---

### Part 8 — IPv6 Dual-Stack

```ios
! R1:
ipv6 unicast-routing
interface Gi0/0
 ipv6 address 2001:DB8:X:X::1/64
 ipv6 address FE80::1 link-local
 no shutdown

ipv6 route ::/0 [ISP1-IPv6-GW]
ipv6 route ::/0 [ISP2-IPv6-GW] 10

! CSW1/CSW2:
ipv6 unicast-routing
interface Gi1/0/1
 ipv6 address 2001:DB8:X:Y::1/64
 ipv6 address FE80::X link-local
```

**Why:** Dual-stack runs IPv4 and IPv6 simultaneously — no tunneling needed, no disruption to existing IPv4 traffic. Link-local (FE80::) addresses are mandatory for IPv6 neighbor discovery. Floating IPv6 static (AD=10) mirrors the IPv4 ISP failover design.

---

### Part 9 — Wireless WLC, WLAN & LWAP

Configured via **WLC1 GUI**:

| Setting | Value |
|---------|-------|
| Dynamic Interface | Wi-Fi |
| VLAN | 40 |
| IP / Gateway | 10.6.0.2 / 10.6.0.1 |
| WLAN Profile | Wi-Fi |
| SSID | Wi-Fi |
| Security | WPA2 + AES (CCMP) |
| Auth | PSK — cisco123 |
| Architecture | Centralized (CAPWAP) |

**Why:** Centralized WLC manages all APs from one place — consistent policy, easier firmware updates, seamless roaming. CAPWAP tunnels carry both control and data to WLC1. VLAN 40 isolates wireless traffic. WPA2-AES replaces deprecated WPA-TKIP for strong per-packet encryption.

---

## Interactive Slides Website

The `slides-website/` directory contains the fully built static version of the presentation.

### Running Locally

```bash
# Serve the static files (any HTTP server works)
cd slides-website
npx serve .

# Or with Python:
python3 -m http.server 8080
```

Then open `http://localhost:8080` in your browser.

### Navigation
- **Arrow keys** or **click arrows** to navigate between slides
- **Cmd+Shift+F** (Mac) / **F11** (Windows) for fullscreen
- 14 slides covering all 3 phases and 9 parts

---

## Tech Stack (Presentation)

| Technology | Purpose |
|-----------|---------|
| React 18 + TypeScript | Slide components |
| Vite 7 | Build tool |
| Tailwind CSS | Styling |
| Space Grotesk | Display font |
| IBM Plex Mono | Code/monospace font |

---

## Verification Commands

```ios
! EtherChannel:
show etherchannel summary
show interfaces trunk

! HSRP:
show standby brief

! STP:
show spanning-tree vlan 10

! OSPF:
show ip ospf neighbor
show ip route ospf

! DHCP:
show ip dhcp binding

! NAT:
show ip nat translations

! Security:
show ip dhcp snooping
show ip arp inspection
show port-security interface Fa0/1

! SSH:
show ip ssh

! IPv6:
show ipv6 interface brief
show ipv6 route
```

---

*ICS445 — KFUPM — Term 252 — April 2026*
