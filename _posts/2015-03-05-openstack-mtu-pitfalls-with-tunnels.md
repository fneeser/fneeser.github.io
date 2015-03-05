---
layout: post
title:  "OpenStack: MTU pitfalls with tunnels"
categories: OpenStack Neutron Overlays
tags: OpenStack Neutron Overlay VXLAN GRE VLAN MTU
comments: true
analytics: true
---
Overlay tenant networks based on VXLAN and GRE tunnels have become extremely popular in OpenStack due to their flexibility and scalability.

Given the conceptual simplicity of tunneling a packet through a number of underlay networks, it may come as a surprise that getting the MTUs (Maximum Transmission Units) right for your OpenStack overlays and underlays is a tricky business with lots of pitfalls.  In this post, I'll provide an overview of the MTU issues related to tunnel encapsulation and current work towards resolving them in OpenStack.

The MTU of an IP network is the maximum size of an IP datagram (including IP header) that is supported by the network, or supported without IP fragmentation. For this reason, the MTU is often referred to as the L3 MTU.  On the other hand, the Ethernet L2 MTU refers to the maximum payload size of an Ethernet frame, which is usually 1500 bytes, but may be up to 9KB when Jumbo frames are supported.  As the size of an IP datagram matches the payload size of the enclosing Ethernet frame, the L3 MTU and the L2 MTU are the same and usually just referred to as "the MTU".

Overlay networks are implemented on top of one or more underlay networks (which may or may not be physical networks) with associated network MTUs.  The MTU effectively available to an overlay network corresponds to the minimum of the underlay MTUs along the path from source to destination, minus the per-packet overhead added by the tunnel encapsulation.  For simplicity, we'll exclude scenarios with nested tunnels.

When a datagram exceeds the MTU of a network that it needs to traverse, the datagram is either dropped or subjected to IP fragmentation.  As both actions have undesirable effects, this implies that in practice, sender and receiver in a given overlay network (a.k.a. "segment" in OpenStack speak) must agree on an MTU size, and the overlay must pass packets of the agreed size, regardless of the underlay networks being traversed and the presence (or not) of an encapsulation between sender and receiver.


## Connectivity problems due to MTU issues with tunnels
Numerous reports of ssh and ping connectivity problems for OpenStack VXLAN or GRE setups can be identified as MTU issues.  I'll mention two exemplary reports, which explicitly attribute the connectivity problems to MTU issues:

The [first report][linux-bridge-mtu-bug-with-vxlan] points out that packets arriving from a Neutron router have an MTU of 1500 bytes but are dropped if they need to be forwarded to another host via a VXLAN tunnel. Refers to an older release using Linux Bridge, but the problem is still present today.

The [second report][path-mtu-discovery-and-gre] nicely describes a scenario where an external site sent 1500B packets
(20 (IPv4-hdr) + 32 (TCP-hdr) + 1448 (TCP-segment) = 1500 bytes) to a VM inside a home datacenter. The 1500B packets passed through a home router (with NAT), traversed a Neutron router on the Network Node (NN) and finally needed to go through a GRE tunnel to reach the VM.
However, the packets got dropped inside the NN because they exceeded the effective GRE tunnel MTU (1454B) for the given underlay MTU (1500B).  In this case, path MTU discovery (PMTUD) was enabled and ICMPs (destination not reachable) were sent to the source.  However, PMTUD failed because the ICMPs were either not "NAT-adjusted" by the poster's home router or discarded by a firewall before reaching the external source.


## MTU problems in OpenStack
An excellent description of the MTU problems in OpenStack and a spec for resolving them can be found in [MTU selection and advertisement][mtu-selection-and-advertisement] -- let me expand on a few key points cited from the "Problem Description":

> "MTU is frequently inconsistent between different endpoints in a segment. Between two VMs on the same host, it is quite common to be able to communicate with transfer units that are bigger than between two VMs on different hosts. The MTU for the segment should be the minimum that works between any pair of hosts, but this is not always true."

A couple of comments on this point:

