# CCNA Lab 3 – Floating Static Routing

## Overview

This lab builds on static routing concepts by introducing **floating static routes** — backup routes that only become active when the primary route fails. A fourth router (R-4) is added to the topology to provide an alternate path, and Administrative Distance (AD) values are used to control route preference.

**Tool:** Cisco Packet Tracer  
**Lab File:** `lab_3.pkt`

---

## Objectives

- Configure a 4-router topology with static routing
- Implement floating static routes using a higher AD value (20)
- Verify failover behavior when the primary link goes down
- Confirm end-to-end connectivity via ping tests from PCs

---

## Network Topology

```
                    R-4 (ISR4331)
                Se0/1/1     Se0/1/0
               /  14.0.0.0/24  \  34.0.0.0/24
              /                  \
R-1 (ISR4331)    12.0.0.0/8    R-2 (ISR4331)    23.0.0.0/24    R-3 (ISR4331)
  Se0/1/0 ─────────────────── Se0/1/0          Se0/1/1 ──────── Se0/1/1
  Gig0/0/0                    Gig0/0/0                           Gig0/0/0
     |                            |                                  |
  SW-1 (Fa0/7)               SW-2 (Fa0/6)                      SW-3 (Fa0/6)
 192.168.10.5               192.168.20.5                       192.168.30.5
     |                            |                                  |
192.168.10.0/24             192.168.20.0/24                   192.168.30.0/24
 PC0, PC1, PC2               PC3, PC4, PC5                    PC6, PC7, PC8
```

---

## IP Addressing

| Device | Interface        | IP Address      | Subnet Mask     |
|--------|-----------------|-----------------|-----------------|
| R-1    | Gig0/0/0        | 192.168.10.5    | 255.255.255.0   |
| R-1    | Serial0/1/0     | 12.0.0.1        | 255.0.0.0       |
| R-1    | Serial0/1/1     | 14.0.0.1        | 255.0.0.0       |
| R-2    | Gig0/0/0        | 192.168.20.5    | 255.255.255.0   |
| R-2    | Serial0/1/0     | 12.0.0.2        | 255.0.0.0       |
| R-2    | Serial0/1/1     | 23.0.0.1        | 255.0.0.0       |
| R-3    | Gig0/0/0        | 192.168.30.5    | 255.255.255.0   |
| R-3    | Serial0/1/1     | 23.0.0.2        | 255.0.0.0       |
| R-3    | Serial0/1/0     | 34.0.0.2        | 255.0.0.0       |
| R-4    | Serial0/1/1     | 14.0.0.2        | 255.0.0.0       |
| R-4    | Serial0/1/0     | 34.0.0.1        | 255.0.0.0       |

---

## Router Configurations

### R-1

```
hostname R-1
!
interface GigabitEthernet0/0/0
 no shutdown
 ip address 192.168.10.5 255.255.255.0
!
interface Serial0/1/0
 no shutdown
 ip address 12.0.0.1 255.0.0.0
!
interface Serial0/1/1
 no shutdown
 ip address 14.0.0.1 255.0.0.0
!
! Primary static routes
ip route 192.168.20.0 255.255.255.0 12.0.0.2
ip route 192.168.30.0 255.255.255.0 12.0.0.2
!
! Floating static route (backup via R-4, AD = 20)
ip route 192.168.30.0 255.255.255.0 14.0.0.2 20
```

### R-2

```
hostname R-2
!
interface GigabitEthernet0/0/0
 no shutdown
 ip address 192.168.20.5 255.255.255.0
!
interface Serial0/1/0
 no shutdown
 ip address 12.0.0.2 255.0.0.0
!
interface Serial0/1/1
 no shutdown
 ip address 23.0.0.1 255.0.0.0
!
ip route 192.168.10.0 255.255.255.0 12.0.0.1
ip route 192.168.30.0 255.255.255.0 23.0.0.2
```

### R-3

```
hostname R-3
!
interface GigabitEthernet0/0/0
 no shutdown
 ip address 192.168.30.5 255.255.255.0
!
interface Serial0/1/1
 no shutdown
 ip address 23.0.0.2 255.0.0.0
!
interface Serial0/1/0
 no shutdown
 ip address 34.0.0.2 255.0.0.0
!
! Primary static routes
ip route 192.168.10.0 255.255.255.0 23.0.0.1
ip route 192.168.20.0 255.255.255.0 23.0.0.1
!
! Floating static route (backup via R-4, AD = 20)
ip route 192.168.10.0 255.255.255.0 34.0.0.1 20
```

### R-4

```
hostname R-4
!
interface Serial0/1/1
 no shutdown
 ip address 14.0.0.2 255.0.0.0
!
interface Serial0/1/0
 no shutdown
 ip address 34.0.0.1 255.0.0.0
!
ip route 192.168.10.0 255.255.255.0 14.0.0.1
ip route 192.168.30.0 255.255.255.0 34.0.0.2
```

---

## What Is a Floating Static Route?

