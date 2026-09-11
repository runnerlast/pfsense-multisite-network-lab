# Multi-Site pfSense Network Lab

A virtualised pfSense-centric lab implementing firewall redundancy, multi-protocol VPNs, traffic shaping, and centralised telemetry

# Overview

This project details the step-by-step design, configuration, and validation of a multi-site network environment. The entire topology is fully virtualized within VMware Workstation, simulating a central headquarters (HQ) connected to a remote branch site over a simulated ISP edge router. Core capabilities include an active/passive pfSense CARP cluster with pfsync session synchronization, 802.1Q VLAN trunking managed by an OpenWRT switch, and a hybrid VPN overlay combining Site-to-Site IPsec with NAT-T, split-tunnel WireGuard, and full-tunnel OpenVPN. Network performance is managed using hierarchical traffic shaping alongside dedicated guest rate limiters, while centralized observability is delivered through a containerized telemetry stack running Loki, Promtail, and Grafana for remote syslog aggregation.

* * *

# Table of Contents

- [Multi-Site pfSense Network Lab](#multi-site-pfsense-network-lab)
- [Overview](#overview)
- [Table of Contents](#table-of-contents)
- [Network Topology](#network-topology)
    - [Logical Structure](#logical-structure)
    - [Subnets, VLANs and Addressing](#subnets-vlans-and-addressing)
    - [Network Diagrams](#network-diagrams)
        - [Data plane topology](#data-plane-topology)
        - [Management plane topology](#management-plane-topology)
- [Implementation Roadmap](#implementation-roadmap)
- [Documentation Index](#documentation-index)

* * *

# Network Topology

## Logical Structure

**Data Plane:** The data plane models upstream ISP transit through a MikroTik RouterOS CHR edge router performing 1:1 NAT mapping to pfSense WAN interfaces. High availability at HQ is deployed using CARP VIPx shared between primary and secondary nodes, with internal client traffic segmented into VLAN 10 (INTERNAL), VLAN 20 (DMZ), and VLAN 30 (GUEST) across an OpenWRT node deployed as an 802.1Q trunk switch. The data plane also routes Site-to-Site IPsec traffic to the Branch site and terminates remote access WireGuard and OpenVPN connections.

**Management Plane:** The management plane provides dedicated out-of-band network administration across isolated HQ and Branch subnets. Inter-site management traffic is routed across a Phase 2 IPsec child SA, enabling remote syslogs from all network devices to reach the central Grafana, Loki, and Promtail telemetry server.

* * *

## Subnets, VLANs and Addressing

| Interface/Zone | Subnet | Gateway/VIP | Host assignments | Notes |
| --- | --- | --- | --- | --- |
| SYNC | 10.0.0.0/30 | N/A | Pri: .1, Sec: .2 | Sync connection |
| WAN (HQ) | 172.16.1.0/29 | CHR: 172.16.1.1 | VIP: .2, Pri: .3, Sec: .4 | NAT: 198.51.100.10 |
| INTERNAL/VLAN10 | 10.0.10.0/24 | CARP VIP: 10.0.10.1 | Internal clients (DHCP) | Trunked connection |
| DMZ/VLAN20 | 10.0.20.0/24 | CARP VIP: 10.0.20.1 | DMZ servers (DHCP) | Trunked connection |
| GUEST/VLAN30 | 10.0.30.0/24 | CARP VIP: 10.0.30.1 | Rate-limited guest clients (DHCP) | Trunked connection |
| WAN (BRANCH) | 172.16.2.0/30 | CHR: 172.16.2.1 | Branch FW: 172.16.2.2 | NAT: 198.51.100.20 |
| BRANCH | 10.0.40.0/24 | Branch FW: 10.0.40.1 | Branch clients/workstations (DHCP) | —   |
| REMOTE_WG | 10.0.50.0/24 | Gateway: 10.0.50.1 | Remote WireGuard client: .100 | Split-Tunnel VPN |
| REMOTE_OVPN | 10.0.51.0/24 | Gateway: 10.0.51.1 | Remote OpenVPN client: .2 | Full-Tunnel VPN |
| MGMT (HQ) | 10.0.100.0/24 | CARP VIP: 10.0.100.2 | Host: .1, Pri: .3, Sec: .4, OpenWRT: .5, NMS: .6 | Out-of-band |
| MGMT (BRANCH) | 10.0.101.0/24 | Branch FW: 10.0.101.2 | Host: .1 | Out-of-band |

<div class="joplin-table-wrapper">

* * *

## Network Diagrams

### **Data Plane Topology**

<img src="./docs/diagram1.png" alt="diagram1.png" width="752" height="318" class="jop-noMdConv">

### **Management Plane Topology**

</div><div class="joplin-table-wrapper"><img src="./docs/diagram2.png" alt="diagram2.png" width="449" height="228" class="jop-noMdConv"></div>

* * *

# Implementation Roadmap

1.  **RouterOS ISP Base:** Configured a MikroTik CHR instance in VMware to act as an upstream ISP, establishing public WAN routing, static route distribution, and 1:1 NAT mappings (198.51.100.10 for HQ, 198.51.100.20 for Branch) to simulate real-world internet transit.
2.  **Basic Firewall Routing:** Set up single pfSense nodes at HQ (HQ-Primary) and Branch with static WAN/LAN addresses, establishing baseline default gateways, stateful firewall rules, and upstream connectivity back to RouterOS.
3.  **Site-to-Site IPsec:** Configured Phase 1 IKEv2 proposals and Phase 2 child security associations between HQ and Branch across the RouterOS WAN, validating NAT-Traversal (NAT-T), and inter-site connectivity.
4.  **HQ 802.1Q VLAN Trunking:** Deployed an OpenWRT virtual switch with 802.1Q tagging to segment HQ into VLAN 10 (INTERNAL), VLAN 20 (DMZ), and VLAN 30 (GUEST), verifying traffic isolation rules on HQ-Primary prior to clustering.
5.  **pfSense CARP High Availability:** Added HQ-Secondary to form an Active/Passive cluster, enabling pfsync over an isolated sync network, firewall rules and configuration synchronization, and shared CARP Virtual IPs across WAN and internal subnets.
6.  **Traffic Shaping & Failover Validation:** Built hierarchical queues for priority traffic alongside dedicated limiters for guest isolation, then executed mid-stream primary node terminations to validate stateful failover.
7.  **Remote Access VPN:** Configured a split-tunnel WireGuard server (10.0.50.0/24) and a full-tunnel OpenVPN server (10.0.51.0/24) bound to HQ's WAN CARP VIP (172.16.1.2), ensuring remote client session persistence during CARP switchovers.
8.  **Telemetry & Syslog Ingestion:** Deployed a Docker-based monitoring stack running Grafana, Loki, and Promtail on HQ MGMT subnet, configuring Promtail to aggregate remote log streams from pfSense nodes.

* * *

# **Documentation Index**

- [Section 1: Host Resourcing & Virtual Network Architecture](./docs/Section1.md)
- [Section 2: Core Routing & Baseline Node Setup](./docs/Section2.md)
- [Section 3: Site-to-Site IPsec VPN](./docs/Section3.md)
- [Section 4: 802.1Q VLAN Segmentation](./docs/Section4.md)
- [Section 5: CARP High Availability Clustering](./docs/Section5.md)
- [Section 6: Hierarchical Traffic Shaping & QoS](./docs/Section6.md)
- [Section 7: Remote Access VPNs (WireGuard & OpenVPN) & QoS Extension](./docs/Section7.md)
- [Section 8: Centralized Logging (Loki & Grafana)](./docs/Section8.md)
* * *

# Repository Structure

```
pfsense-multisite-network-lab/
├── .gitignore
├── assets/
├── configs/
│   ├── chr/
│   ├── openwrt/
│   └── pfsense/
├── docs/
├── README.md
```
[Next Section ->](./docs/Section1.md)