* In an OpenStack overlay network, VMs on the same host communicate via a local VLAN, whereas VMs on different hosts communicate through a tunnel. Nevertheless, the MTU of the overlay should be selected such that it can be supported between any pair of VMs.  Complications may also arise due to the desire to accelerate intra-host VM-to-VM communication by using a larger MTU in combination with segmentation offloading.
* With Neutron routers, the situation is even more complex because traffic may be routed between tenant (overlay) networks and one or more "external" networks.  An "external" network may in fact be visible to the host IP stack, so we also have scenarios where traffic is routed between the host (hypervisor) and a local guest, or between the host and a remote guest.  These observations strongly suggest that tenant networks should have the same MTU as the external network(s) they communicate with.  If an external network is supposed to have a standard MTU of 1500 bytes and a tenant network uses VXLAN, then the underlay carrying the encapsulated traffic must be enabled for Jumbo frames.
* In overlay setups with compute nodes having a single NIC (not currently recommended by OpenStack documentation but nevertheless useful), further difficulties arise because the same external network needs to accommodate both native and encapsulated traffic. One promising idea is to eliminate MTU inconsistencies by splitting the external network into VLANs with different MTUs for native and encapsulated traffic -- I will expand on this idea in a follow-up post.


> "Note that L3 mechanisms for path MTU discovery only take place for packets passing through a L3 element such as a router. They do not take place for traffic transiting an L2 segment ..."

This raises some interesting questions for OpenStack:

* What if a tunnel endpoint (TEP) outputs a packet with the Don't Fragment (DF) bit set in the IP encapsulation header and the packet gets dropped inside the tunnel because it exceeds the MTU of an underlay network?  -- The MTU violation will be detected by a router (in the kernel or an intermediate router), so the router can be expected to send an ICMP Unreachable (Fragmentation Required) error (type=3, code=4) back to the tunnel endpoint. But what is the TEP going to do with the received ICMP?  A (stateless) tunnel endpoint might not easily relay the ICMP back to the ultimate source.
* As a subcase of the above, a (V)TEP and the router detecting the MTU violation may actually be co-located. This might allow a joint (V)TEP/router implementation to send an ICMP Unreachable error directly to the ultimate source. For hardware, such an optimization is suggested by ["Resolve IP fragmentation, MTU, MSS, and PMTUD Issues with GRE and IPSEC", section 'The Router as a PMTUD Participant at the Endpoint of a Tunnel'][cisco-pmtud-ipfrag]. How is this implemented in the Linux kernel?
* Does PMTU discovery work at all with tunnels if the OpenStack admin has no control over the MTUs of intermediate underlay networks?

See also "Related documentation" below!


## MTU selection and advertisement
There is [work in progress][mtu-selection-and-advertisement-patches] towards implementing the changes proposed in [MTU selection and advertisement][mtu-selection-and-advertisement], currently targeting OpenStack Kilo.  See also the more recent [MTU selection spec update][mtu-selection-spec-update].

The plan is that Neutron will internally calculate two MTU values per tenant network, namely
(a) the maximum MTU that will work, and (b) the optimum MTU to use.  Also, Neutron will take into account a couple of new, global configuration variables:

* `path_mtu`: (For use by the L3 service driver to determine) The maximum permissible size of an unfragmented packet travelling from and to addresses where encapsulated Neutron traffic is sent. Drivers calculate maximum viable MTU for validating tenant requests based on this value
(typically, `path_mtu - max_encap` header size). If `<=0`, the path MTU is indeterminate and no calculation takes place.
* `segment_mtu`: (For use by L2 plugins & mechanism drivers to determine) The maximum permissible size of an unfragmented packet travelling the network segment. If `<=0`, the path MTU is indeterminate and no calculation takes place.
* `physnet_mtus`: Physnet MTUs. List of mappings of physnet to MTU value. The format of the mapping is `<physnet>:<mtu val>`. This mapping allows specifying a physnet MTU value that differs from the default `segment_mtu` value.


The following proposed changes caught my eye:

> * the plugin can ensure that network services attached to a network operate with the correct MTU (e.g. router ports have an appropriate interface MTU configured) so that, even in the case of differing MTU sizes on different networks, normal L3 behaviour will accommodate the variance without breaking tenant communications.
<br>
...

> Regardless of the advertisement setting, Neutron network appliances such as routers, and advanced service appliances, will have their interfaces configured to the selected MTU if the MTU is known. This implies that the plugging mechanism for the service understands and correctly implements MTU setting.
<br>
...

> An Openstack network typically has physical networks for (a) external connectivity and (b) provider network use. These networks may or may not have the same MTU size as each other and are quite likely to have an MTU size different from that of the virtual tenant networks. Link MTU for physical networks is specified by configuration and will be used as the maximum and preferred MTU sizes for provider and external networks on those segments.

