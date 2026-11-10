# Enterprise Data Centre Security: Spine-Leaf Lab

A Cisco Packet Tracer lab of a small enterprise data centre built on a spine-leaf fabric, with security controls layered on top: tiered segmentation, an ACL that protects the database tier, restricted administrative access, and a fabric that keeps working when a spine fails.

This repository contains two files:

| File | Purpose |
|---|---|
| `SpineLeaf.pkt` | The Packet Tracer build (open in Cisco Packet Tracer). |
| `Spine-leaf_data_centre_lab.html` | An interactive browser copy of the lab. Open it in any browser, pick a test from the menu, and watch packets cross the fabric. No install needed. |

> **Scope note.** This README is written from the interactive HTML copy, which reproduces the lab's topology, addressing, and command output. Details marked *(inferred)* are deduced from that output rather than read from the device configs. Check them against `SpineLeaf.pkt` before relying on them.

---

## 1. Topology

14 devices, one OSPF area (area 0).

```
                        Edge-Router (Cisco 2911)
                       /        |             \
            Campus-SW           |              \
           (Catalyst 2960)      |               \
            /        \      Spine-01  ------  Spine-02      (3560-24PS each)
   Employee-PC-1  Employee-PC-2   \  \  \     /  /  /
                                   \  \  \   /  /  /       every leaf connects to both spines
                           Leaf-01   Leaf-02   Leaf-03
                              |       /    \     /    \
                         Web-Server App-Srv DB-Srv  Services-Srv  Admin-PC
```

**Fabric links.** The edge router connects to both spines. Each of the three leaf switches connects to both spines. Every fabric link is a routed /30 point-to-point link running OSPF, so the fabric has no spanning-tree dependency and no Layer 2 loops.

**Campus side.** Employee PCs reach the data centre through Campus-SW and the edge router, as DHCP clients on `192.168.10.0/24`.

## 2. Addressing and segmentation

Each tier lives in its own VLAN and subnet, and is attached to a specific leaf.

| Tier | Leaf | VLAN | Subnet | Host |
|---|---|---|---|---|
| Web | Leaf-01 | 10 | `10.10.10.0/24` | Web-Server `10.10.10.11` |
| Application | Leaf-02 | 20 | `10.10.20.0/24` | App-Server `10.10.20.11` |
| Database | Leaf-02 | 30 | `10.10.30.0/24` | DB-Server `10.10.30.11` |
| Services (DHCP and DNS) | Leaf-03 | 40 | `10.10.40.0/24` | Services-Server `10.10.40.11` |
| Management | Leaf-03 | 99 | `10.99.99.0/24` | Admin-PC `10.99.99.10`, Leaf-03 at `10.99.99.1` |
| Employees | Campus-SW | n/a | `192.168.10.0/24` | DHCP pool, gateway `192.168.10.1` |

Fabric /30 links *(inferred from route and neighbour output)*: `10.0.0.0/30` (router to Spine-01), `10.0.0.4/30` (router to Spine-02), `10.1.1.0/24` range for Spine-01 to leaves, `10.1.2.0/24` range for Spine-02 to leaves.

## 3. Security controls

### 3.1 Tier segmentation
Web, application, database, services, and management traffic each sit in a separate VLAN and subnet. Traffic between tiers is routed, which means it passes a point where policy can be applied. The database tier is the most sensitive, so it gets the strictest policy.

### 3.2 Database protection: `PROTECT-DB`
An extended ACL named `PROTECT-DB` is applied on Leaf-02 for the `Vlan30` interface:

```
Extended IP access list PROTECT-DB
    10 permit ip 10.10.20.0 0.0.0.255 10.10.30.0 0.0.0.255
    20 permit ip host 10.99.99.10 10.10.30.0 0.0.0.255
    30 deny ip any any
```

| Source | Result | Why |
|---|---|---|
| App tier (`10.10.20.0/24`) | Permitted | The application needs its database. |
| Admin-PC (`10.99.99.10`) | Permitted | Single management host. |
| Everything else, including employee PCs | **Denied** | Explicit `deny ip any any`. |

### 3.3 Restricted administrative access
SSH to the switches is limited by a VTY access-class that permits only `10.99.99.10` (Admin-PC). An employee PC can reach Leaf-03 on the network but the session is refused.

### 3.4 Resilience as a security property
Availability is part of security. Every leaf has two equal-cost paths (one per spine), and OSPF installs both next hops (ECMP). If Spine-01 is shut down, OSPF reconverges and traffic moves to Spine-02. The lab shows a brief loss of a few packets during convergence, then recovery.

