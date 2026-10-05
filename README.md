# networking-portfolio

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

<img width="960" height="501" alt="Screenshot 2026-10-04 112434" src="https://github.com/user-attachments/assets/ded038f7-87ac-4097-bb10-b94b39ccdb88" />


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

| Device/Interface | IP Address | Subnet Mask | Network/VLAN |
|---|---|---|---|
| R1-HQ Gi0/0 | 192.168.10.65 | 255.255.255.192 (/26) | IT LAN — 192.168.10.64/26 |
| R1-HQ Gi0/1 | 192.168.10.129 | 255.255.255.224 (/27) | Sales LAN — 192.168.10.128/27 |
| R1-HQ Gi0/2 | 192.168.10.161 | 255.255.255.240 (/28) | HR LAN — 192.168.10.160/28 |
| R1-HQ Fa0/1/0 | 192.168.10.177 | 255.255.255.248 (/29) | Server Farm — 192.168.10.176/29 |
| R1-HQ Se0/0/0 | 192.168.10.185 | 255.255.255.252 (/30) | WAN — 192.168.10.184/30 |
| R2-Branch Gi0/0 | 192.168.10.1 | 255.255.255.192 (/26) | Branch LAN — 192.168.10.0/26 |
| R2-Branch Se0/0/0 | 192.168.10.186 | 255.255.255.252 (/30) | WAN — 192.168.10.184/30 |
| SW-IT Vlan1 | 192.168.10.126 | 255.255.255.192 (/26) | IT LAN |
| SW-Sales Vlan1 | 192.168.10.158 | 255.255.255.224 (/27) | Sales LAN |
| SW-HR Vlan1 | 192.168.10.174 | 255.255.255.240 (/28) | HR LAN |
| SW-ServerFarm Vlan1 | 192.168.10.182 | 255.255.255.248 (/29) | Server Farm |
| SW-Branch Vlan1 | 192.168.10.62 | 255.255.255.192 (/26) | Branch LAN |
| PC-IT1 / PC-IT2 | 192.168.10.66 / .67 | 255.255.255.192 (/26) | IT LAN |
| PC-Sales1 / PC-Sales2 | 192.168.10.130 / .131 | 255.255.255.224 (/27) | Sales LAN |
| PC-HR1 / PC-HR2 | 192.168.10.162 / .163 | 255.255.255.240 (/28) | HR LAN |
| PC-Branch1 / PC-Branch2 | 192.168.10.2 / .3 | 255.255.255.192 (/26) | Branch LAN |
| Server-DNS | 192.168.10.178 | 255.255.255.248 (/29) | Server Farm |

## Design Decisions

- **VLSM instead of one fixed mask for every segment.** A single `/26` for
  all five segments would need five separate 64-address blocks (320
  addresses) for a design that only needs ~153 host addresses. VLSM sizes
  each block to its actual requirement, so the whole design fits in 188 of
  256 addresses in the original `/24`.
- **Largest-to-smallest allocation order.** Subnets were carved out starting
  with the biggest host requirement (Branch, 60 hosts) down to the smallest
  (WAN, 2 hosts). Allocating smallest-first risks stranding a later large
  requirement in space a smaller subnet already fragmented.
- **One summarized static route on R2-Branch instead of four.** IT, Sales,
  HR, and the Server Farm all nest inside `192.168.10.64/25`, so a single
  `ip route 192.168.10.64 255.255.255.128 192.168.10.185` replaces four
  separate `/26`–`/29` routes. The alternative — keeping all four individual
  routes — was rejected because it scales badly: every new HQ subnet would
  need its own route on R2, where the summarized line already covers any
  future subnet carved from that same /25.
- **Static routing instead of a dynamic routing protocol.** With only two
  routers and a stable topology, a protocol like OSPF or EIGRP would add
  configuration and convergence overhead with no real benefit here. Static
  routing was chosen deliberately as the right-sized solution for a
  two-router WAN link; this is revisited if the topology grows.
- **SSH-only remote access, Telnet disabled.** `transport input ssh` on the
  VTY lines was chosen over leaving Telnet available, because Telnet sends
  credentials in plaintext. The trade-off is that SSH requires RSA key
  generation and a domain name on every device, which is a small one-time
  setup cost against a real credential-exposure risk.
- **Port security with `restrict` rather than `shutdown` as the violation
  action.** `restrict` drops offending traffic and logs it without taking
  the port down, so a MAC-address violation doesn't require a network
  engineer to manually re-enable the port — the trade-off against
  `shutdown` is a slightly weaker response to a genuine intrusion attempt,
  judged acceptable for end-user access ports on an internal LAN.
- **Switch uplink ports excluded from port security.** FastEthernet0/1 on
  every switch (the uplink to its router) intentionally has no port
  security or PortFast, since it's a trunk-capable infrastructure link, not
  an end-user access port — applying port-security there would be
  incorrect and could block legitimate multi-MAC uplink traffic.

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

Full configs for every device are in [`configs/`](configs/).

## Verification

> Representative output shown below; full raw log in
> [`verification.md`](verification.md).

```
PC-IT1> ping 192.168.10.65
<placeholder — paste real captured output>

R2-Branch# show ip route
<placeholder — paste real captured output, confirm S 192.168.10.64/25>

PC-IT1> nslookup www.technova.com
<placeholder — paste real captured output, should resolve to 192.168.10.178>
```

## Challenges & Lessons Learned

> **Placeholder.** This section is meant to hold the real troubleshooting
> narrative from building and testing the topology — e.g. DCE/DTE clock
> rate mismatches on the serial link, a port-security violation locking a
> port before `sticky` learned the right MAC, or a VTY login failing until
> `ip domain-name` + `crypto key generate rsa` were run in the right order.
> Replace this note with what actually happened once the lab is built.

## Files

- [`project.pkt`](project.pkt) — *placeholder: add the saved Packet Tracer
  file here after building*
- [`topology.png`](topology.png)
- [`configs/`](configs/)
- [`verification.md`](verification.md)