Wait a minute -- for *single-NIC setups*, what if an external network (connected to br-ex, as specified through `bridge_mappings`) needs to carry VLANs having different MTUs? -- In this case, it doesn't make sense to enforce a `physnet_mtu` on Neutron `qg-XXX` gateways.

How about enabling such setups as follows:

First, define a small number of per-VLAN provider networks on top of the given physical network and specify different MTUs for these provider networks, provided they are less than or equal to the `physnet_mtu`. (Let's keep in mind that an OVS bridge enforces a consistent MTU across its ports on a per-VLAN basis.)  This allows splitting the external network into VLANs with different MTUs for native and encapsulated traffic.

Next, create a Neutron router that connects to a specific provider network (VLAN) for native (externally routed) traffic.

I believe this can be achieved within the given "MTU selection" spec -- thoughts?



## Related documentation
OpenStack documentation on MTU configuration is [work in progress][ml2-config-options].

Here's a knowledgable article on resolving MTU issues with tunnels:

* [Resolve IP fragmentation, MTU, MSS, and PMTUD Issues with GRE and IPSEC][cisco-pmtud-ipfrag]


IETF describes several aspects of Path MTU (PMTU) discovery with tunnels:

> [RFC 2784 (GRE)][rfc2784-gre]: "Existing implementations of GRE, when using IPv4 as the Delivery Header, do not implement Path MTU discovery and do not set the Don't Fragment bit in the Delivery Header.  This can cause large packets to become fragmented within the tunnel and reassembled at the tunnel exit (independent of whether the payload packet is using PMTU).  If a tunnel entry point were to use Path MTU discovery, however, that tunnel entry point would also need to relay ICMP unreachable error messages (in particular the "fragmentation needed and DF set" code) back to the originator of the packet, which is not a requirement in this specification. Failure to properly relay Path MTU information to an originator can result in the following behavior: the originator sets the don't fragment bit, the packet gets dropped within the tunnel, but since the originator doesn't receive proper feedback, it retransmits with the same PMTU, causing subsequently transmitted packets to be dropped."

> [RFC 7348 (VXLAN)][rfc7348-vxlan]: "VTEPs MUST NOT fragment VXLAN packets. Intermediate routers may fragment encapsulated VXLAN packets due to the larger frame size. The destination VTEP MAY silently discard such VXLAN fragments. To ensure end-to-end traffic delivery without fragmentation, it is RECOMMENDED that the MTUs (Maximum Transmission Units) across the physical network infrastructure be set to a value that accommodates the larger frame size due to the encapsulation. Other techniques like Path MTU discovery (...) MAY be used to address this requirement as well."

> [draft-gross-geneve-02][draft-gross-geneve-02]: "It is strongly RECOMMENDED that Path MTU Discovery (...) be used by setting the DF bit in the IP header when Geneve packets are transmitted over IPv4 (this is the default with IPv6)."



## Final remarks
Hopefully this post will help resolve the MTU issues.

It's important to make the MTU handling in OpenStack more user friendly, while also enabling overlay setups with single-NIC nodes.  For the ongoing MTU fixes, we should take into account the possibility of splitting a (single) external network into VLANs for separating MTU-incompatible traffic types.

I do appreciate your comments on any related technical aspects.

Please stay tuned for my next post on "OpenStack VXLAN with single-NIC compute nodes"!



[linux-bridge-mtu-bug-with-vxlan]: https://bugs.launchpad.net/openstack-manuals/+bug/1242534
[path-mtu-discovery-and-gre]: http://techbackground.blogspot.ch/2013/06/path-mtu-discovery-and-gre.html

[mtu-selection-and-advertisement]: https://github.com/openstack/neutron-specs/blob/master/specs/kilo/mtu-selection-and-advertisement.rst

[mtu-selection-spec-update]: https://review.openstack.org/#/c/159146/

[mtu-selection-and-advertisement-patches]: https://review.openstack.org/#/c/153733/


[ml2-config-options]: http://docs.openstack.org/trunk/config-reference/content/networking-options-plugins-ml2.html

[cisco-pmtud-ipfrag]: http://www.cisco.com/c/en/us/support/docs/ip/generic-routing-encapsulation-gre/25885-pmtud-ipfrag.html

[rfc2784-gre]: https://tools.ietf.org/html/rfc2784
[rfc7348-vxlan]: https://tools.ietf.org/html/rfc7348
[draft-gross-geneve-02]: http://tools.ietf.org/html/draft-gross-geneve-02
