# Networking-portfolio

# 01 — TechNova Inc.: IP Addressing, VLSM & CIDR

## Scenario

TechNova Inc. is opening a new headquarters building alongside an existing
branch office and needs its entire internal network designed from a single
assigned address block, `192.168.10.0/24`. The network must support five
distinct segments of very different sizes — two large department LANs, two
smaller ones, a server farm, and a point-to-point WAN link — without wasting
address space, and must be reachable end-to-end with no unauthorized access
to any device.

## Requirements

- [ ] Divide `192.168.10.0/24` into right-sized subnets using VLSM (no single
      fixed mask for every segment)
- [ ] Address two routers (R1-HQ, R2-Branch), five switches, eight PCs, and
      one server by hand — static addressing only, no DHCP
- [ ] Bring up a DNS + HTTP server resolving `www.technova.com`
- [ ] Connect HQ and Branch over a serial WAN link with classless static
      routing, including one summarized route in place of four
- [ ] Harden every router and switch: encrypted passwords, SSH-only remote
      access, console timeout, legal banner, disabled HTTP/CDP, port
      security, PortFast + BPDU Guard on access ports
- [ ] Verify full connectivity, DNS resolution, SSH access, and that Telnet
      is blocked

## Topology

<img width="960" height="501" alt="Screenshot 2026-10-04 112434" src="https://github.com/user-attachments/assets/af588bd6-480e-49d1-8eac-1628787ce5af" />


| Device | Role | Model |
|---|---|---|
| R1-HQ | HQ router | Cisco 2911 |
| R2-Branch | Branch router | Cisco 1941 |
| SW-IT | IT LAN switch | Cisco 2960 |
| SW-Sales | Sales LAN switch | Cisco 2960 |
| SW-HR | HR LAN switch | Cisco 2960 |
| SW-ServerFarm | Server farm switch | Cisco 2960 |
| SW-Branch | Branch LAN switch | Cisco 2960 |
| PC-IT1, PC-IT2 | IT workstations | Generic PC |
| PC-Sales1, PC-Sales2 | Sales workstations | Generic PC |
| PC-HR1, PC-HR2 | HR workstations | Generic PC |
| PC-Branch1, PC-Branch2 | Branch workstations | Generic PC |
| Server-DNS | DNS + HTTP server | Generic Server |

## IP Addressing Scheme

### Router Interface Modules

Added before cabling (router powered off → insert module → power on).

| Router | Slot | Module | Adds Interface |
|---|---|---|---|
| R1-HQ | Slot 0 | HWIC-2T | Serial0/0/0 (and Serial0/0/1, unused) |
| R1-HQ | Slot 1 | HWIC-1FE / NM-1FE-TX | FastEthernet0/1/0 |
| R2-Branch | WIC slot | HWIC-2T (or WIC-2T) | Serial0/0/0 |

### Router Interface Addressing

| Device | Interface | IP Address | Subnet Mask |
|---|---|---|---|
| R1-HQ | GigabitEthernet0/0 | 192.168.10.65 | 255.255.255.192 |
| R1-HQ | GigabitEthernet0/1 | 192.168.10.129 | 255.255.255.224 |
| R1-HQ | GigabitEthernet0/2 | 192.168.10.161 | 255.255.255.240 |
| R1-HQ | FastEthernet0/1/0 | 192.168.10.177 | 255.255.255.248 |
| R1-HQ | Serial0/0/0 | 192.168.10.185 | 255.255.255.252 |
| R2-Branch | GigabitEthernet0/0 | 192.168.10.1 | 255.255.255.192 |
| R2-Branch | Serial0/0/0 | 192.168.10.186 | 255.255.255.252 |

### Cabling Table

| From Device | From Port | To Device | To Port | Cable Type |
|---|---|---|---|---|
| R1-HQ | GigabitEthernet0/0 | SW-IT | FastEthernet0/1 | Copper Straight-Through |
| R1-HQ | GigabitEthernet0/1 | SW-Sales | FastEthernet0/1 | Copper Straight-Through |
| R1-HQ | GigabitEthernet0/2 | SW-HR | FastEthernet0/1 | Copper Straight-Through |
| R1-HQ | FastEthernet0/1/0 | SW-ServerFarm | FastEthernet0/1 | Copper Straight-Through |
| R1-HQ | Serial0/0/0 | R2-Branch | Serial0/0/0 | Serial DCE |
| R2-Branch | GigabitEthernet0/0 | SW-Branch | FastEthernet0/1 | Copper Straight-Through |
| SW-IT | FastEthernet0/2, 0/3 | PC-IT1, PC-IT2 | FastEthernet | Copper Straight-Through |
| SW-Sales | FastEthernet0/2, 0/3 | PC-Sales1, PC-Sales2 | FastEthernet | Copper Straight-Through |
| SW-HR | FastEthernet0/2, 0/3 | PC-HR1, PC-HR2 | FastEthernet | Copper Straight-Through |
| SW-ServerFarm | FastEthernet0/2 | Server-DNS | FastEthernet0 | Copper Straight-Through |
| SW-Branch | FastEthernet0/2, 0/3 | PC-Branch1, PC-Branch2 | FastEthernet | Copper Straight-Through |

> Serial DCE/DTE: connect R1-HQ's end first so R1 ends up DCE (clock rate
> is set there). Confirm with `show controllers serial0/0/0` — if R2 ended
> up DCE instead, move `clock rate 64000` to R2's Serial0/0/0.

### Switch Management IP Addressing

