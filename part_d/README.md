# PART D

This section explains how Address Resolution Protocol (ARP) flows are handled by OVS. It also clarifies for what reason MAC address "aa:bb:cc:dd:ee:ff" is used in various tables by OVS in the previous sections.

To revisit the related section in the OVS pipeline, as shown below, an ARP flow would end up being processed by Table 20.

![](2020-10-14_09-36-35.png)

# 12. ArpResponder Table #20

Table 20 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=20 --no-stats
 cookie=0x1020000000000, table=20, priority=200,arp,arp_tpa=10.222.0.1,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:aa:bb:cc:dd:ee:ff,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0xaabbccddeeff->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xade0001->NXM_OF_ARP_SPA[],IN_PORT
 cookie=0x1020000000000, table=20, priority=200,arp,arp_tpa=10.222.2.1,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:aa:bb:cc:dd:ee:ff,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0xaabbccddeeff->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xade0201->NXM_OF_ARP_SPA[],IN_PORT
 cookie=0x1000000000000, table=20, priority=190,arp actions=NORMAL
 cookie=0x1000000000000, table=20, priority=0 actions=drop
vmware@master:~$ 
</code></pre>

The first flow entry is to process the ARP request flows sent for the IP address 10.222.0.1 which is **master** node' s antrea-gw0 IP. 

The second flow entry is to process the ARP request flows sent for the IP address 10.222.2.1 which is the **worker2** node' s antrea-gw0 IP. 

**But why would OVS on worker1 node need to process ARP request flows for other Kubernetes nodes' gw0 interface IPs ?** Isnt each node' s gw0 interface in its unique subnet ? To remind again, the pod subnets assigned by node ipam controller to each individual Kubernetes node in this Kubernetes cluster are as following : 

- master - 10.222.0.0/24
- worker1 - 10.222.1.0/24
- worker2 - 10.222.2.0/24

The answer to the above question lies in the routing table of the worker 1 node. (which was shown back in [Part A Section 3.1.1](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_a#311-worker-1)) Shown below once again. The highlighted lines in the output are the route entries for the pod subnets of the **other** Kubernetes nodes; to be precise worker2 and master nodes. 

<pre><code>
vmware@<b>worker1</b>:~$ ip route
default via 10.79.1.1 dev ens160 proto static
10.79.1.0/24 dev ens160 proto kernel scope link src 10.79.1.201
<b>10.222.0.0/24 via 10.222.0.1 dev antrea-gw0 onlink </b>
10.222.1.0/24 dev antrea-gw0 proto kernel scope link src 10.222.1.1
<b>10.222.2.0/24 via 10.222.2.1 dev antrea-gw0 onlink </b>
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
</code></pre>

The third route entry, **"10.222.2.0/24 via 10.222.2.1 dev antrea-gw0 onlink"**, basically means that the next hop for the subnet 10.222.2.0/24 is 10.222.2.1 and it can be reached through antrea-gw0 interface. ("dev antrea-gw0") But 10.222.2.1 is not part of any directly connected subnets on worker1 node ? So worker1 node would normally need to perform recursive routing to check how it can reach 10.222.2.1. The interesting trick here is the "onlink" parameter used in the same route entry. This parameter makes **the Linux Kernel IP stack to think as if 10.222.2.1 is a local next hop** which is part of the network that antrea-gw0 interface is also directly connected to. **Hence when worker1 node sends any traffic destined to subnet 10.222.2.0/24**, it would first send an ARP request for 10.222.2.1 from its antrea-gw0 interface.

Which traffic pattern would require worker1 node to send an ARP request ? The simple one that comes to mind is the flow which is explained in Section 9, frontend pod on worker1 node to backend2 pod on worker2 node flow. Shown below. **Basically, pod to service traffic which gets load balanced to a pod on another node.** 

DIAGRAM DIAGRAM DIAGRAM

When the flow from 10.222.1.48 (frontend) to 10.222.2.34 (backend2) needs to be sent by worker1 node, that is exactly when worker1 node needs to figure out where to send the tarffic.

The fifth route entry, "10.222.0.0/24 via 10.222.0.1 dev antrea-gw0 onlink" , is for the pod subnet on the other Kubernetes node, which is the master node. 

Whenever a new node is added to the Kubernetes cluster, Antrea Node Controller would detect that by watching the Kubernetes API and then add a route entry, similar to the ones above, on worker1 node' s route table. (Explained [here](https://github.com/vmware-tanzu/antrea/blob/master/docs/architecture.md#antrea-agent))

Until now, the reason why a node would send an ARP request for another node' s gw0 interace IP is explained above. What about how OVS handles these ARP requests ? For that the second flow entry in Table 20 is explained in more detail below.

<pre><code>
cookie=0x1020000000000, table=20, priority=200,arp,arp_tpa=10.222.2.1,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:aa:bb:cc:dd:ee:ff,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0xaabbccddeeff->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xade0201->NXM_OF_ARP_SPA[],IN_PORT
</code></pre>

This flow entry checks if the current flow that comes ingress to OVS is an arp flow (arp), and if the ARP request is sent for IP address 10.222.2.1 (arp_tpa=10.222.2.1, tpa stands for target IP address), and if it is an arp request (arp_op=1 is for arp request). Once all these match with the flow then the following actions are taken on the flow : 

* <b>move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[] :</b> This action moves the MAC address, seen in the source address field of the ethernet header, to the destination address field in the ethernet header of the response
* <b>mod_dl_src:aa:bb:cc:dd:ee:ff :</b> This action puts the MAC address "aa:bb:cc:dd:ee:ff" into the source address field in the ethernet header of the response
* <b>load:0x2->NXM_OF_ARP_OP[] :</b> This action basically sets "0x2" which is "2" in decimal for "arp_op" and "arp_op=2" is used for ARP response
* <b>move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[] :</b> This action moves the MAC address, seen in the sender MAC address field in the ARP header of the ARP request, to the target MAC address field in the ARP header of the response 
* <b>load:0xaabbccddeeff->NXM_NX_ARP_SHA[] :</b> This action puts the MAC address "aa:bb:cc:dd:ee:ff" into the sender MAC address field in the ARP header of the response
* <b>move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[] :</b> This action moves the IP address, seen in the sender IP address field of the ARP header, to the target IP address field in the ARP header of the response 
* <b>load:0xade0201->NXM_OF_ARP_SPA[] :</b> This action puts the IP address "10.222.2.1" (conversion from hex 0xade0201) into the the sender IP address field of the ARP header of the response 
* <b>IN_PORT :</b> This action means "forward the flow back to the port where it came" (which is the antrea-gw0 interface port on OVS on the worker1 node in this case)

Note : More detailed info about these fields can be found in the "ovs-fields" section on [this page](https://docs.openvswitch.org/en/latest/ref/?highlight=fields#man-pages).
