### **Topology**

<img width="538" height="559" alt="image" src="https://github.com/user-attachments/assets/5ca408cb-77f0-4621-83c0-3a392a59a8d0" />


[Configs](https://github.com/aiazza/Containerlab_labs/tree/main/evpn-vxlan-headend-replication/configs) 

## Underlay configuration

First of all we need to configure our underlay, to do that we need to configure point-to-point IP addresses and then configure an underlay routing protocol, to mimic what I have usually seen in large scale Datacenters I am using EBGP as the underlay following the same concepts (not exactly same ASN addressing) as in [RFC7938](https://datatracker.ietf.org/doc/html/rfc7938) . To make it easy I will setup what we call BGP unnumbered following [RFC8950](https://www.rfc-editor.org/rfc/rfc8950) which allows us to form BGP neighborships on top of IPv6 link-local addresses so that saves me the hassle of creating p2p interface addresses. 

### BGP unnumbered setup

**Leaf 1 - 4** 

```jsx
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

**Spine 1-2** 

```jsx
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

This enables IPv6 on leaf uplinks and spine downlinks and enables ip routing, I had an issue during the lab where i couldn’t ping p2p interfaces even though everything was setup correctly, I later found that enabling `ip routing ipv6 interfaces` made everything work so don’t forget that. Now for the BGP configuration 

**Leaf 1 - 4** 

```jsx
router bgp 65000
   router-id 10.0.0.1 <===== unique per router
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

So what I am doing here is creating a peer group called ‘ebgp’ and activating BGP on the interfaces we configured with IPv6 earlier, by activating this BGP will form a session over Eth1 and Eth2 without needing an IP address, it will use the pre-configured ipv6 LLA. 
After that in the ipv4 address family we activate the peer group that allows for route exchange and also tell EOS to allow IPv4 prefixes to be originated with an IPv6 next hop. Without this, the IPv4 routes would expect to be able to resolve an IPv4 next hop and not find it which would prevent the route from being advertised.

Also one thing thats important is that because i am using a unique ASN for the leafs, I am setting enabling `neighbor ebgp allowas-in 3`, if we don’t then any traffic going from leaf to leaf will be blocked due to eBGP loop prevention mechanism where if it see’s its own ASN in the AS Path it will drop the route. 

After configuring this you should see BGP sessions established between leafs and spines :

- Leaf1

```jsx
leaf1#sh ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Neighbor                      V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc PfxAdv
  fe80::a8c1:abff:fe4b:7835%Et2 4 65101              8         7    0    0 00:01:14 Estab   1      1      2
  fe80::a8c1:abff:fef7:33e3%Et1 4 65100            667       665    0    0 00:12:22 Estab   1      1      1
```

- Spine1

```jsx
spine1#sh ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.1.1, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Neighbor                      V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc PfxAdv
  fe80::a8c1:abff:fe19:a943%Et2 4 65000            638       664    0    0 08:44:21 Estab   0      0      2
  fe80::a8c1:abff:fe1d:2534%Et3 4 65000            470       472    0    0 06:37:28 Estab   0      0      2
  fe80::a8c1:abff:fe43:7966%Et4 4 65000            482       496    0    0 06:19:27 Estab   1      1      1
  fe80::a8c1:abff:fe97:eb98%Et1 4 65000            649       667    0    0 00:12:34 Estab   1      1      1
```

Great, now that we have our underlay setup we can start configuring the VXLAN stuff. 

## VXLAN setup

First thing we need to do is configure our loopbacks and advertise them in BGP 

- leaf1

```jsx
interface Loopback0
   ip address 10.1.1.1/32
router bgp 65000
	 address-family ipv4
	    network 10.1.1.1/32
```

- leaf4

```jsx
interface Loopback0
   ip address 10.1.1.4/32
router bgp 65000
	 address-family ipv4
	    network 10.1.1.4/32
```

Each leaf should now be receiving each other leafs loopback address  : 

- leaf1

```jsx
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65000
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Pending FIB install
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.1.1/32            -                     -       -          -       0       i
 * >      10.1.1.4/32            fe80::a8c1:abff:fef7:33e3%Et1 0       -          100     0       65100 65000 i
 *        10.1.1.4/32            fe80::a8c1:abff:fe4b:7835%Et2 0       -          100     0       65101 65000 i
```

- leaf4

```jsx
BGP routing table information for VRF default
Router identifier 10.0.0.4, local AS number 65000
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Pending FIB install
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.1.1/32            fe80::a8c1:abff:fe2a:5fe8%Et1 0       -          100     0       65100 65000 i
 *        10.1.1.1/32            fe80::a8c1:abff:feff:df82%Et2 0       -          100     0       65101 65000 i
 * >      10.1.1.4/32            -                     -       -          -       0       i
```

So to setup VXLAN we first need to start by configuring a VLAN and then setting up a VXLAN interface with a VLAN <> VNI mapping 

- leaf1

```jsx
vlan 100,200
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 100 vni 100
   vxlan vlan 200 vni 200
```

- leaf4

```jsx
vlan 100,200
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 100 vni 100
   vxlan vlan 200 vni 200
```

What we did there is basically tie VLAN 100,200 to VXLAN VNI 100,200. Now what happens if host1 sends an ARP to host2 ? It still won’t work as ARP traffic won’t be carried over a Layer3 network, now we need to add a replication list to tell VXLAN that any BUM traffic it receives on that interface, it should replicate it to the following VTEP IP’s, so think of it as statically defining a list of remote hosts you would “flood to”

- leaf1

```jsx
interface Vxlan1
   vxlan vlan 100 flood vtep 10.1.1.1 10.1.1.2 10.1.1.3 10.1.1.4
```

- leaf4

```jsx
interface Vxlan1
   vxlan vlan 100 flood vtep 10.1.1.1 10.1.1.2 10.1.1.3 10.1.1.4
```

that should be all thats required for VXLAN to carry L2 traffic over L3, let’s give it a try :

- host1

```jsx
/ # ping 192.168.10.12
PING 192.168.10.12 (192.168.10.12): 56 data bytes
64 bytes from 192.168.10.12: seq=0 ttl=64 time=25.482 ms
64 bytes from 192.168.10.12: seq=1 ttl=64 time=5.390 ms
64 bytes from 192.168.10.12: seq=2 ttl=64 time=12.027 ms
64 bytes from 192.168.10.12: seq=3 ttl=64 time=9.344 ms
^C
```

let’s check this with a packet capture on leaf1’s uplink 

<img width="1510" height="1656" alt="image" src="https://github.com/user-attachments/assets/c448561a-92a0-4114-85f4-1c364fc8e7a7" />

you can see that the packet is being encapsulated in VNI 100 , you can also see the ARP request from host1 to host2 a little lower.
