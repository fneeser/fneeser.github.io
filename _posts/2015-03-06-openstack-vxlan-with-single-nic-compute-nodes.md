---
layout: post
title:  "OpenStack VXLAN with single-NIC compute nodes"
categories: OpenStack Neutron Overlays
tags: OpenStack Neutron OVS Overlay VXLAN GRE VLAN MTU
comments: true
analytics: true
---

In my [previous post on "OpenStack: MTU pitfalls with tunnels"]({% post_url 2015-03-05-openstack-mtu-pitfalls-with-tunnels %}), I mentioned the current difficulties in setting up an OpenStack overlay network with single-NIC compute nodes, which are caused by "mixing" native and encapsulated traffic on the same LAN.  Also, I briefly sketched an idea to eliminate any MTU inconsistencies by splitting the external network into VLANs with different MTUs for native and encapsulated traffic.

In this post, I'm going to describe this VLAN-based traffic separation for overlays in more detail.

My test setup at home is a VXLAN overlay network with two compute nodes on top of a single physical underlay network. I'm using the OpenStack *Juno* release on Fedora 20.

Besides enabling VXLAN East-West traffic among guests sitting on the two compute nodes, I wanted to take advantage of the Neutron L3 agent routing/NAT functionality to give my guests access to the "external" network (LAN and Internet) for North-South traffic.

## OpenStack single-NIC setups

Due to a functional split between Nova and Neutron, OpenStack uses separate [Open vSwitch (OVS)][openvswitch] instances, for connecting the guests and for controlling tunnel endpoints (TEPs), namely an integration bridge br-int and a tunnel bridge br-tun. OpenStack also assumes an additional bridge for each "external" network.

See ["Networking in too much detail"][networking-in-too-much-detail] for a typical OpenStack overlay setup; there are only minor differences between a GRE setup and a VXLAN setup.
The [OpenStack Cloud Admin Guide][openstack-cloud-admin-guide] provides general information on the use of OVS bridges in OpenStack and shows various networking scenarios; have a look at the Network Host configurations, which assume at least two NICs and separate LANs.
In addition, a third NIC is often recommended for management.

My nodes have just a single NIC, so it seems natural to start with a setup as shown in Fig. 1.


<!--
image.html

Features:
- Figure is autosized depending on the size of the browser window!
- width tag allows resizing a figure while keeping it horizontally centered
- url tag makes figure hyperlinked, with a title given by the title tag
- Provides figure caption support
- No grey background

  Tag     Example                             Description
  ------------------------------------------------------------------------------
  img     img="assets/my-pic.png"             Path to image source
  
  width   width="70%"                         Image width in percent (<= 100%),
                                              relative to the usable width of
                                              the browser window
                                              
  title   title="Beautiful sunset"            Title shown when the mouse pointer
                                              is placed on top of the image.
                                              If the url tag is also present,
                                              then the title is shown next to
                                              the image hyperlink.
                                              
  caption caption="Fig. 1: Beautiful sunset"  Caption is shown centered
                                              underneath the image
  
  url    url="/assets/my-pic.png"             Optional tag.  If present, the
                                              image acts as a hyperlink when
                                              the mouse pointer is placed on
                                              top of it.                                  

FIXME:
- url tag requires a leading slash (like an absolute path), whereas img tag
  requires NO leading slash (like a relative path)
  ==> Should make this more uniform
  
- URL redirection

-->

{% include image.html img="assets/openstack-setup-single-nic-nodes-and-L2-switch.png" width="70%" title="VXLAN setup with single-NIC nodes" caption="Fig. 1: VXLAN setup with single-NIC nodes: Initial attempt" url="/assets/openstack-setup-single-nic-nodes-and-L2-switch.png" %}


For compatibility with the new Distributed Virtual Routing (DVR) capability in OpenStack Juno, each Compute Node is assumed to have a Neutron Router. However, for the present study, it's sufficient to consider centralized routing, where only one host has an active Neutron Router.

OpenStack users have tried several alternative *single-NIC setups*; see for example this [Multi-Node Single-Nic OpenStack setup][multi-node-single-nic-fredski]. However, many users reported difficulties with MTU issues and packet drops.

So what's the problem with overlays "using a single LAN"? -- The problem is that mixing native and encapsulated traffic on the same LAN may result in silent packet drops within the host and an inability to ssh into guests, as the encapsulation will cause packets to exceed the MTU when the source generates MTU-sized packets.  For example, a VXLAN encapsulation with an IPv4 outer header and no inner VLAN tag results in an overhead of `20 (IPv4) + 8 (UDP) + 8 (VXLAN) + 14 (Enet w/o VLAN tag) = 50` bytes, so connecting both sides of a tunnel endpoint to the same LAN is a conceptual mistake.  Furthermore, the problem cannot be solved by configuring bridge ports differently, because the ports of a bridged network are required to have the same MTU.

