<div align="center">

# 🛡️ Secure Campus Network: Design & Implementation

**A multi-site university network built in Cisco Packet Tracer and hardened end to end: VLAN segmentation, role-based ACLs, RADIUS-backed AAA, WPA2-Enterprise wireless, and a site-to-site IPsec VPN.**

![Cisco Packet Tracer](https://img.shields.io/badge/Cisco%20Packet%20Tracer-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![Cisco IOS](https://img.shields.io/badge/Cisco%20IOS-15.1-049FD9?style=for-the-badge&logo=cisco&logoColor=white)
![Course](https://img.shields.io/badge/CSCE%203423-Intro%20to%20Cybersecurity-8A2BE2?style=for-the-badge)
![Term](https://img.shields.io/badge/Fall-2025-555?style=for-the-badge)

<img src="docs/screenshots/topology-full.png" alt="Full logical topology of the secure campus network" width="100%">

<sub>Full logical topology: Main Campus (Buildings A, B, C), Small Campus (Building D + Library), the remote Research Facility, and the external email server.</sub>

</div>

---

## 📑 Table of Contents

- [🛡️ Secure Campus Network: Design \& Implementation](#️-secure-campus-network-design--implementation)
  - [📑 Table of Contents](#-table-of-contents)
  - [🧭 Overview](#-overview)
  - [✨ Highlights](#-highlights)
  - [🏗️ Network Architecture](#️-network-architecture)
    - [Building-level topologies](#building-level-topologies)
  - [🗺️ VLAN \& IP Addressing Plan](#️-vlan--ip-addressing-plan)
  - [🧩 Core Services](#-core-services)
  - [🔒 Security Implementation](#-security-implementation)
    - [1. Layer 2 Security](#1-layer-2-security)
    - [2. Inter-VLAN Routing \& DHCP Relay](#2-inter-vlan-routing--dhcp-relay)
    - [3. Role-Based Access Control (ACLs)](#3-role-based-access-control-acls)
    - [4. Secure Management: SSH, AAA, SNMP, Syslog](#4-secure-management-ssh-aaa-snmp-syslog)
    - [5. Wireless (WLC + Lightweight APs)](#5-wireless-wlc--lightweight-aps)
    - [6. Site-to-Site IPsec VPN](#6-site-to-site-ipsec-vpn)
    - [7. Email Service](#7-email-service)
  - [✅ Verification Results](#-verification-results)
  - [⚠️ Risk Assessment \& Security Policies](#️-risk-assessment--security-policies)
  - [📁 Repository Structure](#-repository-structure)
  - [🚀 Getting Started](#-getting-started)
  - [🔭 Future Improvements](#-future-improvements)
  - [👥 Team](#-team)

---

## 🧭 Overview

**X University** runs two campuses 25 miles apart and a remote **Research Field Station**. It has four faculties: Sciences, Business, Engineering and Liberal Arts. The network has to give faculty, staff, students and guests reliable wired and wireless access, including BYOD, while keeping each group separated according to its privileges.

This project designs that network from scratch, prototypes it in **Cisco Packet Tracer**, and secures it in layers, from switch ports up to the WAN. Every addressing and routing decision is carved out of the base network **`10.20.0.0/16`**.

| Site | Buildings / Areas | Role |
|---|---|---|
| **Main Campus** | **Building A**: Administration (HR, Finance) & Faculty of Business<br>**Building B**: Faculties of Engineering & Sciences<br>**Building C**: IT, student labs & data center (DHCP, DNS, Web, AAA, Syslog) | Core of the network; hosts all central services |
| **Small Campus** | **Building D**: Faculty of Liberal Arts<br>**Library** & Student Services | Connected to the Main Campus over a point-to-point `/30` serial link |
| **Research Field Station** | Small office / lab | Connected over the WAN through a **site-to-site IPsec VPN** |
| **External** | Cloud-hosted email server (`campus.edu`) | Reached through a static route |

---

## ✨ Highlights

- 🧱 **15 VLANs** segmenting users by role: Department Heads, Professors, Students (wired and Wi-Fi), Admin Staff, Network Admins, Servers, per-SSID Wi-Fi VLANs, Library, Liberal Arts, Management, plus a **black-hole VLAN 999** for every unused port.
- 🔀 **Router-on-a-stick** inter-VLAN routing on both campus routers using 802.1Q subinterfaces.
- 📡 **Centralized DHCP** in Building C (`10.20.60.10`) serves *every* VLAN on both campuses through `ip helper-address` relay.
- 🚦 **Seven named ACLs** that put least-privilege rules into practice: students and guests can't reach internal subnets, nobody outside IT can touch the management VLAN, and SNMP and VTY access are restricted to admins.
- 🔐 **AAA with RADIUS** (`10.20.60.11`) plus local fallback, for login, exec authorization, and start-stop accounting.
- 🖥️ **SSH-only** device management (`transport input ssh`), with hashed `enable secret`.
- 📶 **4 SSIDs** on WLC-managed lightweight APs: **WPA2-Enterprise** (RADIUS) for faculty and staff, **WPA2-PSK** for students and guests, with guests limited to web traffic.
- 🌐 **IPsec site-to-site VPN** (ESP-AES / ESP-SHA-HMAC) from the Research Facility to the main campus gateway.
- 📊 **SNMP monitoring** and a central **Syslog** server for visibility.

---

## 🏗️ Network Architecture

```mermaid
flowchart TB
    subgraph EXT["☁️ External"]
        MAIL["Email Server<br/>209.165.200.10<br/>campus.edu"]
    end

    subgraph RF["🔬 Research Field Station"]
        RFR["Research Router<br/>LAN 10.20.250.0/24"]
        RPC["Research PCs"]
        RFR --- RPC
    end

    subgraph MAIN["🏛️ Main Campus"]
        GW["Router-Gateway-University<br/>Router-on-a-stick · AAA · IPsec peer"]
        CORE["SW-Main-Campus<br/>(802.1Q trunks)"]
        A["Building A<br/>Admin · HR · Finance · Business"]
        B["Building B<br/>Engineering & Sciences"]
        C["Building C<br/>DHCP/DNS · Web · AAA · Syslog · WLC"]
        GW --- CORE
        CORE --- A
        CORE --- B
        CORE --- C
    end

    subgraph SMALL["🏫 Small Campus"]
        SR["Router-Small-Campus<br/>Router-on-a-stick"]
        SSW["SW-Small-Campus"]
        D["Building D<br/>Liberal Arts"]
        LIB["Library"]
        WLC2["WLC-Small-Campus"]
        SR --- SSW
        SSW --- D
        SSW --- LIB
        SSW --- WLC2
    end

    GW ==>|"Serial /30<br/>10.20.1.0/30"| SR
    GW -. "🔒 IPsec VPN tunnel<br/>over WAN" .- RFR
    GW -->|"static route"| MAIL
```

### Building-level topologies

<table>
  <tr>
    <td align="center" width="50%"><b>Building A: Administration & Business</b><br><img src="docs/screenshots/building-a.png" alt="Building A topology"></td>
    <td align="center" width="50%"><b>Building B: Engineering & Sciences</b><br><img src="docs/screenshots/building-b.png" alt="Building B topology"></td>
  </tr>
  <tr>
    <td align="center"><b>Building C: IT, Labs & Data Center</b><br><img src="docs/screenshots/building-c.png" alt="Building C topology"></td>
    <td align="center"><b>Small Campus: Building D & Library</b><br><img src="docs/screenshots/small-campus.png" alt="Small campus topology"></td>
  </tr>
</table>

---

## 🗺️ VLAN & IP Addressing Plan

**Base network:** `10.20.0.0/16`. User VLANs get a `/24` each, and the inter-site links use `/30`. Each campus uses its own subnet for the same VLAN ID, so the same role gets the same VLAN everywhere while the routing stays clean.

| VLAN | Name | Role / Purpose | Main Campus Subnet | Small Campus Subnet | SSID |
|:---:|---|---|---|---|---|
| **10** | `DEPT_HEADS` | Department heads (restricted high-level access) | `10.20.10.0/24` | `10.20.15.0/24` | — |
| **20** | `PROFESSORS` | Professors & faculty (wired) | `10.20.20.0/24` | `10.20.25.0/24` | — |
| **30** | `STUDENTS_WIRED` | Student labs | `10.20.30.0/24` | `10.20.35.0/24` | — |
| **31** | `STUDENTS_WIFI` | Student wireless | `10.20.31.0/24` | `10.20.36.0/24` | `StudentNet` |
| **40** | `ADMIN_STAFF` | HR / Finance / Administration | `10.20.40.0/24` | `10.20.45.0/24` | — |
| **50** | `NET_ADMINS` | IT / network administrators | `10.20.50.0/24` | — | — |
| **60** | `SERVERS` | DHCP/DNS, Web, AAA, Syslog | `10.20.60.0/24` | — | — |
| **70** | `FACULTY_WIFI` | Faculty wireless | `10.20.70.0/24` | `10.20.75.0/24` | `FacultyNet` |
| **80** | `STAFF_WIFI` | Admin & IT staff wireless | `10.20.80.0/24` | `10.20.85.0/24` | `StaffNet` |
| **90** | `GUEST_WIFI` | Guests: internet-only | `10.20.90.0/24` | `10.20.95.0/24` | `GuestNet` |
| **100** | `LIBRARY_STAFF` | Library & student services | `10.20.100.0/24` | `10.20.105.0/24` | — |
| **110** | `LIBERAL_ARTS` | Faculty of Liberal Arts | `10.20.110.0/24` | `10.20.115.0/24` | — |
| **200** | `MGMT_VLAN` | Management: SSH, AAA, SNMP, Syslog | `10.20.200.0/24` | `10.20.205.0/24` | — |
| **999** | `UNUSED_BLACKHOLE` | Parking VLAN for all unused ports | — | — | — |

**WAN and point-to-point links**

| Link | Subnet | Endpoints |
|---|---|---|
| Main ↔ Small Campus (Serial) | `10.20.1.0/30` | Gateway `S0/0/0` `10.20.1.1` ↔ Small Campus `10.20.1.2` |
| Main Campus ↔ WAN / Research | `10.20.1.4/30` | Gateway `S0/0/1` `10.20.1.5` |
| IPsec peers | — | Gateway `10.20.1.5` ↔ Research Facility `10.20.1.10` |
| Research Facility LAN | `10.20.250.0/24` | Protected by the VPN crypto map |

> The gateway on every VLAN is the `.1` address. Clients get their addresses from the DHCP server in Building C. For example, a Sci & Eng student PC is leased `10.20.30.10/24`, gateway `10.20.30.1`, DNS `10.20.60.10`.

---

## 🧩 Core Services

| Service | Host | Address | Notes |
|---|---|---|---|
| **DHCP + DNS** | `DHCP-Server` (Building C) | `10.20.60.10` | One pool per VLAN per campus; reached through relay |
| **AAA (RADIUS)** | `AAA` server (Building C) | `10.20.60.11` | Device login and WPA2-Enterprise wireless auth |
| **Syslog** | `SysLog` server (Building C) | VLAN 60 | Collects logs from routers and key switches |
| **Web** | `Web Server` (Building C) | VLAN 60 | On-premises university web services |
| **WLC** | `WLC_Main_Campus` / `WLC_Small_Campus` | — | Manage the lightweight APs and SSIDs |
| **Email** | Cloud email server | `209.165.200.10` | SMTP + POP3, domain `campus.edu` |

<details>
<summary>📸 <b>DHCP server pools</b> (click to expand)</summary>
<br>
<img src="docs/screenshots/dhcp-server-pools.png" alt="DHCP server pools for each VLAN on both campuses">
</details>

---

## 🔒 Security Implementation

### 1. Layer 2 Security

| Control | Implementation |
|---|---|
| **VLAN segmentation** | Role-based VLANs on every access switch |
| **Port security** | `maximum 2` MAC addresses, **sticky** learning, violation mode **restrict** |
| **Unused ports** | Moved into the black-hole **VLAN 999** (`UNUSED_BLACKHOLE`) |
| **Trunking** | Static 802.1Q trunks (`mode on`) with **explicit allowed-VLAN lists** per link |

<table>
  <tr>
    <td align="center" width="50%"><b>Port security on SW-Build-A</b><br><img src="docs/screenshots/port-security.png" alt="show port-security interface fa0/2"></td>
    <td align="center" width="50%"><b>Unused ports parked in VLAN 999</b><br><img src="docs/screenshots/vlans-building-a-blackhole.png" alt="show vlan brief on SW-Build-A"></td>
  </tr>
  <tr>
    <td align="center"><b>Main-campus trunks: pruned allowed VLANs</b><br><img src="docs/screenshots/trunks-main-campus.png" alt="show interfaces trunk main campus"></td>
    <td align="center"><b>Small-campus trunks</b><br><img src="docs/screenshots/trunks-small-campus.png" alt="show interfaces trunk small campus"></td>
  </tr>
</table>

<details>
<summary>📸 <b>VLAN database on the core switch</b></summary>
<br>
<img src="docs/screenshots/vlans-core-switch.png" alt="show vlan brief on SW-Main-Campus">
</details>

### 2. Inter-VLAN Routing & DHCP Relay

Both campus routers use **router-on-a-stick**. Each VLAN gets an 802.1Q subinterface on `G0/0` that carries:
- the VLAN's default gateway (`.1`)
- `ip helper-address 10.20.60.10`, which relays DHCP to the central server in Building C
- an inbound ACL (`ip access-group <ROLE>_OUT in`) that enforces that role's policy

```cisco
interface GigabitEthernet0/0.30
 description Small Campus - Students VLAN 30
 encapsulation dot1Q 30
 ip address 10.20.30.1 255.255.255.0
 ip helper-address 10.20.60.10
 ip access-group STUDENTS_OUT in
```

<table>
  <tr>
    <td align="center" width="50%"><b>Gateway subinterfaces (<code>show ip int brief</code>)</b><br><img src="docs/screenshots/gateway-interfaces.png" alt="Router-Gateway-University show ip interface brief"></td>
    <td align="center" width="50%"><b>Small-campus router subinterfaces</b><br><img src="docs/screenshots/small-campus-router-interfaces.png" alt="Router-Small-Campus show ip interface brief"></td>
  </tr>
  <tr>
    <td align="center"><b>DHCP relay on every subinterface</b><br><img src="docs/screenshots/dhcp-relay-helper-small-campus.png" alt="show run include helper"></td>
    <td align="center"><b>Client receives a lease through the relay</b><br><img src="docs/screenshots/dhcp-client-lease.png" alt="PC obtaining address via DHCP"></td>
  </tr>
</table>

<details>
<summary>📸 <b>Full subinterface configuration</b></summary>
<br>
<img src="docs/screenshots/router-on-a-stick-subinterfaces.png" alt="show run section interface GigabitEthernet0/0">
<br><br>
<img src="docs/screenshots/dhcp-relay-helper-gateway.png" alt="Gateway helper-address verification">
</details>

### 3. Role-Based Access Control (ACLs)

Each user role has its own **extended ACL** applied inbound on its gateway subinterface. Every ACL follows the same pattern: let the essentials through (DHCP, DNS, the server farm), block what the role shouldn't reach, then allow internet-bound traffic.

| ACL | Applied to | Policy |
|---|---|---|
| `STUDENTS_OUT` | VLAN 30, 31 | ❌ SNMP · ✅ DHCP, DNS · ✅ Servers (`10.20.60.0/24`) · ❌ **all other internal subnets** (`10.20.0.0/16`) · ✅ Internet |
| `GUEST_OUT` | VLAN 90 (main campus) | ❌ SNMP · ✅ DHCP, DNS · ✅ **HTTP / HTTPS only** · ❌ **all internal subnets** · internet-only access |
| `PROFS_OUT` | VLAN 20, 70 (main), 110 | ❌ SNMP · ✅ DHCP, DNS, Servers · ❌ **Management VLAN** · ✅ everything else |
| `ADMIN_OUT` | VLAN 40, 80 | ❌ SNMP · ✅ DHCP, DNS, Servers · ❌ **Management VLAN** · ✅ everything else |
| `DEPTHEADS_OUT` | VLAN 10 | ❌ SNMP · ✅ DHCP, DNS, Servers · ❌ **Management VLAN** · ✅ everything else |
| `LIBRARY_OUT` | VLAN 100 (small campus) | Library role policy |
| `MGMT_PROTECT` | VLAN 200 | ✅ Net-Admin subnet (`10.20.50.0/24`) ↔ Management · ❌ other access to `10.20.200.0/24` |
| `MGMT_VTY_ONLY` *(standard)* | VTY lines | Only `10.20.50.0/24` may open SSH sessions |
| `MGMT_SNMP_ONLY` *(standard)* | SNMP community | Only `10.20.200.0/24` may poll SNMP |

```cisco
ip access-list extended GUEST_OUT
 10 deny   udp any any eq snmp
 20 permit udp any any eq bootps
 30 permit udp any eq bootpc any eq bootps
 40 permit udp any any eq domain
 50 permit tcp any any eq www
 60 permit tcp any any eq 443
 70 deny   ip any 10.20.0.0 0.0.255.255
 80 permit ip any any
```

**Proof that it works:** a student PC and a non-admin PC both try to reach a host in the Net-Admins VLAN (`10.20.50.5`) and get dropped.

<table>
  <tr>
    <td align="center" width="50%"><b>Student → Net-Admin: <i>Destination host unreachable</i></b><br><img src="docs/screenshots/acl-test-student-blocked.png" alt="Student ping blocked by ACL"></td>
    <td align="center" width="50%"><b>Unauthorized PC → Net-Admin: <i>Request timed out</i></b><br><img src="docs/screenshots/acl-test-pc5-blocked.png" alt="Unauthorized PC ping blocked"></td>
  </tr>
</table>

<details>
<summary>📸 <b>ACL definitions and interface bindings</b></summary>
<br>
<img src="docs/screenshots/acl-definitions-1.png" alt="show access-lists part 1">
<br><br>
<img src="docs/screenshots/acl-definitions-2.png" alt="show access-lists part 2">
<br><br>
<img src="docs/screenshots/acl-applied-interfaces.png" alt="ACLs applied to subinterfaces">
</details>

### 4. Secure Management: SSH, AAA, SNMP, Syslog

**SSH-only remote access.** VTY lines accept SSH only (`transport input ssh`), restricted to the Net-Admin subnet, and the privileged password is stored as a hashed `enable secret`.

**AAA.** RADIUS comes first, with the local user database as a fallback so admins aren't locked out if the server goes down:

```cisco
aaa new-model
aaa authentication login default group radius local
aaa authorization exec default group radius local
aaa accounting exec default start-stop group radius
```

**SNMP.** Communities are protected by `MGMT_SNMP_ONLY`. A management PC uses the MIB Browser to query routers, for example `sysName` returns `Router-Small-Campus`.

**Syslog.** Routers and key switches send their logs to the central Syslog server, so config changes, link flaps and security events are all collected in one place.

<table>
  <tr>
    <td align="center" width="50%"><b>SSH enabled, VTY locked down</b><br><img src="docs/screenshots/ssh-config.png" alt="show ip ssh"></td>
    <td align="center" width="50%"><b>Admin PC SSH login (<code>ssh -l netadmin</code>)</b><br><img src="docs/screenshots/ssh-login-mgmt-pc.png" alt="SSH login from management PC"></td>
  </tr>
  <tr>
    <td align="center"><b>AAA configuration</b><br><img src="docs/screenshots/aaa-config.png" alt="show run section aaa"></td>
    <td align="center"><b>Hardening: <code>enable secret</code> + SSH-only</b><br><img src="docs/screenshots/hardening-ssh-secret.png" alt="Hardening commands"></td>
  </tr>
  <tr>
    <td align="center"><b>Central Syslog receiving events</b><br><img src="docs/screenshots/syslog-server.png" alt="Syslog server entries"></td>
    <td align="center"><b>SNMP MIB Browser query</b><br><img src="docs/screenshots/snmp-mib-browser.png" alt="SNMP MIB browser"></td>
  </tr>
</table>

### 5. Wireless (WLC + Lightweight APs)

Lightweight APs in every building join a **Wireless LAN Controller** on each campus. A single AP broadcasts all four SSIDs and maps each one to its own VLAN.

| SSID | VLAN | Security | Authentication | Who |
|---|:---:|---|---|---|
| `FacultyNet` | 70 | **WPA2-Enterprise**, AES | RADIUS `10.20.60.11` | Professors & academic staff |
| `StaffNet` | 80 | **WPA2-Enterprise**, AES | RADIUS `10.20.60.11` | Administrative & IT staff |
| `StudentNet` | 31 | WPA2-PSK, AES | Pre-shared key | Registered students |
| `GuestNet` | 90 | WPA2-PSK, AES + `GUEST_OUT` ACL | Pre-shared key | Visitors: web access only |

<table>
  <tr>
    <td align="center" width="50%"><b>FacultyNet: WPA2-Enterprise via RADIUS</b><br><img src="docs/screenshots/wlan-facultynet.png" alt="FacultyNet WLAN settings"></td>
    <td align="center" width="50%"><b>StaffNet: WPA2-Enterprise via RADIUS</b><br><img src="docs/screenshots/wlan-staffnet.png" alt="StaffNet WLAN settings"></td>
  </tr>
  <tr>
    <td align="center"><b>StudentNet: WPA2-PSK</b><br><img src="docs/screenshots/wlan-studentnet.png" alt="StudentNet WLAN settings"></td>
    <td align="center"><b>GuestNet: WPA2-PSK, ACL-restricted</b><br><img src="docs/screenshots/wlan-guestnet.png" alt="GuestNet WLAN settings"></td>
  </tr>
  <tr>
    <td align="center"><b>All WLANs mapped to the AP group</b><br><img src="docs/screenshots/wlc-ap-groups.png" alt="WLC AP groups"></td>
    <td align="center"><b>One AP serving clients on different VLANs</b><br><img src="docs/screenshots/wireless-clients.png" alt="Laptops connected to Building B AP"></td>
  </tr>
</table>

### 6. Site-to-Site IPsec VPN

The Research Field Station connects to the main campus through an **IPsec tunnel** over the WAN. Traffic between the research LAN (`10.20.250.0/24`) and the campus (`10.20.0.0/16`) matches the crypto ACL and is encrypted with **ESP-AES** and authenticated with **ESP-SHA-HMAC**.

<table>
  <tr>
    <td align="center" width="50%"><b><code>show crypto ipsec sa</code>: packets encrypted and decrypted</b><br><img src="docs/screenshots/vpn-ipsec-sa.png" alt="show crypto ipsec sa"></td>
    <td align="center" width="50%"><b>Research PC → DHCP server in Building C across the tunnel</b><br><img src="docs/screenshots/vpn-ping-research-to-dhcp.png" alt="Ping across VPN"></td>
  </tr>
</table>

> The first ping times out while IKE negotiates the security association. After that, traffic flows encrypted through the tunnel.

### 7. Email Service

The external email server hosts the `campus.edu` domain with **SMTP** and **POP3** turned on. Each user mailbox has its own password, and the campus reaches the server through a static route.

<table>
  <tr>
    <td align="center" width="33%"><b>Mail server</b><br><img src="docs/screenshots/email-server.png" alt="Email server configuration"></td>
    <td align="center" width="33%"><b>Client configuration</b><br><img src="docs/screenshots/email-client-config.png" alt="Mail client configuration"></td>
    <td align="center" width="33%"><b>Mail delivered</b><br><img src="docs/screenshots/email-received.png" alt="Mail received in inbox"></td>
  </tr>
</table>

---

## ✅ Verification Results

| # | Test | Expected | Result |
|:-:|---|---|:-:|
| 1 | Client in VLAN 30 requests an address | Lease from `10.20.60.10` via relay | ✅ `10.20.30.10/24` |
| 2 | Student PC → Net-Admin host `10.20.50.5` | Blocked by `STUDENTS_OUT` | ✅ Unreachable |
| 3 | Non-admin PC → Net-Admin host `10.20.50.5` | Blocked | ✅ Timed out |
| 4 | Net-Admin PC → `ssh -l netadmin 10.20.200.1` | SSH session, AAA login | ✅ Logged in |
| 5 | `show ip ssh` on gateway | SSH enabled | ✅ v1.99, 120 s timeout, 3 retries |
| 6 | Port security on access port | Secure-up, sticky MACs | ✅ 2 sticky, 0 violations |
| 7 | Research PC → `10.20.60.10` | Encrypted through IPsec | ✅ 3/4 replies (1st lost to IKE setup) |
| 8 | `show crypto ipsec sa` | Encaps/decaps increasing | ✅ 3 encrypted / 3 decrypted |
| 9 | Wireless clients on one AP | Placed in different VLANs per SSID | ✅ |
| 10 | SNMP GET `sysName` from management PC | Returns hostname | ✅ `Router-Small-Campus` |
| 11 | Config change on router | Logged to Syslog | ✅ `%SYS-5-CONFIG_I` received |
| 12 | `ramy@campus.edu` → `omar@campus.edu` | Mail delivered | ✅ Send success / received |

---

## ⚠️ Risk Assessment & Security Policies

| Risk | Impact | Mitigation in this design |
|---|---|---|
| **Unauthorized inter-VLAN access** | Students or guests reach HR, Finance or infrastructure | Per-role extended ACLs on every subinterface; management VLAN reachable only by Net-Admins |
| **Rogue DHCP / ARP spoofing / DHCP starvation** | Man-in-the-middle, denial of service | Port security (MAC limits, sticky); centralized, relayed DHCP (DHCP Snooping + DAI planned, see below) |
| **VLAN hopping** | Traffic leaking between segments | Static 802.1Q trunks (`mode on`), pruned allowed-VLAN lists, unused ports in black-hole VLAN 999 |
| **Compromised guest devices** | Pivoting into the internal network | Dedicated Guest VLAN 90; `GUEST_OUT` permits only DHCP/DNS/HTTP/HTTPS and denies `10.20.0.0/16` |
| **Weak Wi-Fi security** | Credential theft, unauthorized access | WPA2-Enterprise with RADIUS for faculty and staff; WPA2-AES for everyone |
| **Unsecured device management** | Device takeover | SSH-only VTY, VTY ACL, `enable secret`, RADIUS AAA with local fallback |
| **No visibility into events** | Attacks go unnoticed | Central Syslog and SNMP monitoring, restricted to the management VLAN |
| **Interception of remote-site traffic** | Research data exposure | IPsec site-to-site VPN with AES encryption and SHA-HMAC integrity |

**Policies adopted:** ACLs between all user VLANs · WPA2-Enterprise for staff and faculty · Guest isolation · SSH-only management with AAA · Centralized logging · Encrypted VPN for the remote site · Unused ports parked in a black-hole VLAN.

---

## 📁 Repository Structure

```
.
├── Cyber_Project.pkt                                    # Cisco Packet Tracer project (full topology + configs)
├── Secure Campus Network Design & Implementation.pdf   # Full project report (48 pages)
├── docs/
│   └── screenshots/                                    # All figures used in this README
└── README.md
```

📄 **[Read the full report (PDF)](Secure%20Campus%20Network%20Design%20%26%20Implementation.pdf)**

---

## 🚀 Getting Started

1. Install **[Cisco Packet Tracer](https://www.netacad.com/cisco-packet-tracer)** (8.x or newer recommended).
2. Clone the repository:
   ```bash
   git clone https://github.com/<your-username>/<repo-name>.git
   ```
3. Open **`Cyber_Project.pkt`** in Packet Tracer.
4. Explore and try the following:
   - Click any router or switch → **CLI** → `show vlan brief`, `show ip interface brief`, `show access-lists`, `show crypto ipsec sa`
   - From a PC → **Desktop → Command Prompt** → ping across VLANs to watch the ACLs at work
   - From the Net-Admin PC in Building C → `ssh -l netadmin 10.20.200.1`
   - Switch to **Simulation mode** to follow packets through the VPN tunnel

> 🔑 **Note:** All credentials, pre-shared keys and RADIUS secrets in this project exist only inside the simulation and are for lab use. Never reuse them on real equipment.

---

## 🔭 Future Improvements

- Enable **DHCP Snooping** and **Dynamic ARP Inspection** on all access switches, with trusted uplinks, to fully counter rogue DHCP, starvation and ARP spoofing
- Apply `GUEST_OUT` / `PROFS_OUT` on the small-campus Wi-Fi subinterfaces (VLAN 90 and 70), which currently have no ACL bound
- Add `switchport nonegotiate` on trunks and `shutdown` on unused ports
- Move the trunk **native VLAN** off VLAN 1 to an unused VLAN to further harden against double-tagging VLAN-hopping attacks
- Enable `service password-encryption` so the remaining type-7 passwords are hidden in the running config
- Add **SPAN** sessions and a Wireshark capture workflow for traffic analysis
- Add a **firewall / ASA** at the internet edge, and bring in **802.1X** on wired access ports

---

## 👥 Team

| Name |
|---|
| **Ramy Shehata** |
| **Omar Ganna** |
| **Adham Gohar** |

<div align="center">

<sub>Built with Cisco Packet Tracer · Academic term project</sub>

</div>
