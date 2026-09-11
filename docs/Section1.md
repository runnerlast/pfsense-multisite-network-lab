# Section 1. Host Resourcing & Virtual Network Architecture

This section details the cost of each node and the usage of VMnet connections.

* * *

### VM Resourcing

Nodes were sized to the minimum viable footprint for its role, keeping within the host's limited resources.

| Node | vCPU | RAM | Disk | NICs |
| --- | --- | --- | --- | --- |
| RouterOS-CHR | 1   | 256MB | 128MB | 3 (ether1: HQ WAN, ether2: BRANCH WAN, ether3: Bridged) |
| pfSense-HQ-Primary | 1   | 1GB | 5GB | 4 (em0: WAN, em1: MGMT, em2: SYNC, em3: TRUNK) |
| pfSense-HQ-Secondary | 1   | 1GB | 5GB | 4 (em0: WAN, em1: MGMT, em2: SYNC, em3: TRUNK) |
| pfSense-Branch | 1   | 1GB | 5GB | 3 (em0: WAN, em1: MGMT, em2: BRANCH) |
| OpenWRT | 1   | 256MB | 128MB | 5 (eth0: MGMT, eth1: TRUNK, eth2: INTERNAL, eth3: DMZ, eth4: GUEST) |
| Grafana/Loki NMS | 2   | 2GB | 10GB | 1 (eth0: MGMT) |

* * *

### vmnet Resourcing

Isolated vmnets were used to segment WAN links, trunked VLAN traffic, and out-of-band management from each other.

| vmnet # | Subnet | VM interfaces | Notes |
| --- | --- | --- | --- |
| vmnet0 | 192.168.100.0/24 | Host, CHR ether3 | Bridged connection, internet egress |
| vmnet1 | 172.16.1.0/29 | CHR ether1, HQ-Primary WAN, HQ-Secondary WAN | Simulated ISP link to HQ |
| vmnet2 | 172.16.2.0/30 | CHR ether2, Branch WAN | Simulated ISP link to Branch |
| vmnet3 | 10.0.0.0/30 | HQ-Primary SYNC, HQ-Secondary SYNC |     |
| vmnet4 | N/a | HQ-Primary TRUNK, HQ-Secondary TRUNK, OpenWRT eth1 | Shared 802.1Q trunk, VLAN 10/20/30 |
| vmnet5 | 10.0.10.0/24 | OpenWRT eth2, INTERNAL clients | VLAN10 access segment |
| vmnet6 | 10.0.20.0/24 | OpenWRT eth3, DMZ clients | VLAN20 access segment |
| vmnet7 | 10.0.30.0/24 | OpenWRT eth4, GUEST clients | VLAN30 access segment |
| vmnet8 | 10.0.40.0/24 | Branch clients |     |
| vmnet9 | 10.0.100.0/24 | Host, HQ-Primary/Secondary MGMT, OpenWRT eth0, Monitoring VM | Out-of-band, HQ |
| vmnet10 | 10.0.101.0/24 | Host, Branch MGMT | Out-of-band, Branch |

[<- Previous Section](./README.md) | [Next Section ->](./Section2.md)