Each switch's VLAN 1 management IP is the last usable address in the LAN
segment it serves, so it never collides with a PC or the router's gateway.

| Switch | Management IP | Subnet Mask | Default Gateway |
|---|---|---|---|
| SW-Branch | 192.168.10.62 | 255.255.255.192 | 192.168.10.1 |
| SW-IT | 192.168.10.126 | 255.255.255.192 | 192.168.10.65 |
| SW-Sales | 192.168.10.158 | 255.255.255.224 | 192.168.10.129 |
| SW-HR | 192.168.10.174 | 255.255.255.240 | 192.168.10.161 |
| SW-ServerFarm | 192.168.10.182 | 255.255.255.248 | 192.168.10.177 |

### PC and Server Addressing

All static (no DHCP in this build).

| Device | IP Address | Subnet Mask | Default Gateway | DNS Server |
|---|---|---|---|---|
| PC-Branch1 | 192.168.10.2 | 255.255.255.192 | 192.168.10.1 | 192.168.10.178 |
| PC-Branch2 | 192.168.10.3 | 255.255.255.192 | 192.168.10.1 | 192.168.10.178 |
| PC-IT1 | 192.168.10.66 | 255.255.255.192 | 192.168.10.65 | 192.168.10.178 |
| PC-IT2 | 192.168.10.67 | 255.255.255.192 | 192.168.10.65 | 192.168.10.178 |
| PC-Sales1 | 192.168.10.130 | 255.255.255.224 | 192.168.10.129 | 192.168.10.178 |
| PC-Sales2 | 192.168.10.131 | 255.255.255.224 | 192.168.10.129 | 192.168.10.178 |
| PC-HR1 | 192.168.10.162 | 255.255.255.240 | 192.168.10.161 | 192.168.10.178 |
| PC-HR2 | 192.168.10.163 | 255.255.255.240 | 192.168.10.161 | 192.168.10.178 |
| Server-DNS | 192.168.10.178 | 255.255.255.248 | 192.168.10.177 | 192.168.10.178 |

## Design Decisions
 
- **Used VLSM instead of one mask size for every LAN.** If every segment
  used the same `/26` mask, the design would need five 64-address blocks —
  320 addresses — to serve about 153 real users. VLSM lets each subnet be
  sized to what it actually needs, so the same design only uses 188 of the
  256 available addresses.
- **Sized the biggest subnets first.** Branch (60 hosts) was addressed
  before WAN (2 hosts), not the other way around. Carving out small subnets
  first can leave an awkward gap that's too small for a later, bigger
  subnet to fit into — starting big avoids that problem entirely.
- **Replaced four routes on R2-Branch with one summary route.** IT, Sales,
  HR, and the Server Farm all happen to fit inside one block,
  `192.168.10.64/25`. So instead of R2 needing a separate route for each of
  those four LANs, one line — `ip route 192.168.10.64 255.255.255.128
  192.168.10.185` — covers all of them. It's also easier to maintain: a new
  HQ subnet carved from that same /25 would already be covered, with no
  new route needed.
- **Used static routes instead of a routing protocol like OSPF.** With only
  two routers and a topology that isn't going to change on its own, a
  dynamic routing protocol would just add setup complexity without solving
  a real problem. Static routes are simpler here and get the job done.
- **Turned off Telnet, allowed SSH only.** Telnet sends passwords in plain
  text over the network, so anyone watching the traffic could read them.
  SSH encrypts the session instead. The only cost is a bit of extra setup
  (generating RSA keys, setting a domain name) — worth it for not exposing
  passwords.
- **Set port security to `restrict` instead of `shutdown`.** If an
  unauthorized device plugs into a port, `restrict` blocks its traffic and
  logs the attempt, but leaves the port itself running. `shutdown` would
  disable the port entirely, meaning someone has to manually turn it back
  on. For ordinary user ports, restrict keeps things secure without
  creating unnecessary extra work.
- **Left port security off the switch-to-router uplink ports.** Each
  switch's FastEthernet0/1 connects back to the router, not to a single PC
  — port security is meant for ports where exactly one device's MAC address
  should ever appear. Turning it on for an uplink could block legitimate
  traffic, so it's intentionally left out there.

## Key Configuration

**VLSM-derived interface addressing (R1-HQ):**
```
interface GigabitEthernet0/0
 ip address 192.168.10.65 255.255.255.192
interface GigabitEthernet0/1
 ip address 192.168.10.129 255.255.255.224
```

**Summarized static route (R2-Branch):**
```
ip route 192.168.10.64 255.255.255.128 192.168.10.185
```

**SSH-only VTY access (all routers/switches):**
```
line vty 0 4
 transport input ssh
 login local
 exec-timeout 5 0
```

**Port security on access ports (all switches):**
```
switchport port-security
switchport port-security maximum 1
switchport port-security violation restrict
switchport port-security mac-address sticky
spanning-tree portfast
spanning-tree bpduguard enable
```

## Verification

## PC-IT1> ping 192.168.10.65

<img width="414" height="229" alt="image" src="https://github.com/user-attachments/assets/bcaefd4d-0744-4f03-8129-95d91f6b3fdd" />

## R2-Branch# show ip route

<img width="472" height="533" alt="image" src="https://github.com/user-attachments/assets/fc25a507-9057-4c3c-8329-ae96cc21cc01" />

## PC-HR2> nslookup www.technova.com

<img width="475" height="223" alt="image" src="https://github.com/user-attachments/assets/9bcce9e3-f475-4b1c-bd86-4624bf8dbe3f" />