A floating static route is a **backup route** configured with a higher Administrative Distance than the primary route. The router only installs it in the routing table if the primary route disappears (e.g., due to a link failure).

**Default static route AD = 1**  
**Floating static route AD = 20** (in this lab)

Because AD 20 > AD 1, the floating route stays hidden while the primary is up. When the primary link fails, the floating route automatically takes over.

---

## Routing Tables

### R-1 – Normal Operation (both links up)

```
C    12.0.0.0/8 is directly connected, Serial0/1/0
C    14.0.0.0/8 is directly connected, Serial0/1/1
C    192.168.10.0/24 is directly connected, GigabitEthernet0/0/0
S    192.168.20.0/24 [1/0] via 12.0.0.2
S    192.168.30.0/24 [1/0] via 12.0.0.2
                     [1/0] via 14.0.0.2   ← floating route hidden (same AD shown when primary has two paths)
```

### R-1 – After R-2 Link Goes Down

```
C    14.0.0.0/8 is directly connected, Serial0/1/1
C    192.168.10.0/24 is directly connected, GigabitEthernet0/0/0
S    192.168.30.0/24 [20/0] via 14.0.0.2   ← floating route now active!
```

The route to 192.168.20.0/24 disappears (no backup was configured for it), while the floating route to 192.168.30.0/24 via R-4 activates with AD 20.

---

## Failover Test

**Simulating failure:** R-2's Serial0/1/0 interface was shut down with:
```
R-2(config)# interface serial0/1/0
R-2(config-if)# shutdown
```

**Result on R-2's routing table after shutdown:**
- The 12.0.0.0 network (link to R-1) disappeared
- R-2 retained only its connection to R-3 via 23.0.0.1
- R-2 still reached 192.168.30.0/24 via 23.0.0.2

---

## Ping Verification

### From R-1 side PC (192.168.10.x) — pinging R-2 and R-3 networks

| Destination      | Result         | Notes                     |
|-----------------|----------------|---------------------------|
| 192.168.20.3    | 3/4 received   | 25% loss (first packet ARP) |
| 192.168.30.2    | 3/4 received   | 25% loss (first packet ARP) |

### From R-2 side PC (192.168.20.x) — pinging R-3 network

| Destination      | Result         |
|-----------------|----------------|
| 192.168.30.1    | 3/4 received   |
| 192.168.30.2    | 4/4 received   |
| 192.168.30.3    | 3/4 received   |

### From R-3 side PC (192.168.30.x) — pinging R-1 and R-2 networks

| Destination      | Result         |
|-----------------|----------------|
| 192.168.20.1    | 4/4 received   |
| 192.168.10.2    | 3/4 received   |
| 192.168.10.3    | 3/4 received   |

> **Note:** The first ping packet timing out is normal in Packet Tracer. It occurs because the router needs to resolve the next-hop MAC address via ARP before forwarding the first packet. Subsequent packets succeed normally.

---

## Key Concepts Demonstrated

| Concept | Description |
|--------|-------------|
| **Floating Static Route** | Backup route with higher AD that activates on primary failure |
| **Administrative Distance** | Metric used to prefer one route source over another (lower = more preferred) |
| **Failover** | Automatic rerouting when a link goes down |
| **Load balancing** | When two routes have equal AD/metric, traffic is shared (seen briefly on R-1 with two paths to 192.168.30.0) |

---

## Lab Files

| File | Description |
|------|-------------|
| `lab_3.pkt` | Cisco Packet Tracer topology file |
| Screenshots | Router configs, routing tables, ping tests, failover verification |

---

## Screenshots

| Screenshot | Description |
|-----------|-------------|
| `floating_static_routing.png` | Full network topology |
| `R-1_configuration.png` | R-1 interface and hostname setup |
| `R-1_ip_interface_brief.png` | R-1 interface status |
| `R-1_ip_route.png` | R-1 routing table (normal + floating route visible) |
| `R-1_changing_AD_value.png` | Adding floating static route with AD 20 |
| `R-1_ip_route_when_R-2_link_down.png` | R-1 routing table after failover |
| `R-1_pc_ping_command.png` | Ping from R-1 LAN to remote networks |
| `R-2_configuration.png` | R-2 interface and static route setup |
| `R-2_ip_interface_brief.png` | R-2 interface status |
| `R-2_ip_route.png` | R-2 routing table |
| `R-2_link_down_from_R-1.png` | R-2 Serial0/1/0 shut down (failover trigger) |
| `R-2_pc_ping_command.png` | Ping from R-2 LAN to R-3 network |
| `R-3_configuration.png` | R-3 interface and static route setup |
| `R-3_ip_interface_brief.png` | R-3 interface status |
| `R-3_ip_route.png` | R-3 routing table |
| `R-3_pc_ping_command.png` | Ping from R-3 LAN to remote networks |
| `R-4_configuration.png` | R-4 interface setup (backup router) |
| `R-4_interface_brief.png` | R-4 interface status |
| `R-4_ip_route.png` | R-4 routing table |

---

*Part of my CCNA self-study lab series — practiced in Cisco Packet Tracer.*
