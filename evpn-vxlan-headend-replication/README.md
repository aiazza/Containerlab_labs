## Topology

<img width="538" height="559" alt="image" src="https://github.com/user-attachments/assets/5ca408cb-77f0-4621-83c0-3a392a59a8d0" />

Configs:
https://github.com/aiazza/Containerlab_labs/tree/main/evpn-vxlan-headend-replication/configs

---

## Underlay architecture (BGP in the Datacenter)

This lab follows the BGP-in-the-datacenter architecture described in RFC 7938. The fabric is built using a classic leaf–spine topology with eBGP as the sole underlay routing protocol.

The underlay is an IPv6-only fabric using unnumbered links, where BGP sessions are formed directly on interfaces using IPv6 link-local addresses. This design removes the need for an IGP and avoids assigning numbered point-to-point addresses, which aligns with common large-scale datacenter practices.

While RFC 7938 defines the overall architecture, the ability to transport IPv4 reachability across an IPv6-only underlay relies on the IPv4-over-IPv6 next-hop mechanisms defined in RFC 8950.

---

## Underlay configuration

### Interface and IPv6 enablement

All leaf uplinks and spine downlinks are configured as Layer 3 interfaces with IPv6 enabled.

### Leaf configuration (Leaf 1–4)

```text
interface Ethernet1
   no switchport
   ipv6 enable
!
interface Ethernet2
   no switchport
   ipv6 enable
!
ip routing ipv6 interfaces
!
ipv6 unicast-routing
```

### Spine configuration (Spine 1–2)

```text
interface Ethernet1
   no switchport
   ipv6 enable
!
interface Ethernet2
   no switchport
   ipv6 enable
!
interface Ethernet3
   no switchport
   ipv6 enable
!
interface Ethernet4
   no switchport
   ipv6 enable
!
ip routing ipv6 interfaces
!
ipv6 unicast-routing
```

Enabling `ip routing ipv6 interfaces` is required to allow IPv6 forwarding across routed interfaces.

---

## BGP unnumbered configuration

```text
router bgp 65000
   router-id 10.0.0.1
   no bgp default ipv4-unicast
   neighbor ebgp peer group
   neighbor ebgp allowas-in 3
   neighbor interface Et1 peer-group ebgp remote-as 65100
   neighbor interface Et2 peer-group ebgp remote-as 65101
   !
   address-family ipv4
      neighbor ebgp activate
      neighbor ebgp next-hop address-family ipv6 originate
```

The next-hop originate statement is required to advertise IPv4 routes with an IPv6 next-hop over the IPv6-only fabric.

---

## VXLAN configuration

### Loopback and VTEP configuration

```text
interface Loopback0
   ip address 10.1.1.1/32
!
router bgp 65000
   address-family ipv4
      network 10.1.1.1/32
```

### VXLAN interface and VNI mapping

```text
vlan 100,200
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 100 vni 100
   vxlan vlan 200 vni 200
```

### Head-end replication (static flood list)

```text
interface Vxlan1
   vxlan vlan 100 flood vtep 10.1.1.1 10.1.1.2 10.1.1.3 10.1.1.4
```

---

## Traffic validation

### Host connectivity test

```text
/ # ping 192.168.10.12
64 bytes from 192.168.10.12: seq=0 ttl=64 time=25.482 ms
64 bytes from 192.168.10.12: seq=1 ttl=64 time=5.390 ms
```

---

## Key takeaways

- RFC 7938 defines the datacenter routing architecture
- RFC 8950 enables IPv4 routing over an IPv6-only underlay
- eBGP unnumbered simplifies underlay design
- Static head-end replication provides L2 flooding without EVPN
