---
layout: post
title:  "OpenStack VXLAN: VLAN configuration tips for single-NIC setups"
categories: OpenStack Neutron Overlays
tags: OpenStack Neutron OVS Overlay VXLAN GRE VLAN MTU
comments: true
analytics: true
---

This page provides some VLAN configuration tips for the setup described in my [post "OpenStack VXLAN with single-NIC compute nodes"]({% post_url 2015-03-06-openstack-vxlan-with-single-nic-compute-nodes %}).
 
A great reference for the OVS VLAN configuration is the "Port TABLE" section of the ovs-vswitchd.conf.db(5) man page, which shows the VLAN-related port attributes (vlan_mode, tag, trunks), followed by a detailed description of the 4 VLAN modes (trunk, access, native-tagged, native-untagged) and the *native VLAN* concept.

### br-ex VLAN configuration and native-untagged VLAN mode
On my home LAN, I use VLAN tagging primarily to distinguish traffic types that require a different MTU.

For connectivity with the host IP stack, bridge br-ex needs one or more internal ports to which an IP address is assigned.  Any OVS bridge has an internal port with the same name, configured in trunk mode by default.  Trunk mode is not so useful from a host-stack routing perspective, so instead of using the br-ex port, I prefer to create VLAN access ports named `br-ex.vlan` for my VLANs.

For compatibility with my ADSL router and nodes that are not 802.1Q-enabled, it's convenient to (i) use VLAN 1 as the native VLAN and (ii) make sure that packets egressing via eth0 in the native VLAN will be untagged.

With OVS I can have such a configuration by setting port eth0 to *native-untagged* VLAN mode. This mode resembles the trunk mode except that (i) a packet that ingresses on a native-untagged port without an 802.1Q header (VLAN tag) is in the native VLAN and (ii) a packet that egresses on a native-untagged port in the native VLAN will not have an 802.1Q header (VLAN tag). So, after issuing the commands

{% highlight bash %}
ovs-vsctl set port eth0 vlan_mode=native-untagged
ovs-vsctl set port eth0 trunks=[1, 12, 22, 66]
ovs-vsctl set port eth0 tag = 1
{% endhighlight bash %}

eth0 accepts traffic on the VLANs identified by trunks, handles untagged received frames by tagging them "1" inside the bridge and strips any tags with VLAN 1 for outbound frames.  B.t.w., you may list OVS port attributes using

{% highlight bash %}
ovs-vsctl list port <port-name>
{% endhighlight bash %}

Based on the above, the br-ex VLAN configuration looks as follows:

<table>
  <thead>
    <tr>
      <th>Port</th>
      <th>vlan_mode</th>
      <th>trunks</th>
      <th>tag</th>
      <th>Effective mode</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>qg-&lt;id&gt;</td>
      <td>[]</td>
      <td>[]</td>
      <td>1</td>
      <td>Access port</td>
    </tr>
    <tr>
      <td>br-ex.1</td>
      <td>[]</td>
      <td>[]</td>
      <td>1</td>
      <td>Access port</td>
    </tr>
    <tr>
      <td>br-ex.12</td>
      <td>[]</td>
      <td>[]</td>
      <td>10</td>
      <td>Access port</td>
    </tr>
    <tr>
      <td>br-ex.22</td>
      <td>[]</td>
      <td>[]</td>
      <td>11</td>
      <td>Access port</td>
    </tr>
    <tr>
      <td>br-ex.66</td>
      <td>[]</td>
      <td>[]</td>
      <td>66</td>
      <td>Access port</td>
    </tr>    
    <tr>
      <td>eth0</td>
      <td>native_untagged</td>
      <td>[1, 12, 22, 66]</td>
      <td>1</td>
      <td>Native untagged</td>
    </tr>
  </tbody>
</table>

B.t.w., the configuration of physical NIC interfaces on br-ex is not provided by OpenStack. I statically configured eth0 as an OVSPort.

Like OVS, physical switches supporting 802.1Q should allow for an equivalent "native untagged" VLAN membership configuration. For simplicity, I use a uniform native VLAN configuration, where eth0 on br-ex and *all* ports on the external switch are native-untagged, so native traffic (including externally routed traffic) is untagged on all physical links and in fact the only traffic class that is untagged.

Alternatively, I could use the native-tagged mode inside the LAN and the native-untagged mode only on the port facing the ADSL router. This would be equivalent in terms of isolating traffic classes, but might be beneficial if I wanted to consistently enable 802.1Q Priority Flow Control (PFC) on my underlay.