Sounds easy to fix in OpenStack? -- For example, how about setting the MTUs to 1450B for your guest virtual interfaces and to 1500B for br-ex, the interface associated with the internal OVS port of bridge br-ex? -- Unfortunately, that's not going to work in all situations:

> When the host (hypervisor) itself injects an MTU-sized IP datagram (IP datagram size = MTU = 1500B) into br-ex but the packet targets a *floating IP* representing a guest on a remote compute node, then a Neutron router will *cause the packet to take an extra turn* through br-int, the patch "cable" connecting patch-tun with patch-int, br-tun and the VXLAN Tunnel End Point (TEP), *and finally to reenter br-ex*. Due to the VXLAN encapsulation, the original 1500B datagram will end up as a 1550B datagram and get dropped by the br-ex interface.

One way to address this problem is to use two physical interfaces (NICs) and LANs for native and encapsulated traffic, or even three NICs and LANs for separating externally routed, management, and encapsulated traffic.
This approach also helps to avoid losing connectivity during remote administration.  OTOH, physically separating traffic types via dedicated NICs and links does not provide high availability (HA) -- the failure of one NIC or link still breaks the system. If HA is important, then carrying all traffic types across a bond comprised of two links is a better approach.

Next we consider a more affordable, VLAN-based solution that works with single-NIC nodes as well as with HA setups using bonded links.

### Using VLANs to separate incompatible traffic types

OVS and the physical L2 switch in the setup of Fig. 1 both support 802.1Q VLANs, so let's try to separate the conflicting traffic types using a small number of (underlay) VLANs on a single LAN ("external" network), as illustrated in Fig. 2.

{% include image.html img="assets/openstack-setup-single-nic-nodes-and-L2-switch-with-VLANs.png" width="70%" title="VXLAN setup with VLANs for traffic separation" caption="Fig. 2: VXLAN setup with single-NIC nodes using VLANs for traffic separation" url="/assets/openstack-setup-single-nic-nodes-and-L2-switch-with-VLANs.png" %}

Using VLANs allows segregating encapsulated and native traffic and setting appropriate MTUs on a per-VLAN (traffic type) basis.  For example, we may need to adhere to the standard MTU of 1500B for externally routed traffic but want to allow Jumbo frames for internal communication, to accommodate the VXLAN encapsulation overhead or optimize the MTU for networked storage traffic.

Two (birectional) flows involving a Neutron Router are shown in Fig. 2: The orange flow is between the host (hypervisor) and the colocated VM x.1, whereas the yellow flow is between the host and a non-colocated VM. In both cases, the host reaches the VM via an OpenStack floating IP. Packets of the yellow flow pass through br-ex twice (without and with encapsulation), but *on different VLANs*.