### 3.5 DHCP relay instead of a flat network
The DHCP server sits in the services segment, not on the employee network. The edge router uses `ip helper-address` to relay the broadcast Discover as unicast to `10.10.40.11`, which avoids stretching a Layer 2 domain across the fabric.

## 4. Running the tests

Open `Spine-leaf_data_centre_lab.html`, pick a test from the menu (or select a device and type the command), and use `help` on any device to list the commands it accepts. **Reset lab** restores the starting state.

| # | Device | Command | What it demonstrates | Expected result |
|---|---|---|---|---|
| 1 | Employee-PC-1 | `ping 10.10.10.11` | Reachability from campus to the web tier | Replies; the first request may time out while ARP resolves |
| 2 | Employee-PC-1 | `ipconfig /renew` | DHCP relay | Lease: `192.168.10.100/24`, gateway `.1`, DNS `10.10.40.11` |
| 3 | Employee-PC-1 | `browser http://www.company.local` | DNS then HTTP across the fabric | `200 OK` from `10.10.10.11` (needs test 2 first) |
| 4 | Spine-01 | `show ip ospf neighbor` | OSPF adjacencies | Four `FULL` neighbours: router plus three leaves |
| 5 | Leaf-01 | `show ip route ospf` | ECMP | Two equal-cost next hops, one per spine |
| 6a | Employee-PC-1 | `ping 10.10.30.11` | ACL blocks non-app hosts | `Destination host unreachable` |
| 6b | App-Server | `ping 10.10.30.11` | ACL permits the app tier | Replies |
| 6c | Leaf-02 | `show access-lists` | ACL hit counters | Permit and deny match counts increase |
| 7a | Admin-PC | `ssh -l admin 10.99.99.1` | Permitted management host | Login prompt |
| 7b | Employee-PC-1 | `ssh -l admin 10.99.99.1` | Blocked host | `Connection refused by remote host` |
| 8 | Employee-PC-1 | `ping -t 10.10.10.11`, then Spine-01 `shutdown` | Spine failover | Short loss, then replies resume via Spine-02 |
| 8 | Spine-01 | `no shutdown` | Recovery | Adjacencies return to `FULL`, ECMP resumes |
| Extra | Leaf-02 | `show vlan brief` | VLAN 20 and 30 membership | VLAN 20 on Fa0/1, VLAN 30 on Fa0/2 |

**Suggested walkthrough:** run tests 2, 1, 3, then 6a and 6b to compare, then 6c to see the counters, then 7a and 7b. Finish with test 8: start `ping -t`, shut down Spine-01, watch the loss, then bring it back.

## 5. Limits of this lab and recommended hardening

The lab shows segmentation, ACL enforcement, restricted management, and redundancy. A production enterprise data centre needs more. The items below are **not demonstrated in this lab** and are recommendations, not descriptions of the current build.

**Routing and fabric**
- Enable OSPF authentication (MD5 or SHA) on every fabric link so a rogue device cannot form an adjacency.
- Set non-fabric interfaces to `passive-interface` by default.
- Add BFD for faster failure detection than OSPF timers alone.

**Layer 2 and access**
- Enable DHCP snooping, Dynamic ARP Inspection, and IP Source Guard on access ports.
- Use port security or 802.1X on edge ports; shut down unused ports and move them to an unused VLAN.
- Enable BPDU Guard and Root Guard on host-facing ports.

**Policy**
- Place a stateful firewall between campus and the data centre, and between tiers, rather than relying on stateless ACLs alone. The ACLs here are stateless, so return-traffic handling and application-aware rules need a firewall.
- Apply equivalent ACLs to the web and services tiers (only `PROTECT-DB` is shown here), and restrict which ports are allowed (for example, TCP 80/443 to web, DNS and DHCP to services) instead of `permit ip`.
- Add `log` to deny entries and send logs to a central collector.

**Management plane**
- Use SSH version 2 only, AAA with TACACS+ or RADIUS, and a dedicated out-of-band management network.
- Add login banners, exec timeouts, and rate limits on VTY lines. Disable Telnet and HTTP management.
- Back up configurations and track changes in version control.

**Monitoring**
- Enable NTP, syslog, SNMPv3, and NetFlow or IPFIX for visibility.

## 6. Requirements

- **Interactive copy:** any modern browser. No dependencies or network access are needed. It respects the reduced-motion setting and light or dark mode.
- **Packet Tracer build:** Cisco Packet Tracer (open `SpineLeaf.pkt`). Use a Packet Tracer version recent enough to support Catalyst 3560 and 2960 switches and the 2911 router.
