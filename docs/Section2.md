# Section 2. Core Routing & Baseline Node Setup

This section details the initial setup and baseline configurations for RouterOS-CHR (simulated ISP edge), pfSense-HQ-Primary, and pfSense-Branch.

* * *

### Set up RouterOS-CHR

**1\. Bring up ether3 and license CHR**

From the console, enter the following commands:

```
/ip dhcp-client add interface=ether3 disabled=no
/system license renew		# MikroTik.com login, level p1
/system license print		# Confirm CHR is licensed
```

**2\. Assign interface addresses**

```
/ip address add address=172.16.1.1/30 interface=ether1
/ip address add address=172.16.2.1/30 interface=ether2
/ip address add address=198.51.100.1/28 interface=ether1
```

The 198.51.100.1/28 address serves as the cosmetic public IP presence on ether1, not routed. ether3 gets its address from DHCP.

**3\. NAT 1:1 per site + internet egress**

```
/ip firewall nat add chain=dstnat dst-address=198.51.100.10 action=dst-nat to-addresses=172.16.1.2 comment="to HQ"
/ip firewall nat add chain=srcnat src-address=172.16.1.2 out-interface=ether2 action=src-nat to-addresses=198.51.100.10 comment="from HQ"
/ip firewall nat add chain=dstnat dst-address=198.51.100.20 action=dst-nat to-addresses=172.16.2.2 comment="to Branch"
/ip firewall nat add chain=srcnat src-address=172.16.2.2 out-interface=ether1 action=src-nat to-addresses=198.51.100.20 comment="from Branch"
/ip firewall nat add chain=srcnat out-interface=ether3 action=masquerade comment="internet egress"
```

out-interface on the two site srcnat rules is required, without it they match on source address alone and hijack internet-bound traffic before it reaches the masquerade rule.

**Default route check:**

```
/ip route print   # confirm 0.0.0.0/0 via ether3's DHCP gateway
```

* * *

### Set up pfSense-HQ/pfSense-Branch

**1\. Assign WAN and LAN (MGMT) addresses**

From the console -> option 2 (Set interface IP addresses):

- HQ: WAN (em0): 172.16.1.3/29 GW: 172.16.1.1, LAN (will be MGMT, em1): 10.0.100.3/24, OPT1 (will be TRUNK, but is used initally as LAN subnet for internal HQ clients): 10.0.10.2/24
- Branch: WAN (em0): 172.16.2.2/30 GW: 172.16.2.1, LAN (will be MGMT, em1): 10.0.101.2/24, OPT1 (will be Branch): 10.0.40.1/24

**2\. Add WAN rule exceptions**

Log in from the initial LAN address and complete the initial setup. On both FWs: Interfaces -> WAN -> uncheck "Block private networks" and "Block bogon networks" -> Save/Apply -> reboot. This is required because WAN legitimately sits in RFC1918 space here, without it, WAN doesn't bind an IPv4 address at all.

* * *

### Troubleshooting log

**Issue 1 - Wrong RouterOS image:**

- Symptom: /system license renew returns "bad command name renew", system unable to renew license and is stuck in a 24h limited free trial.
- Cause: Installed using an x86 RouterOS ISO build, not CHR, which uses a different licensing model, with no renew.
- Fix: Rebuilt node from CHR raw disk image.

**Issue 2 - pfSense WAN had no IPv4 bound:**

- Symptom: ifconfig em0 showed no address at all despite GUI showing 172.16.1.3/29 configured, CHR reported host unreachable.
- Cause: Default "Block private networks" and "Block bogon networks" on WAN, while this lab's WAN addresses legitimately sits in RFC1918 space.
- Fix: Unchecked both restrictions on HQ and Branch WAN, addresses were only successfully assigned after a full reboot.

**Issue 3 - Internet worked on CHR, not on pfSense:**

- Symptom: /ping 8.8.8.8 works on RouterOS, but no nodes connected to it could reach the internet.
- Cause: Site-specific srcnat rules matched on src-address alone, catching general internet-bound traffic from HQ's WAN IP and rewriting it to 198.51.100.10 before the masquerade rule ever saw it. Running /ip firewall connection print detail on CHR returned reply-dst-address=198.51.100.10(20) on connections that should've exited ether3.
- Fix: Added out-interface=ether2/ether1 to the two site srcnat rules.

[<- Previous Section](./Section1.md) | [Next Section ->](./Section3.md)
&nbsp;