Setting different MTUs per VLAN is enabled on bridge br-ex by first creating a separate *VLAN access port* `br-ex.vlan` for each vlan in `{1,12,22,66}`.  Each of these ports has an identically named interface that needs to be assigned a suitable IP address, which is further explained below. Next, we need to make sure that L3 sources inject their traffic into the appropriate `br-ex.vlan` interface, as indicated by the solid, dotted and dashed lines within the router of the host network stack. For example, VXLAN traffic must be source-routed to `br-ex.12`.  On many Linux distros including Fedora and Ubuntu, such traffic steering can be accomplished with *Linux policy routing*, as explained in [Scott Lowe's use case for handling tunnel traffic.][slowe-use-case-for-policy-routing]

For the small system shown in Fig. 2, I will assume that the IP addresses assigned to the `br-ex.vlan` interfaces of compute node `c` are identical, namely, `ip(c,vlan) = 192.168.1.10`. It is still possible to select a specific `br-ex.vlan` interface, say, for VXLAN traffic destined to UDP port 4789.

Fig. 2 also shows Neutron router gateway interfaces `qg-*` with floating IPs (denoted symbolically as `ip(f*,1)`) associated with VMs. Here we assume that the floating IPs are in the same subnet as `ip(c,1)`.

The following table shows the desired VLAN configuration:

<table>
  <thead>
    <tr>
      <th>Traffic type</th>
      <th>VLAN ID</th>
      <th>IP subnet</th>
      <th>MTU <br> [bytes]</th>      
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Routable North/South</td>
      <td>1</td>
      <td>192.168.1.0/24</td>
      <td>1500</td>      
    </tr>
    <tr>
      <td>VXLAN East/West</td>
      <td>12</td>
      <td>192.168.1.0/24</td>
      <td>1550</td>      
    </tr>
    <tr>
      <td>Storage East/West</td>
      <td>22</td>
      <td>192.168.1.0/24</td>
      <td>9000</td>       
    </tr>
    <tr>
      <td>Management</td>
      <td>66</td>
      <td>192.168.1.0/24</td>
      <td>1500</td>       
    </tr>        
  </tbody>
</table>


Note that `br-ex` needs to be configured by the administrator.  Besides configuring the `br-ex.vlan` access ports (including assigning IPs to them), the admin has to set up eth0 as an OVSPort, in such a way that the link will carry the VLANs shown by the colored VLAN membership indicators. Note also the port VLAN membership of the physical L2 switch in Fig. 2.

The MTU of eth0 (i.e., of the physical network) must be configured to the max of the MTUs in the table above.
Moreover, the per-VLAN MTUs must be set with

	# ip link set dev br-ex.<vlan> mtu <mtu-value>

Due to OVS implementation aspects, the latter MTUs should be set only after the MTU of eth0 has been set; achieving this order can be tricky in a systemd-based boot environment.

For a more scalable setup with independent L2 broadcast domains and L3 routing in between, one could set `ip(c,vlan) = 192.168.vlan.10` to associate the VLANs with different subnets.  I will explore this in a future post.

If you'd like to try out the VLAN-based traffic separation, [here are some VLAN configuration tips]({% post_url 2015-03-06-openstack-vxlan-vlan-config-tips-for-single-nic-setups %}) that you may find useful.

> After setting up the VLANs as shown above, are we done yet? -- Not quite. There's a remaining VLAN configuration issue with the Neutron external gateway ports `qg-*` -- see the section "Neutron router gateway configuration" below.


## Routing/NAT via a VLAN-enabled external network


### Neutron router and br-ex

A Neutron router uses a *router port* named `qr-<id>` on bridge br-int and a *gateway port* `qg-<id>`  on the bridge that connects to the desired "external" network.

In my case, the gateway ports `qg-<id>` are on br-ex, which connects to a VLAN-capable LAN via OVSPort eth0.  If I configure eth0 for native VLAN 1, then any traffic to/from a Neutron router must be tagged "1" as well inside br-ex. On the other hand, the Neutron routing/NAT function is on Layer 3 and by definition unaware of the VLAN (L2) properties of the external network.

Unfortunately, I found that the Neutron L3 agent in Juno is unaware of the VLAN properties of the "external" network associated with the router gateway port, even if the Neutron server is made aware of them.

So let's have a closer look at Neutron router gateway ports.


### Neutron router gateway configuration

After I had set up the above VLAN configuration, native VLAN communication between the host stack and external hosts worked just fine through ports br-ex.1 and eth0.  However, Neutron routing/NAT was not working yet.  I found that the Neutron L3 agent -- unaware of the VLAN properties of the "external" network (a.k.a. provider network) -- does not set a VLAN ID when it creates a `qg-<id>` gateway port on br-ex.  As a result, port `qg-<id>` operates in the default trunking mode.

Why is this a problem? -- Here's why: Frames injected into `qg-<id>` by the Neutron router are **untagged** and hence not handled by eth0, whereas untagged frames from the NIC are **tagged** "1" as they are injected via eth0 but discarded when they reach `qg-<id>`, which has `trunks=[]` (emtpy list) and `tag=[]` (optional integer missing).

To control the VLAN for routable North/South traffic, I had created a Neutron router between a VXLAN overlay/tenant network and a *VLAN provider network* with `segment_id = 1`, i.e., VLAN 1. One would expect Neutron to configure `qg-<id>` as a *VLAN access port* by setting `tag = 1`. But when I looked at the output of `ovs-vsctl show` for port `qg-<id>`, the tag was missing.

If `qg-<id>` was a VLAN access port for VLAN 1, then traffic to/from `qg-<id>` would be tagged "1" inside br-ex as desired (cleanly separated from other traffic types), whereas the Neutron routing/NAT would be able to handle native traffic to/from `qg-<id>` as it does today.  So I entered

	# ovs-vsctl set port qg-<id> tag=1

and *bingo*, native traffic to/from the Neutron router started flowing!

So, while getting rid of [MTU pitfalls with tunnels]({% post_url 2015-03-05-openstack-mtu-pitfalls-with-tunnels %}), it seems like this could become a clean solution for overlay setups with single-NIC compute nodes -- if we just put together a few missing elements.

Interested? -- I'm curious about your thoughts - feel free to add a comment!


## Coming up next
Of course, administratively "fixing" the configuration of Neutron gateway ports would not be a persistent solution as Neutron manages (creates/deletes) these ports dynamically.

In another post, I'll show why the Neutron L3 agent is unaware of the external network VLAN properties and how this could be fixed with a small patch.

<!---
TODO: Should we use a special font for symbols such as 'eth0', 'br-ex' and inlined code?
-->


[multi-node-single-nic-fredski]: http://www.fredski.nl/multi-node-single-nic-openstack-setup/
[networking-in-too-much-detail]: https://openstack.redhat.com/Networking_in_too_much_detail
[openstack-cloud-admin-guide]: http://docs.openstack.org/admin-guide-cloud/content/under_the_hood_openvswitch.html#under_the_hood_openvswitch_scenario1_network
[openvswitch]: http://openvswitch.org
[slowe-use-case-for-policy-routing]: http://blog.scottlowe.org/2013/05/30/a-use-case-for-policy-routing-with-kvm-and-open-vswitch/
